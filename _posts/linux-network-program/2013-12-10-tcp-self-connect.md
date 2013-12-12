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

首先考虑下如何重现。根据上边介绍的情况，其实就相当于一直连接一个并没有被监听的端口，所以我们可以这样:  
	
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

当然，我们也可以在调用connect之前，调用bind(2)给这个套接字绑定一个端口(source port)，当通常很少这样做。在这种情况下，给这个套接字绑定source port的工作，就交给内核去做了。内核通常会从一个叫做Ephemeral port的范围内，**循环**地选择一个端口分配给这个套接字。   

> 请注意我这里提到的循环的选择端口，至于是如何循环的?下边我再解释.  
 
那么，什么叫做Ephemeral port呢? 我们看wikipedia的解释, [Ephemeral port][6]:  

	An ephemeral port is a short-lived transport protocol port for 
	Internet Protocol (IP) communications allocated automatically from a predefined 
	range by the IP software.

其实，Ephemeral port就是一种在IP连接中的一种短暂生命期的端口，可以在一个预先定义的范围内自动分配。对应到linux系统中，我们可以使用下述命令，查看我们机器中的ephemeral ports的范围:  

	$ cat /proc/sys/net/ipv4/ip_local_port_range  
	32768	61000  

在我自己的机器上，ephemeral ports的范围是：32768至61000。   

再回到我们telnet的例子，当我们运行脚本后,telnet启动，内核在指定的ephemeral ports范围内，选择一个端口作为source port,然后就尝试连接我们指定的：127.0.0.1 50000， 由于并没有进程在50000端口在监听，所以tcp会回复一个RST分节，同时telnet收到这个分节并提示:Connection refused。  

我们可以用tcpdump查看这一过程:  

![img3][self_connect_tcpdump_rst]

如图所示，每个客户端tcp发送一个SYN分节后，服务端tcp回复一个RST。同时，这里还需要注意的是，每组请求/响应中请求的源端口,从图中看出，依次是  
	
	36561
	36562
	36563
	36564
	36565
	36566
	36567

这里也验证了我前边提到的: 内核循环的选择一个ephemeral port分配给套接字。  

正是由于这种循环迭代的选择端口，并且我们要连接的目的端口是50000，在ephemeral ports的范围内，所以最终会出现源端口跟目的端口相同的情况。  

那么发起一个源端口跟目的端口相同的连接为什么也能成功建立连接呢? 

我们先看下图，是tcp的各种状态转换:  

![img4][tcp-stat]  

首先当源端口被选择为50000时，tcp立即初始化为CLOSED状态；然后，telnet尝试连接 127.0.0.1:50000这个server，这里意味着tcp协议栈会开始三路握手过程，发送一个SYN分节，此时源端口进入:SYN_SENT 状态；由于这里目的端口与源端口相同，所以，先前发出的那个SYN分节会在50000这个端口上被接收到，此时这个端口的tcp状态转换为:SYN RECEIVED,同时呢，它会发送一个SYN+ACK；同样的，这个SYN+ACK同样会在50000这个端口被接收到；为了使tcp进入ESTABLISHED状态，还需要最后一个ACK。看到这里，有两个问题需要解决:

1. 为什么在一个没有被监听的端口上接到SYN，不发送RST，而是发送SYN+ACK?  
2. tcp状态是如何转变成为ESTABLISHED以及最后那个ACK是什么时候发送的来完成三路握手的呢?  

