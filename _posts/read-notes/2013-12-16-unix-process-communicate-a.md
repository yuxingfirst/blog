---
title: Unix进程通信-pipes
layout: post
categories: [读书笔记]
tags: [APUE读书笔记-Chapter15, 进程通信]
description: Unix进程通信之管道(pipes)
---

pipes是在所有unix系统中均被支持的最老的进程通信方式。同时，它也有两个限制:  

1. 半双工通信(half duplex)。  
2. pipes只能在拥有共同祖先的两个进程之间使用。(例如:  进程A创建了一个管道pa, 然后进程Afork了进程B，那么pipes pa能够在进程A、B之间使用)  

我们可以调用pipe来创建一个管道:  

	#include <unistd.h>
	int pipe(int filedes[2]);  
				//returns 0 ok; -1 on error  

我们注意到pipe的参数: int filedes， pipe内部借助这个参数给调用者返回一对文件描述符，filedes[0]用于读(read)操作；filedes[1]用于写(write)操作。也就是说，filedes[1]的输出是filedes[0]的输入。  

> pipes在4.3BSD, 4.4BSD和Mac OS X 10.3上是基于unix domain socket实现的，但是，虽然unix domain socket是全双工通信的，但是这些系统在实现pipes的时候仍旧只用半双工方式操作。


-EOF-


