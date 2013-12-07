---
title: socket超时设置
layout: post
category: [linux-network-program]
tags: [network]
--- 

我们知道，对于一个套接字的读写(read/write)操作默认是阻塞的，如果当前套接字还不可读/写，那么这个操作会一直阻塞下去，这样对于一个需要高性能的服务器来说，是不能接受的。所以，我们可以在进行读写操作的时候可以指定超时值,这样就读写操作就不至于一直阻塞下去。

在涉及套接字的I/O操作上设置超时的方法有三种：

1. 调用alarm，它在指定的超时期满时产生SIGALRM信号。这个方法涉及信号处理，而信号处理在不同的实现上存在差异，而且可能干扰进程中现有的alarm调用。

2. 在select中阻塞等待I/O(select有内置的时间限制)，依次代替直接阻塞在read或write调用上。(linux2.6以后的内核也可以使用epoll的epoll_wait)

3. 使用较新的SO_RCVTIMEO和SO_SNDTIMEO套接字选项。这个方法的问题在于并非所有的实现都支持这两个套接字选项。

上述这三个技术都适用于输入和输出操作(read、write，及其变体recv/send, readv/writev, recvfrom,sendto)。  

不过我们也期待可以用于connect的技术，因为TCP内置的connect超时相当长(**典型值为75秒**)，而我们在写服务器程序的时候，也不会希望一个连接的建立需要花费这么长时间。select可用来在connect上设置超时的先决条件是相应的套接字是非阻塞的，而那两个套接字选项对connect并不适用；同时也应当指出，前两个技术适用于任何描述符，而第三个技术仅仅适用于套接字描述符。

下边我们主要展示利用select进行connect超时设置的实例(比较常用)：  

	#include <stdio.h>
	#include <stdlib.h>
	#include <errno.h>
	#include <string.h>
	#include <netdb.h>
	#include <sys/types.h>
	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <unistd.h>
	#include <fcntl.h>
	#include <sys/select.h>
	#include <time.h>
	#define PORT 9900
	#define MAXDATASIZE 5000
	int main(int argc,char **argv)
	{
	    int sockfd,nbytes;
	    char buf[1024];
	    struct hostent *he;
	    struct sockaddr_in srvaddr;
	    if(argc!=2)
	    {
	        perror("Usage:client hostname\n");
	        return 0;
	    }
	    if((he=gethostbyname(argv[1]))==NULL)
	    {
	        perror("gethostbyname");
	        return 0;
	    }
	    if((sockfd=socket(AF_INET,SOCK_STREAM,0))==-1)
	    {
	        perror("create socket error");
	        return 0;
	    }
	    bzero(&srvaddr,sizeof(srvaddr));
	    srvaddr.sin_family=AF_INET;
	    srvaddr.sin_port=htons(PORT);
	    srvaddr.sin_addr=*((struct in_addr *)he->h_addr);
	    fcntl(sockfd, F_SETFL, O_NONBLOCK);
	    timeval timeout = {3, 0};
	    if(connect(sockfd,(struct sockaddr *)&srvaddr,sizeof(struct sockaddr)) == -1)
	    {
	        if( errno != EINPROGRESS ) {
	            close(sockfd);
	            perror("connect error");
	            return 0;
	        }
	    }
	 
	    fd_set readSet;
	    FD_ZERO(&readSet);
	    FD_ZERO(&writeSet);
	    FD_SET(sockfd, &writeSet);
	    int ret = select(sockfd + 1, &readSet, &writeSet, NULL, &timeout);
	    printf("%d", ret);    
	}     

在打开套接字然后调用connect的时候，由于我们对套接字设置了O_NONBLOCK选项，所以此时connect不会阻塞，通常是会返回-1，然后errno被设置为EINPROGRESS，表示连接仍在进行中，所以在后边，我们只要将这个套接字注册到select的可写集合中(writeSet，因为这里是去连接对端服务器)，然后在调用select的时候设置超时值，如本例中设置为3秒；那么在3秒后，如果连接仍未建立，那么select将返回0，表示超时；如果在3秒内连接成功建立，套接字变为可写状态，那么select将返回1。  

> 注：关于另外两种方式，可以通过阅读：《unix网络编程》第一卷第14章。
