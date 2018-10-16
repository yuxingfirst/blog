---
title: twemproxy源码分析part4-conn及client请求逻辑
layout: post
categories: [技术研究]
tags: [redis, twemproxy]
description: .
---

##connection, proxy，client,server
在分析message的处理流程之前，我们先来明确一下twemproxy中比较关键的几个概念。  
我们知道，作为一个代理中间件来说，其主要职责是接受客户端的请求，然后将消息发往服务器端，之后在接受服务器回复的消息并对客户端做出响应。在这个过程中，我们可以看到有三个角色  

1. proxy 中间件本身
2. client 客户端
3. server 后端服务器  

其实，相对于client来说，proxy是其服务器，因为client直接跟proxy交互，有proxy去代理其请求。   
对应到代码层面来说，首先比较重要的一点就是tcp连接的管理了，因为所有请求、响应的交互都是基于tcp连接的。  

我们来看下twemproxy中connection的定义：  

	struct conn {
    TAILQ_ENTRY(conn)  conn_tqe;      /* link in server_pool / server / free q */
    void               *owner;        /* connection owner - server_pool / server */

    int                sd;            /* socket descriptor 套接字描述符*/
    int                family;        /* socket address family 协议族*/
    socklen_t          addrlen;       /* socket length */
    struct sockaddr    *addr;         /* socket address (ref in server or server_pool) */

    struct msg_tqh     imsg_q;        /* incoming request Q */
    struct msg_tqh     omsg_q;        /* outstanding request Q */
    struct msg         *rmsg;         /* current message being rcvd */
    struct msg         *smsg;         /* current message being sent */

    conn_recv_t        recv;          /* recv (read) handler */
    conn_recv_next_t   recv_next;     /* recv next message handler */
    conn_recv_done_t   recv_done;     /* read done handler */
    conn_send_t        send;          /* send (write) handler */
    conn_send_next_t   send_next;     /* write next message handler */
    conn_send_done_t   send_done;     /* write done handler */
    conn_close_t       close;         /* close handler */
    conn_active_t      active;        /* active? handler */

    conn_ref_t         ref;           /* connection reference handler */
    conn_unref_t       unref;         /* connection unreference handler */

    conn_msgq_t        enqueue_inq;   /* connection inq msg enqueue handler */
    conn_msgq_t        dequeue_inq;   /* connection inq msg dequeue handler */
    conn_msgq_t        enqueue_outq;  /* connection outq msg enqueue handler */
    conn_msgq_t        dequeue_outq;  /* connection outq msg dequeue handler */

    size_t             recv_bytes;    /* received (read) bytes */
    size_t             send_bytes;    /* sent (written) bytes */

    uint32_t           events;        /* connection io events */
    err_t              err;           /* connection errno */
    unsigned           recv_active:1; /* recv active? */
    unsigned           recv_ready:1;  /* recv ready? */
    unsigned           send_active:1; /* send active? */
    unsigned           send_ready:1;  /* send ready? */

    unsigned           client:1;      /* client? or server? 是否为client->proxy的连接*/
    unsigned           proxy:1;       /* proxy? proxy监听的conn */
    unsigned           connecting:1;  /* connecting? */
    unsigned           connected:1;   /* connected? */
    unsigned           eof:1;         /* eof? aka passive close? */
    unsigned           done:1;        /* done? aka close? */
    unsigned           redis:1;       /* redis? */
	};  

首先，对于一个connection来说，其代表一个"通信信道"，那么在这种代理中间件中，一个信道就会有不同的两端，分别是：  

> client -> proxy  
> proxy  -> server  

同时，proxy本身也是一个server(接受client请求)，所以其监听的套接字也会映射为一个connection。 所以上述connection定义中，
  
	unsigned proxy:1;  

即标识这个connection是不是一个监听套接字连接，而：  

	unsigned           client:1;  

用来标识连接是client端的连接还是到server的连接。然后，  

	unsigned           connecting:1;  /* connecting? */
    unsigned           connected:1;  

