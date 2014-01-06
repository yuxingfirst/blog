---
title: 协程简介及协程库libcoro性能测试
layout: post
categories: [coroutine]
tags: [coroutine]
description:   .
---  

###协程(coroutine)

相信你肯定知道进程(process)、线程(thread),进程是一个实体，一个"执行中的程序"；每一个进程有自己的地址空间，一般包括:文本区域，数据区和堆栈。而线程又叫做轻量级进程，它是程序执行流的最小单元。  

那么什么是协程呢?首先请移步看看协程的定义[协程](http://zh.wikipedia.org/wiki/%E5%8D%8F%E7%A8%8B)。正像wikipedia中描述的，子例程的起始处是惟一的入口点，一旦退出即完成了子程序的执行，子程序的一个实例只会返回一次。协程可以通过yield来调用其它协程。通过yield方式转移执行权的协程之间不是调用者与被调用者的关系，而是彼此对称、平等的。协程的起始处是第一个入口点，在协程里，返回点之后是接下来的入口点，也就是说，当一个协程重新获得cpu的时候，它会接着上次yield之后的代码继续执行。子例程的生命期遵循后进先出（最后一个被调用的子例程最先返回）；相反，协程的生命期完全由他们的使用的需要决定。  

看到这里你可能还是会觉得很抽象，下边我们通过一个例子再来看看:

	coroutine produce
		loop
		   while q is not full
		       create some new items
		       add the items to q
		   yield to consume  

	coroutine consume
		loop
		   while q is not empty
		       remove some items from q
		       use the items
		   yield to produce  

如上程序，coroutine produce和coroutine consume是两个独立的协程,而yield能否实现从一个协程切换到另一个协程，两个协程交替获取cpu，也即是说，一会执行produce，一会执行consume。

###libcoro

下边主要是对协程库libcoro做一点测试。libcoro实现协程的切换主要多种方式，我这里主要测试UNCONTEXT和ASM的切换性能。更多的实现方式参考: [libcoro源码](https://github.com/yuxingfirst/libcoro)

示例程序在这里 [coro test](https://github.com/yuxingfirst/libcoro/blob/master/t.c),感兴趣的读者也可以自己编译试试。我主要测试：2个协程，10个协程，100个协程，1000个协程之间的切换效率。整体测试下来，协程越少，ASM的切换效率高UCONTEXT越多。当100个协程时，1秒内，ASM方式可以切换5000W次，而UCONTEXT只有近400W次，差不多是10倍以上。  

###为什么需要协程

看到这里，你可能会有一个疑问，协程可以用来干吗呢?为什么需要协程呢?或者说，协程主要解决了什么问题呢?  

我们知道，倘若我们基于线程实现一个网络服务器(当然是非阻塞模型)，必定会需要进行异步回调，当收到一个请求时，需要回调某一个处理函数。如果逻辑复杂一点，则异步回调的部分写起来应该也会比较的复杂。不过如果是基于协程的话，每当收到一个请求，就可以创建一个协程进行处理(因为协程比较轻量，在一个进程中，我们可以创建数以万计的协程来处理请求，而线程的创建是有限制的)，然后，我们只要有一个调度器，用以实现协程之间的切换，则我们就可以用**同步的方式写出异步的代码**。

-EOF-



