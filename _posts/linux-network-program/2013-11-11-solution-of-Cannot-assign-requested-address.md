---
title: can-not-assign-requested-address出现的原因及解决方案
layout: post
categories: [网络编程]
tags: [network]
---

今天用php连接最近新开发的一个服务做测试，发现命令行打印出：Cannot assign requested address

网上找了下原因，大致上是由于客户端频繁的连服务器，由于每次连接都在很短的时间内结束，导致很多的TIME_WAIT，以至于用光了可用的端 口号，所以新的连接没办法绑定端口，即“Cannot assign requested address”。是客户端的问题不是服务器端的问题。通过netstat，的确看到很多TIME_WAIT状态的连接。

client端频繁建立连接，而端口释放较慢，导致建立新连接时无可用端口。

网上的解决方法：

执行命令修改如下2个内核参数 （需要root权限）     
    开启对于TCP时间戳的支持,若该项设置为0，则下面一项设置不起作用  
    sysctl -w net.ipv4.tcp_timestamps=1  
	表示开启TCP连接中TIME-WAIT sockets的快速回收  
	sysctl -w net.ipv4.tcp_tw_recycle=1  
	
-EOF-
