---
title: tcp选项TCP_DEFER_ACCEPT分析
layout: post
category: [linux-network-program]
tags: [network]
---

最近在追查一个代理服务器请求后端业务逻辑服务时，出现地址不可达的bug，反映到tcp这边的提示是  

    connection reset by peer。  

后来通过查看代理服务器这边的代码和业务逻辑服务器那边的代码后，发现是由于业务逻辑server那边在对一个端口设置监听的时候，对打开的socket设置了**TCP_DEFER_ACCEPT**这个选项，同时业务逻辑server这端对到来的tcp连接会在一个时间段后关闭这个连接。  

正常情况下，代理server这边有重连机制，当发现业务逻辑server那边关闭连接后，他会立即启动重连机制；但由于业务逻辑server这边对打开的socket设置了TCP_DEFER_ACCEPT选项，那么对于一个connect上来连接而言，它不会真正去完成tcp的三次握手进入establish状态，而是会对最后一个客户端的ack，仅仅把这个socket标记为acked，然后丢弃它，而对于这个socket，仍将是处于syn_recv状态；只有当客户端真正有数据到达的时候，内核才会去accept这个连接并且接收数据，到此时server这段的这个socket才会是establish状态。

在客户端client上来，到实际发送数据这段过程中，server这边的这个socket始终是syn_recv的，同时它会去重发syn/ack，当重传超过TCP_DEFER_ACCEPT指定的数值后(测试发现并不是十分精准，这个选项指定的是一个秒数，而内核会换算为重传的次数)如果还没数据到达，那么就会丢掉这个请求，并关闭连接。对地我们目前这个代理服务器来说，如果此时在通过它去请求后端逻辑，就会得到： connection reset by peer的错误了。

　　man 7 tcp中对这个选项的描述：  

    TCP_DEFER_ACCEPT   
 
> Allows a listener to be awakened only when data arrives on the socket. Takes an  integer value (seconds), this can bound the maximum number of attempts TCP will make to complete the connection. This option should not be used in code intended to be portable.

这个就是说如果设置了这个选项，将允许一个监听者只在有数据到达这个套接字的时候才会被唤醒，它将一个整型值（指定秒数）绑定为最大尝试次数，但是后边那个“TCP will make to complete the connection”到底如何理解，我没有很清楚。

看了一篇分析文章：[http://www.pagefault.info/?p=346](http://www.pagefault.info/?p=346)，那里边说是：“defer的连接将会被加入到establish队列”， 但是我通过测试后发现，到超过一定时间后，这个连接会被close掉，不知道这个make to complete the connection到底是怎么回事。

-EOF-