用来标识这个连接是正在进行中还是已经完成连接。因为当我们在向一个对端server发起tcp连接时，如果对该套接字设置了nonblock属性，
那么有可能在connect函数返回时，连接可能还正在进行中。然后在后边，我们会利用这个connection来传输数据，所以需要一个标记标识该connection
是否已经可用。我们可以来看下twemproxy中的实现代码：  

	//nc_server.c
	rstatus_t
	server_connect(struct context *ctx, struct server *server, struct conn *conn)	{
		if (conn->sd > 0) {		//取保在connection中的套接字有效后，不会重复连接
        	/* already connected on server connection */
        	return NC_OK;
    	}	
		//...
		conn->sd = socket(conn->family, SOCK_STREAM, 0);	//创建socket套接字描述符
		//...
		status = nc_set_nonblocking(conn->sd);
		//...
		status = event_add_conn(ctx->ep, conn);	//将conn加入epoll并设置读写事件，因为当连接不能立即完成时，我们后边要借助epoll来探测
		//...
		//因为conn->sd设置了nonblock属性，则connect立即返回，此时如果errno为EINPROGRESS，所以连接过程(三次握手)
		//还在继续进行中，这是种正确的状态，所以将connecting设置为1(true)
		status = connect(conn->sd, conn->addr, conn->addrlen);
		if (status != NC_OK) {
		    if (errno == EINPROGRESS) {
		        conn->connecting = 1;
		        log_debug(LOG_DEBUG, "connecting on s %d to server '%.*s'",
		                  conn->sd, server->pname.len, server->pname.data);
		        return NC_OK;
		    }

		    log_error("connect on s %d to server '%.*s' failed: %s", conn->sd,
		              server->pname.len, server->pname.data, strerror(errno));

		    goto error;
		}

		//...
		conn->connected = 1;	//如果连接立即完成，则将connected设置为1(true)
		/...
	}  

因为twemproxy既能作为memcached的代理，也能作为redis的代理，所以：
	
	unsigned           redis:1;       /* redis? */  

redis这个字段用来标识是redis的代理，还是memcached的代理，因为两者所使用的协议不一样，所以需要区分使用不同的消息处理函数。  

下边，我们在来分析connection中的处理函数:  

	conn_recv_t        recv;          /* recv (read) handler */
    conn_recv_next_t   recv_next;     /* recv next message handler */
    conn_recv_done_t   recv_done;     /* read done handler */
    conn_send_t        send;          /* send (write) handler */
    conn_send_next_t   send_next;     /* write next message handler */
    conn_send_done_t   send_done;     /* write done handler */
    conn_close_t       close;         /* close handler */
    conn_active_t      active;        /* active? handler */
	conn_ref_t         ref;           /* connection reference handler */
    conn_unref_t       unref;         /* connection unreference handler */

    conn_msgq_t        enqueue_inq;   /* connection inq msg enqueue handler */
    conn_msgq_t        dequeue_inq;   /* connection inq msg dequeue handler */
    conn_msgq_t        enqueue_outq;  /* connection outq msg enqueue handler */
    conn_msgq_t        dequeue_outq;  /* connection outq msg dequeue handler */

这些都是函数指针，因为对于一个connection来说，可能是client发起到proxy的连接；也可能是proxy监听的套接字连接;还可能是proxy到
server的连接。所以对于这三种情况，当有数据需要进行收发时，需要调用不同的处理函数，下边我们来看具体的实现过程。  

在程序初始化过程中，在将配置文件的内容分析完毕并生成对应的context， server_pool, server后，会根据配置，预先对后端的server(memcached、redis)创建一个连接：
	
	//nc_core.c
	static struct context *
	core_ctx_create(struct instance *nci) {
		//...
		/* preconnect? servers in server pool */
		status = server_pool_preconnect(ctx);
		if (status != NC_OK) {
		    server_pool_disconnect(ctx);
		    event_deinit(ctx);
		    stats_destroy(ctx->stats);
		    server_pool_deinit(&ctx->pool);
		    conf_destroy(ctx->cf);
		    nc_free(ctx);
		    return NULL;
		}
		
		/* initialize proxy per server pool */
		status = proxy_init(ctx);
		if (status != NC_OK) {
		    server_pool_disconnect(ctx);
		    event_deinit(ctx);
		    stats_destroy(ctx->stats);
		    server_pool_deinit(&ctx->pool);
		    conf_destroy(ctx->cf);
		    nc_free(ctx);
		    return NULL;
		}
	}

