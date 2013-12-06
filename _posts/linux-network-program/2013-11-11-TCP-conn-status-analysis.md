---
title: TCP连接状态分析
layout: post
category: [linux-network-program]
tags: [network]
---

说起TCP，我们一般都需要知道发起一个tcp连接和终止一个tcp连接是所发生的事情，下边，我将跟大家介绍下tcp的三次握手及四次挥手的过程。  

###TCP三路握手  

1. 服务器必须准备好接受外来的连接。这通常在调用socket，bind，listen这三个函数来完成，我们称之为被动打开(passive open)。  
2. 客户通过调用socket，connect发起主动打开(active open)。这导致客户tcp发送一个SYN（同步）分节，它告诉服务器客户将在待建立的tcp连接中发送数据的初始序列号。通常SYN分节不携带数据，其所在的IP数据报只包含有一个IP首部、一个TCP首部及可能有的TCP选项。  
3. 服务器必须确认(ACK)客户的SYN，同时自己也得发送一个SYN分节，它含有服务器将在同一连接中发送的数据的初始序列号。服务器在单个分节中发送SYN和对客户SYN的ACK(确认)。  
4. 客户必须确认服务器的SYN。  

 这种交换至少需要三个分组，因此称之为TCP的三路握手,如下图：  

![img1](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/linux-network-program/tcp-sanluweoshou.jpg)  

###TCP四路挥手  
TCP建立一个连接需要3个分节，而终止一个连接一般需要4个分节，所以一般也称为四路挥手。  

1. 某个应用进程首先调用close，我们称为该端执行主动关闭(active close)，该端的tcp于是发送一个FIN分节，表示数据发送完毕。  
2. 接收到这个FIN的对端执行被动关闭(passive close)。这个FIN由tcp确认。它的接受也作为一个文件结束符传递给接收端的应用进程（放在已排队等候该应用进程接受的任何其他数据之后），因为FIN的接受意味着接收端应用进程在相应的连接上再无额外数据可接受。  
3. 一段时间后，接受到这个文件结束符的应用进程将调用close关闭它的套接字。这导致它的tcp也发送一个FIN分节。  

> 注：为什么是需要一段时候后，我的理解是由于接收到的fin分节作为一个文件结束符放在已排队等候应用进程接受的所有数据之后，而应用进程调用read读取数据得等它把这个文件描述符之前的所有数据读取完后，才能读取到该文件描述符，此时read返回0， 所以此时应用进程才知道对端已关闭了套接字，进而它也调用close关闭i套接字。  

 4.接受这个最终FIN的原发送端tcp（即执行主动关闭的那一端）确认这个FIN。  

 如图展示的这些分组：  

![img2](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/linux-network-program/tcp-sicihuishou.png)  

###TCP连接发起和终止时套接字的状态

tcp为一个连接定义了11中状态，并且tcp规定如何基于当前状态及在该状态下所接收的分节从一个状态转换到另一个状态。下边我们通过图3来展示tcp连接的分组交换情况及tcp套接字的状态：  

![img3](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/linux-network-program/tcp-zhuangtai.gif)

 一旦建立一个连接，客户就构造一个请求并发送给服务器。服务器处理该请求并发送一个应答，需要注意的是，服务器对客户请求的确认是伴随器应答发送的。这种做法称为捎带(piggybacking)，它通常在服务器处理请求并产生应答的时间少于200ms时发生。如果服务器耗用更长时间，如1s，那么我们将看到先是确认后是应答。  

接下来展示了终止tcp连接时交换的4个分节，从图中我们可以看到，发送一个单分节请求和接受一个单分节的应答，使用tcp至少需要8个分节的开销；如果改用udp，那么只需要交换两个分组：一个承载请求，一个承载应答。不过从tcp切换到udp也失去了tcp提供给应用进程的全部可靠性，迫使应用进程来做可靠性处理。  