下边，我将从[RFC793](http://www.ietf.org/rfc/rfc793.txt "RFC793")和linux内核源码[tcp_input.c](https://github.com/yuxingfirst/linux/blob/master/net/ipv4/tcp_input.c)这两个方面来分析这两个问题。

###RFC793
---------

我们知道正常的client -> server需要经过三次握手(three-way handshake),如下所示:

	 TCP A                                                TCP B

  	1.  CLOSED                                               LISTEN

  	2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               --> SYN-RECEIVED

 	3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED

  	4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED

  	5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED 

TCP A为客户端，TCP B为服务器端，TCP A对正在监听的TCP B发起连接请求。  

除了这种经典的连接交互之外，tcp(Transmission Control Protocol)还提供了另外一个特性:  
	
	simultaneous open  

那么什么是"simultaneous open"呢? 其实就是假设在tcp连接中存在这样一种情况:  
	
> 相互独立的两台主机的两个tcp 套接字, 可能会在同一时刻向对方发起连接。在这种情况下，就跟我上边说的经典连接模型不一样了。  

我们来看下RFC793中对这一情况的描述:  

	Simultaneous initiation is only slightly more complex, as is shown in
	figure 8.  Each TCP cycles from CLOSED to SYN-SENT to SYN-RECEIVED to
	ESTABLISHED.

      TCP A                                            TCP B

	  1.  CLOSED                                           CLOSED
	
	  2.  SYN-SENT     --> <SEQ=100><CTL=SYN>              ...
	
	  3.  SYN-RECEIVED <-- <SEQ=300><CTL=SYN>              <-- SYN-SENT
	
	  4.               ... <SEQ=100><CTL=SYN>              --> SYN-RECEIVED
	
	  5.  SYN-RECEIVED --> <SEQ=100><ACK=301><CTL=SYN,ACK> ...
	
	  6.  ESTABLISHED  <-- <SEQ=300><ACK=101><CTL=SYN,ACK> <-- SYN-RECEIVED
	
	  7.               ... <SEQ=101><ACK=301><CTL=ACK>     --> ESTABLISHED

                Simultaneous Connection Synchronization

TCP A和TCP B同时发送SYN，两者同时进入SYN-SENT状态，然后又分别收到了对方的SYN于是又进入SYN-RECEIVE，随后，又立即发送了SYN+ACK，最后两者都接收到ACK于是进入ESTABLISHED状态。我们的例子是destination port 和 source port是同一个端口，跟上边的这种情况有点差别，不过也同样可以适用。  

###linux内核 net/ipv4/tcp_input.c
--------------------  
	
	 5681: static int tcp_rcv_synsent_state_process()

	 5861: if (th->syn) {
     5862:        /* We see SYN without ACK. It is attempt of
     5863:         * simultaneous connect with crossed SYNs.
     5864:         * Particularly, it can be connect to self.
     5865:        */
     5866:        tcp_set_state(sk, TCP_SYN_RECV);
	
				  //...
	 5893: 		  tcp_send_synack(sk);

请看上边这个代码片段, 从函数名可以得知，这个函数是在tcp state转换为SYN-SENT后执行的。前边有提到过，tcp state在发送完SYN后进入SYN-SENT；然后，当程序执行到5861这行代码时，它看到了一个syn, 于是在5866这行将tcp state设置为TCP_SYN_RECV。然后，在5893行处发送SYN+ACK。  

> 此外，从代码的注释我们可以看到，此处接受到的是一个没有ACK的SYN，说明不是来自server的回射，它这里认为是simultaneous connect中的交叉SYN。然后，注释中还特别提到，这里有可能是self-connect。  

然后，发送出去的SYN+ACK将回送回来并且在tcp_rcv_state_process这个函数中进行处理:  

	5931: int tcp_rcv_state_process()	

	5991: if (!tcp_validate_incoming(sk, skb, th, 0))
    5992:           return 0;
	5993: 
    5994: /* step 5: check the ACK field */
    5995: if (th->ack) {
    5996:         int acceptable = tcp_ack(sk, skb, FLAG_SLOWPATH) > 0;
	5997: 
    5998:         switch (sk->sk_state) {
    5999:         case TCP_SYN_RECV:
    6000:               if (acceptable) {
    6001:                       tp->copied_seq = tp->rcv_nxt;
    6002:                       smp_mb();
    6003:                       tcp_set_state(sk, TCP_ESTABLISHED);

在tcp_validate_incoming做一些检验，然后会发送最后一个ack。随后，在第6003行，将tcp sate设置为ESTABLISHED。

好了，通过上边的分析，我们大致明白了self-connect产生的原因；同时我们也知道了要产生这种现象，需要一些前提条件:  

1. 我们的client程序连接的端口需要在ephemeral ports的范围内。即/proc/sys/net/ipv4/ip_local_port_range这个文件配置的端口范围。  
2. client和server需要在同一台主机。  

所以为了避免这个问题，我们在给我们的一些server程序选择端口的时候，尽量不要用处于ephemeral ports范围内的端口；否则，一旦出现这种情况，就有可能很难定位了。  

参考资料:  

1. [net/ipv4/tcp_input.c](https://github.com/yuxingfirst/linux/blob/master/net/ipv4/tcp_input.c "tcp_input.c")  
2. [Ephemeral_port wikipedia](http://en.wikipedia.org/wiki/Ephemeral_port "Ephemeral_port")  
3. [TCP client self connect...](http://sgros.blogspot.com/2013/08/tcp-client-self-connect.html "self connect")  
4. [RFC793](http://www.ietf.org/rfc/rfc793.txt "RFC793")

-EOF-

[self_connect_telnet]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/self-connects-telnet.png "self-connect telnet test"  
[self_connect_netstat]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/self-connects-netstat.png "self-connect netstat"  
[self_connect_tcpdump_rst]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/self_connect_tcpdump_rst.png "tcpdump"  
[tcp-stat]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/tcp-stat.png  "tcp-stat"

[1]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0510.html "linux kernel mail list"  
[2]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0510.html
[3]: http://en.wikipedia.org/wiki/Postel%27s_law "Robustness principle's wikipedia"  
[4]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0438.html  
[5]: http://lkml.indiana.edu/hypermail/linux/kernel/9909.3/0510.html  
[6]: http://en.wikipedia.org/wiki/Ephemeral_port "Ephemeral port"
