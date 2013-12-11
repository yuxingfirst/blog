---
title: TCP self-connects
layout: post
categories: [网络编程]
tags: [network]
description: 连接自身的tcp连接.
---

我们知道，在一些基于tcp的网络服务器中，譬如代理服务器,一般都要处理断线重连的情况。某种极端
的情况下，当后端server(db server, nosql server)宕机的时候，如果不做一些额外的处理，此时代理
服务器可能会一直处于重连的循环中。   

最初遇到这个问题，还是在去年，当时组里有一个redis的代理模块，一个同事为了测试该模块的
断线重连功能， kill掉redis进程，然后查看模块连接redis的情况。由于该模块一直在进行重连并且此时redis
服务不可用(没有将进程启动起来)，一段时间后，竟然出现了该代理模块连上了自己，用netstat查看，发现
Local Address和Foreign Address竟然都是一样的。不过后来由于一些事情，这个问题也没有深究。   

恰好今天在一篇帖子上有提到 Linux Tcp Self-connection， 这里就对这个问题做点研究。  

首先考虑下如何重现。像上段介绍的情况，其实就相当于一直连接一个并没有被监听的端口，所以我们可以这样:  
	
	//self_connect.sh
	#!/bin/bash
	while true
	do
		telnet 127.0.0.1 50000
	done

按正常逻辑来说，应该一直会提示connection refused，因为此时50000端口并没有被监听，连接不可能完成。那么真实
情况是这样的吗? 好，运行脚本：  

	$ sh self_connect.sh

之后，我们会看到提示:

	Trying 127.0.0.1...
	telnet: Unable to connect to remote host: Connection refused  

不过，等待一段时间后，竟然连上了，用netstat查看连接状态，竟然是ESTABLISHED, 如下图所示:  

![img1][self_connect_telnet]

![img2][self_connect_netstat]  

看到这里，我们可能会感到非常奇怪?怎么会自己连接自己并且还能成功建立连接呢?  

通过google，我发现这个问题在linux kernel maillist有过讨论:[tcp/ip bug (2.2.12) or telnet client bug][4]，其中Craig Milo Rogers做出了一些[解答][5]。  

为了彻底搞清楚这个问题出现的原因，首先，我们先来看看linux内核的端口分配规则。 

我们知道，一个tcp连接由四个元组唯一标识，分别是：  
	
	(source IP, source port, destination IP, destination port)  

所以，当一个client(如上边所述的telnet程序)创建一个套接字(socket)并且尝试连接到一个server的时候，source IP, destination IP, destination port，这三个参数已经是确定了的。唯一没有确定的就是source port， 因为一般情况下，我们在写客户端程序连接服务器的时候，一般这样做:

	int fd = socket(...);
	connect(fd, ...);  

当然，我们也可以在调用connect之前，调用bind(2)给这个套接字绑定一个端口(source port)，当通常很少这样做。在这种情况下，给这个套接字绑定source port的工作，就交给内核去做了。内核通常会从一个叫做Ephemeral port的范围内，循环地选择一个端口分配给这个套接字。  

那么，什么叫做Ephemeral port呢? 我们看wikipedia的解释, [Ephemeral port][6]:  

	An ephemeral port is a short-lived transport protocol port for 
	Internet Protocol (IP) communications allocated automatically from a predefined 
	range by the IP software.

其实，Ephemeral port就是一种在IP连接中的一种短暂生命期的端口，可以在一个预先定义的范围内自动分配。对应到linux系统中，我们可以使用下述命令，查看我们机器中的ephemeral ports的范围:  

	$ cat /proc/sys/net/ipv4/ip_local_port_range  
	1024	65000  

在我自己的机器上，ephemeral ports的范围是：1024至65000 


-EOF-

[self_connect_telnet]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/self-connects-telnet.png "self-connect telnet test"  
[self_connect_netstat]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/self-connects-netstat.png "self-connect netstat"  

[1]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0510.html "linux kernel mail list"  
[2]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0510.html
[3]: http://en.wikipedia.org/wiki/Postel%27s_law "Robustness principle's wikipedia"  
[4]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0438.html  
[5]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0510.html  
[6]: http://en.wikipedia.org/wiki/Ephemeral_port "Ephemeral port"
