---
title: 协程的多线程切换
layout: post
categories: [linux-c]
tags: [coroutine]
description: coro multiple thread switch.
---

我们知道，在一个基于协程的应用程序中，可能会产生数以千记的协程，所有这些协程，会有一个的调度器来统一调度。  
另外我们知道，高性能的程序首要注意的就是避免程序阻塞。那么，在以协程为最小运行单位的程序中，同样也需要确保这一点，即每一个协程都不能发生阻塞。因为只要某一个协程发生了阻塞，那么整个调度器就阻塞住了，后续等待的协程得不到运行，整个程序此时也将死翘翘了。那么，如果需要保证任意一个协程都不阻塞，该怎么做呢？  
  
通常，协程主调度器管理着本线程中所有的协程，并依次调度这些协程运行，在一个协程执行完之后，需要将执行权限交给调度器,即进行 <code>yield</code> 操作，以便调度器能够调度后续等待运行的协程。如果在某个协程内，含有阻塞操作，如打开数据库连接：  

	exe_non_block1();

	open_db_conn(...);
	
	exe_non_block2();  

如上边代码所示，先执行了<code>exe_non_block1</code>, 然后打开一个数据库连接，之后再执行<code>exe_non_block2</code>， 如果对这段代码不做任何处理，倘若打开数据库连接需要耗时很多，那么在这期间，整个程序就阻塞住了，这种情况是绝对不能容许的。怎么解决呢？   

我们看，先执行了<code>exe_non_block2</code>, 然后执行 <code>open_db_conn</code>, 如果能够把<code>open_db_conn</code>这个函数调用放到其他的线程中去执行，同时本协程 <code>yield</code>，交出执行权限；之后，当<code>open_db_conn</code>在其他线程执行完毕时，再切换到这个协程，并且能够接着 <code>exe_non_block2</code> 继续执行，那么我们就可以解决上边提出的问题, 保证整个主调度器不会阻塞，即便某一个协程中需要进行阻塞操作，我们也可以把这段会阻塞的代码放到其他线程中取执行，等执行完成后，再切换回来。换句话说，就是需要协程有这样的特性:  

	能够按照代码顺序依次在多个线程中执行  

那么，协程有这样的特性吗? 很幸运，协程是有的，因为在每次交出执行权限(即<code>yield</code>)时，都会保留栈信息。请看下边的实验代码([multi_thread_switch.c](https://github.com/yuxingfirst/libcoro/blob/master/test/multi_thread_switch.c "multi_thread_switch") ):

	#include "../libcoro/coro.h"
	#include <pthread.h>
	#include <stdio.h>
	#include <stdlib.h>

	coro_context main_ctx;
	coro_context parallel_ctx;

	coro_context ctxa;

	pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

	coro_context *current_main_ctx;


	void parallel_coro_func(void *arg);

	void* thread_main_func(void* arg)
	{
		pthread_t pid = pthread_self();
		printf("subthread:%lu\n", pid);
		void *stk = malloc(8192);
		coro_create(&parallel_ctx, &parallel_coro_func, NULL, stk, 8192);
		coro_context ctx;
		coro_transfer(&ctx, &parallel_ctx);
	}

	void parallel_coro_func(void *arg)
	{
		while(1) {
		    pthread_mutex_lock(&mutex);
		    current_main_ctx = &parallel_ctx;
		    coro_transfer(&parallel_ctx, &ctxa);
		    pthread_mutex_unlock(&mutex);
		    usleep(1000*10);
		}
	}

	void coro_funca(void *arg) 
	{
		pthread_t ptid;
		while(1) {

		    int c = 0;
		    printf("----------start loop---------\n");

		    ++c;
		    ptid = pthread_self();  
		    printf("coro_funca, thread: %lu, count:%d\n", ptid, c);
		    coro_transfer(&ctxa, current_main_ctx);

		    ++c;
		    ptid = pthread_self();  
		    printf("coro_funca, thread: %lu, count:%d\n", ptid, c);
		    coro_transfer(&ctxa, current_main_ctx);

		    ++c;
		    ptid = pthread_self();  
		    printf("coro_funca, thread: %lu, count:%d\n", ptid, c);
		    coro_transfer(&ctxa, current_main_ctx);

		    ++c;
		    ptid = pthread_self();  
		    printf("coro_funca, thread: %lu, count:%d\n", ptid, c);
		    coro_transfer(&ctxa, current_main_ctx);

		    printf("----------end loop---------\n\n");
		}
	}

	int main(void)
	{
		void *stka = malloc(8192);
		coro_create(&ctxa, &coro_funca, NULL, stka, 8192);

		pthread_t ptid1;     
		if(pthread_create(&ptid1, NULL, &thread_main_func, NULL) != 0) {
		    exit(0); 
		}
		pthread_t mpid = pthread_self();
		printf("main thread:%lu\n", mpid);
		while(1) {
		    pthread_mutex_lock(&mutex);
		    current_main_ctx = &main_ctx;
		    coro_transfer(&main_ctx, &ctxa);
		    pthread_mutex_unlock(&mutex);
		    usleep(1000*10);
		}
		return 0;
	}

-EOF-

