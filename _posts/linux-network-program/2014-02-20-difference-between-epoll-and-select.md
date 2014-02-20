---
title: 两种i/o多路复用技术epoll和select的区别
layout: post
categories: [网络编程]
tags: [network]
description: .
---

linux网络编程中，有关IO多路复用技术，用的比较多的要数epoll和select这两个函数了，select传承自unix，而epoll在linux 2.6以后才被支持，那这两者有什么区别呢?以及这两者的原理分别是什么样的呢?下边来探讨下。  

首先来看select：

	int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);  

select允许程序监听多个文件描述符，等待直到有一个或多个文件描述符进入就绪状态(ready， 就是说可以基于这个就绪的文件描述符做响应的I/O操作而不会引起阻塞)。  



-EOF-
