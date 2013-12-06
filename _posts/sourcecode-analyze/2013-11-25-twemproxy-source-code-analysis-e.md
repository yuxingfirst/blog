---
title: twemproxy源码分析part5-客户端请求处理
layout: post
category: [源码分析]
tags: [redis, twemproxy]
---

这篇， 我将重点介绍twemproxy中消息的处理过程，即对：
	
	接收客户端连接请求->读取客户端请求数据->发送至服务器端->接受服务器返回->响应客户端请求  

这个流程在twemproxy中具体是如何实现的做出详细分析。  

上一篇([这里](http://yuxingfirst.github.io/posts/twemproxy-source-code-analysis-d.html))，我们介绍了当epoll检查到有I/O事件发生的时候，最后都会调用到下边这两个函数中：
	
	//nc_core.c
	status = core_recv(ctx, conn);
	status = core_send(ctx, conn);

下边我们看下这两个函数的实现：  

	static rstatus_t
	core_recv(struct context *ctx, struct conn *conn)
	{
	    rstatus_t status;
	
	    status = conn->recv(ctx, conn);
	    if (status != NC_OK) {
	        log_debug(LOG_INFO, "recv on %c %d failed: %s",
	                  conn->client ? 'c' : (conn->proxy ? 'p' : 's'), conn->sd,
	                  strerror(errno));
	    }
	
	    return status;
	}
	
	static rstatus_t
	core_send(struct context *ctx, struct conn *conn)
	{
	    rstatus_t status;
	
	    status = conn->send(ctx, conn);
	    if (status != NC_OK) {
	        log_debug(LOG_INFO, "send on %c %d failed: %s",
	                  conn->client ? 'c' : (conn->proxy ? 'p' : 's'), conn->sd,
	                  strerror(errno));
	    }
	
	    return status;
	}  

可以看到，最后都会针对conn调用其recv/send函数。由于twemproxy中的conn可以分为三种类型：  

1. client
2. proxy
3. server    

上一篇([这里](http://yuxingfirst.github.io/posts/twemproxy-source-code-analysis-d.html))中我同样也为大家介绍了针对这三种不同的conn, 会给他里边各个函数指针设置不同的处理函数。

同时，我们应该明确一个概念，twemproxy中的三种conn各自分工不同：  

1. 对于一个proxy conn来说，其最主要的职责应该就是接受(accept)客户端连接请求，而对于具体的数据接收过程，则不应在它所管辖的范畴。  
2. 而对于client conn来说，它主要负责proxy跟client之间的数据交互。  
3. 而server conn主要就是负责proxy跟server之间的数据交互。 

有了这点认识，我们下边就来分析这三种不同的conn各自的实现方式。  

##proxy
我们知道，在c/c++网络编程中，客户端可能并不会在一连接到服务端的时候就立马发送数据。所以，在编写服务端代码的时候，我们不能在接受到一个客户端连接时，就立马对这个套接字进行读数据操作，此时有可能因为并没有数据到达而导致阻塞(如果对套接字设置了nonblock，则不会阻塞而是会设置相应的错误码)， 而是应该借助系统提供的I/O multiplexing函数(epoll, select, poll)来探测，在客户端数据真正到达时，我们才去对套接字进行读数据操作。基于这个原则，我们看看proxy接受客户端连接的实现代码：  

	static rstatus_t
	proxy_accept(struct context *ctx, struct conn *p)
	{
	    rstatus_t status;
	    struct conn *c;
	    int sd;
	
	    ASSERT(p->proxy && !p->client);
	    ASSERT(p->sd > 0);
	    ASSERT(p->recv_active && p->recv_ready);
	
	    for (;;) {
	        sd = accept(p->sd, NULL, NULL);
	        if (sd < 0) {
	            if (errno == EINTR) {	//accept由于系统中断而返回，需要立即再次重试
	                log_debug(LOG_VERB, "accept on p %d not ready - eintr", p->sd);
	                continue;
	            }
	
	            if (errno == EAGAIN || errno == EWOULDBLOCK) {	//连接请求还未达到，需要在下次轮询中重试
	                log_debug(LOG_VERB, "accept on p %d not ready - eagain", p->sd);
	                p->recv_ready = 0;
	                return NC_OK;
	            }
	
	            /*
	             * FIXME: On EMFILE or ENFILE mask out IN event on the proxy; mask
	             * it back in when some existing connection gets closed
	             */
	
	            log_error("accept on p %d failed: %s", p->sd, strerror(errno));
	            return NC_ERROR;
	        }
	
	        break;
	    }
		//接受到客户端连接， 需要新拿到一个conn对象，用于跟client之间的数据交互
		//注意这里拿到的conn都属于client类型
	    c = conn_get(p->owner, true, p->redis);		
	    if (c == NULL) {
	        log_error("get conn for c %d from p %d failed: %s", sd, p->sd,
	                  strerror(errno));
	        status = close(sd);
	        if (status < 0) {
	            log_error("close c %d failed, ignored: %s", sd, strerror(errno));
	        }
	        return NC_ENOMEM;
	    }
	    c->sd = sd;	//设置conn的套接字描述,即后续的conn都会基于这个套接字读写数据
	
	    stats_pool_incr(ctx, c->owner, client_connections);
	
	    status = nc_set_nonblocking(c->sd);	//设置套接字非阻塞
	    if (status < 0) {
	        log_error("set nonblock on c %d from p %d failed: %s", c->sd, p->sd,
	                  strerror(errno));
	        c->close(ctx, c);
	        return status;
	    }
	
	    if (p->family == AF_INET || p->family == AF_INET6) {
	        status = nc_set_tcpnodelay(c->sd);
	        if (status < 0) {
	            log_warn("set tcpnodelay on c %d from p %d failed, ignored: %s",
	                     c->sd, p->sd, strerror(errno));
	        }
	    }
		//将这个conn加入到epoll的管理，这样当客户端的请求数据到达的时候，epoll就会通知我们做读取数据操作。
	    status = event_add_conn(ctx->ep, c);	
	    if (status < 0) {
	        log_error("event add conn of c %d from p %d failed: %s", c->sd, p->sd,
	                  strerror(errno));
	        c->close(ctx, c);
	        return status;
	    }
	
	    log_debug(LOG_NOTICE, "accepted c %d on p %d from '%s'", c->sd, p->sd,
	              nc_unresolve_peer_desc(c->sd));
	
	    return NC_OK;
	}

##client
对于client conn来说，既需要读取来自client端的数据；也需要给client端发送数据。并且，在上一篇文章中我们分析过，一个client类型的conn， 其recv/send分别被设置成如下函数：  

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
		...  

conn-recv为接收客户端数据的函数，conn->send为给客户端发送数据的函数。  

