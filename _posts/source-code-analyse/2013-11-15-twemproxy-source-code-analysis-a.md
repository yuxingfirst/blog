---
title: twemproxy源码分析part1-启动流程
layout: post
categories: [技术研究]
tags: [redis, twemproxy]
description: .
---

###twemproxy简介
twemproxy,也叫nutcraker。是一个twtter开源的一个redis和memcache代理服务器。 redis作为一个高效的缓存服务器，非常具有应用价值。但是当使用比较多的时候，就希望可以通过某种方式 统一进行管理。避免每个应用每个客户端管理连接的松散性。  

github地址: [twemproxy](https://github.com/twitter/twemproxy)  

后边，我将通过通过一系列的文章，来对twemproxy的源代码做一个深入的分析。  

本篇中，我们首先来看twemproxy的启动过程。  

###整体启动流程
我们从main函数入手。  

首先是根据命令行参数，初始化服务器选项:  

	nc_set_default_options(&nci);

    status = nc_get_options(argc, argv, &nci);   

通过这两个函数，主要是对struct instance的一部分属性以及一些全局的变量做初始化。然后，通过下边的这部分代码，启动服务：  

	status = nc_pre_run(&nci);
    if (status != NC_OK) {
        nc_post_run(&nci);
        exit(1);
    }

    nc_run(&nci);  

首先是nc\_pre\_run：  

	static rstatus_t nc_pre_run(struct instance *nci)；  

这个函数主要是初始化日志，信号设置，创建pid文件以及设置是否为deamon进程。然后，调用nc\_run启动服务：
	
	static void nc_run(struct instance *nci)
	{
		rstatus_t status;
	    struct context *ctx;
	
	    ctx = core_start(nci);
	    if (ctx == NULL) {
	        return;
	    }
	
	    /* run rabbit run */
	    for (;;) {
	        status = core_loop(ctx);
	        if (status != NC_OK) {
	            break;
	        }
	    }
	
	    core_stop(ctx);
	}  

core\_start函数中主要是读取配置文件，并对整个服务做初始化，中间详细的实现过程我们后边在详述。然后是死循环进行事件轮询。下边我们看下core\_loop的代码：  

	rstatus_t core_loop(struct context *ctx)
	{
	    int i, nsd;
	
	    nsd = event_wait(ctx->ep, ctx->event, ctx->nevent, ctx->timeout);
	    if (nsd < 0) {
	        return nsd;
	    }
	
	    for (i = 0; i < nsd; i++) {
	        struct epoll_event *ev = &ctx->event[i];
	
	        core_core(ctx, ev->data.ptr, ev->events);
	    }
	
	    core_timeout(ctx);
	
	    stats_swap(ctx->stats);
	
	    return NC_OK;
	}

epoll部分的代码相信大家都不陌生， core\_timeout是检查超时请求(twemproxy对后端server的请求)，这部分这里也先不做分析。  
我们主要看下core\_core这个函数:  

	static void core_core(struct context *ctx, struct conn *conn, uint32_t events)
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
	    if (events & (EPOLLIN | EPOLLHUP)) {
	        status = core_recv(ctx, conn);
	        if (status != NC_OK || conn->done || conn->err) {
	            core_close(ctx, conn);
	            return;
	        }
	    }
	
	    if (events & EPOLLOUT) {
	        status = core_send(ctx, conn);
	        if (status != NC_OK || conn->done || conn->err) {
	            core_close(ctx, conn);
	            return;
	        }
	    }
	}  

我们知道，一般网络I/O相关的模块中，主要是就是读/写以及错误事件进行处理，所以这里也不例外，根据events的类型，调用core_recv/core_send/core_error进行事件处理。这里我们重点看下core\_recv/core\_send这个两个函数：  

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

对于一个代理来说，它既要处理来自客户端的请求，也需要向后端server发送请求，接受响应。所以，这两个函数的参数conn可能是客户端的连接，也可能是其自身向后端server发起的连接，根据这两者的不同，处理方式也不一样。所以，上边的：
	
	status = conn->recv(ctx, conn);
	status = conn->send(ctx, conn);  

这两个函数调用，recv/send都是函数指针，根据连接的不同调用不同的处理函数。这两个函数具体指向那个函数，我们在介绍配置解析这部分的时候在依次说明。此时，我们可以先定位到connection.h这个文件的下边这部分代码：
	
	typedef rstatus_t (*conn_recv_t)(struct context *, struct conn*);
	typedef rstatus_t (*conn_send_t)(struct context *, struct conn*);

	struct conn {
		...
		conn_recv_t        recv;          /* recv (read) handler */
		...
		conn_send_t        send;          /* send (write) handler */
	}  

可以看出，上边的recv/send分别为函数指针。  

至此，我们就分析完了twemproxy的整个启动流程及请求的简要处理逻辑。  

-EOF-
