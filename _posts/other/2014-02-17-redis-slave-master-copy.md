---
title: Redis主从复制机制
layout: post
categories: [开源存储]
tags: [redis]
description: .
---

redis主从复制配置和使用都非常简单。通过主从复制可以允许多个slave server拥有和master server相同的数据库副本。下面是关于redis主从复制的一些特点  

1. master可以有多个slave  
2. 除了多个slave连到相同的master外，slave也可以连接其他slave形成图状结构  
3. 主从复制不会阻塞master。也就是说当一个或多个slave与master进行初次同步数据时，master可以继续处理client发来的请求。相反slave在初次同步数据时则会阻塞不能处理client的请求。
4. 主从复制可以用来提高系统的可伸缩性,我们可以用多个slave 专门用于client的读请求，比如sort操作可以使用slave来处理。也可以用来做简单的数据冗余
5. 可以在master禁用数据持久化，只需要注释掉master 配置文件中的所有save配置，然后只在slave上配置数据持久化。  


下面介绍下主从复制的过程：

当设置好slave服务器后，slave会建立和master的连接，然后发送`sync`命令。无论是第一次同步建立的连接还是连接断开后的重新连 接，master都会启动一个后台进程，将数据库快照保存到文件中，同时master主进程会开始收集新的写命令并缓存起来。后台进程完成写文件 后，master就发送文件给slave，slave将文件保存到磁盘上，然后加载到内存恢复数据库快照到slave上。  

接着master就会把缓存的命令转发给slave。而且后续master收到的写命令都会通过开始建立的连接发送给slave。从master到slave的同步数据的命令和从 client发送的命令使用相同的协议格式。  

当master和slave的连接断开时slave可以自动重新建立连接。如果master同时收到多个slave发来的同步连接命令，只会使用启动一个进程来写数据库镜像，然后发送给所有slave。  

配置slave服务器很简单，只需要在配置文件中加入如下配置
	
	slaveof 192.168.1.1 6379  #指定master的ip和端口

-EOF-