然后在server_pool_preconnect这个函数的处理过程中，最后会通过conn_get获取一个connection对象：  

	struct conn *
	conn_get(void *owner, bool client, bool redis)
	{
		struct conn *conn;

		conn = _conn_get();
		if (conn == NULL) {
		    return NULL;
		}

		/* connection either handles redis or memcache messages */
		conn->redis = redis ? 1 : 0;

		conn->client = client ? 1 : 0;

		if (conn->client) {
		    /*
		     * client receives a request, possibly parsing it, and sends a
		     * response downstream.
		     */
		    conn->recv = msg_recv;
		    conn->recv_next = req_recv_next;
		    conn->recv_done = req_recv_done;

		    conn->send = msg_send;
		    conn->send_next = rsp_send_next;
		    conn->send_done = rsp_send_done;

		    conn->close = client_close;
		    conn->active = client_active;

		    conn->ref = client_ref;
		    conn->unref = client_unref;

		    conn->enqueue_inq = NULL;
		    conn->dequeue_inq = NULL;
		    conn->enqueue_outq = req_client_enqueue_omsgq;
		    conn->dequeue_outq = req_client_dequeue_omsgq;
		} else {
		    /*
		     * server receives a response, possibly parsing it, and sends a
		     * request upstream.
		     */
		    conn->recv = msg_recv;
		    conn->recv_next = rsp_recv_next;
		    conn->recv_done = rsp_recv_done;

		    conn->send = msg_send;
		    conn->send_next = req_send_next;
		    conn->send_done = req_send_done;

		    conn->close = server_close;
		    conn->active = server_active;

		    conn->ref = server_ref;
		    conn->unref = server_unref;

		    conn->enqueue_inq = req_server_enqueue_imsgq;
		    conn->dequeue_inq = req_server_dequeue_imsgq;
		    conn->enqueue_outq = req_server_enqueue_omsgq;
		    conn->dequeue_outq = req_server_dequeue_omsgq;
		}

		conn->ref(conn, owner);

		log_debug(LOG_VVERB, "get conn %p client %d", conn, conn->client);

		return conn;
	}

从上边的函数可以看出，会根据是client还是server，对conn中的各个函数指针设置不同的处理函数。然后，在后边进行消息处理的时候，分别调用不同的函数。  

这里，我们也顺便看下proxy监听的套接字连接的处理函数：  
	
	struct conn *
	conn_get_proxy(void *owner)
	{
		struct server_pool *pool = owner;
		struct conn *conn;

		conn = _conn_get();
		if (conn == NULL) {
		    return NULL;
		}

		conn->redis = pool->redis;

		conn->proxy = 1;

		conn->recv = proxy_recv;
		conn->recv_next = NULL;
		conn->recv_done = NULL;

		conn->send = NULL;
		conn->send_next = NULL;
		conn->send_done = NULL;

		conn->close = proxy_close;
		conn->active = NULL;

		conn->ref = proxy_ref;
		conn->unref = proxy_unref;

		conn->enqueue_inq = NULL;
		conn->dequeue_inq = NULL;
		conn->enqueue_outq = NULL;
		conn->dequeue_outq = NULL;

		conn->ref(conn, owner);

		log_debug(LOG_VVERB, "get conn %p proxy %d", conn, conn->proxy);

		return conn;
	}

因为对于一个监听套接字连接来说，只需要接受client端的连接请求(accept调用)， 所以，只需要设置  

	conn->recv = proxy_recv;   

recv函数即可。  

最后，在初始化完成后， nc_core.c/core_ctx_create 创建context完毕， 会进入事件循环处理部分：  

	rstatus_t
	core_loop(struct context *ctx)
	{
		int i, nsd;

		nsd = event_wait(ctx->ep, ctx->event, ctx->nevent, ctx->timeout);
		if (nsd < 0) {
		    return nsd;
		}

		for (i = 0; i < nsd; i++) {
		    struct epoll_event *ev = &ctx->event[i];

		    core_core(ctx, ev->data.ptr, ev->events);	//epoll探测到有读写事件发生，进行处理。
		}

		core_timeout(ctx);

		stats_swap(ctx->stats);

		return NC_OK;
	}

	static void
	core_core(struct context *ctx, struct conn *conn, uint32_t events)
	{
		rstatus_t status;

		log_debug(LOG_VVERB, "event %04"PRIX32" on %c %d", events,
		          conn->client ? 'c' : (conn->proxy ? 'p' : 's'), conn->sd);

		conn->events = events;

		/* error takes precedence over read | write */
		if (events & EPOLLERR) {
		    core_error(ctx, conn);
		    return;
		}

		/* read takes precedence over write */
		if (events & (EPOLLIN | EPOLLHUP)) {	//读事件
		    status = core_recv(ctx, conn);
		    if (status != NC_OK || conn->done || conn->err) {
		        core_close(ctx, conn);
		        return;
		    }
		}

		if (events & EPOLLOUT) {				//写事件
		    status = core_send(ctx, conn);
		    if (status != NC_OK || conn->done || conn->err) {
		        core_close(ctx, conn);
		        return;
		    }
		}
	}

最后，在：

	status = core_recv(ctx, conn);
	status = core_send(ctx, conn);  

这两个函数中，进行具体的i/o操作处理。  

-EOF-








