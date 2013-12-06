---
title: 深入分析tcp close与shutdown
layout: post
category: [linux-network-program]
tags: [network]
---

###关闭socket-close
我们知道，tcp是一种支持全双工(full-duplex)通信的的协议，也就是说建立连接的两端可以在同一个时刻发送、接受数据。在需要关闭套接字的时候，我们一般调用:  

	int close(int fd)  

调用close后，这个套接字描述符将不再指向任何文件继而可以被重复使用。同时调用close后，tcp连接断开，client端和server段都不能再发送/接受数据。  

###额外的选择-shutdown
但有时候我们可能只想关闭连接的一端，也就是说在连接的一端关闭读/写，指明其不再接受(read)或者发送(write)。  

例如，“客户端所有的数据都发送完毕，需要通知服务器自己已经发送完了所有的数据，服务器可以开始对客户端的数据进行处理；另外，客户端自己还需要接收来自服务器的响应”， 在这种情况下简单的使用close是无法做到的。需要使用另一个系统函数shutdow了：  
	
	int shutdown(int s, int how)  

首先对shutdown的参数做一点说明：  

1. int s: 套接字描述符  
2. int how: 有三个宏供我们选择  

	> SHUT_RD 读半部关闭，后续的读操作会被禁止
	    
	> SHUT_WR 写半部关闭，后续的写操作会被禁止
	  
	> SHUT_RDWR 读写都关闭，后续的读写操作都会被禁止  

回到上边的例子，如果客户端在发送完数据后要通知服务端，那么可以将该套接字的写半部关闭，即：  

	shutdown(fd, SHUT_WR)  

此时，tcp协议栈会给对端发送一个FIN分节，当服务段在进行read的时候，read会返回0（如果对一个已经关闭了写半部的套接字继续调用write,linux内核会给进程递交一个SIG_PIPE信号，并且errno会被设置为32：Broken pipe），之后，我们可以继续利用该套接字读取来自服务器的响应。代码示例如下：

	shutdown(sfd, SHUT_WR);
	int n = read(sfd, buf, buflen);  

示例代码为一个echo server, 完整的代码请参考：[echo](https://github.com/yuxingfirst/snipper/tree/master/echo)  

###close vs shutdown-套接字状态
通过这篇文章[TCP连接状态分析](http://yuxingfirst.github.io/posts/TCP-conn-status-analysis.html)我们知道close会触发四次挥手过程进而断开tcp连接；那么在调用shutdown的时候，socket的状态会怎么变化呢? 下边我们通过netstat来进行分析。

    void str_cli(FILE *fp, int sfd ) {
        char sendline[1024], recvline[2014];
        while( fgets(sendline, 1024, fp) != NULL ) {
           write(sfd, sendline, strlen(sendline)); 
           sleep(3);
           shutdown(sfd, SHUT_WR);
           sleep(3);
           if( read(sfd, recvline, 1024) == 0 ) {
                printf("server terminal prematurely.\n"); 
                return;
           }
           fputs(recvline, stdout);
        }
    }  

为了便于查看套接字的状态，我们在第一次给server发送完数据后休眠3秒；shutdown后（即client端的tcp发送完fin）同样也休眠3秒。运行程序后，我们查看到套接字的各个状态如下所示:

![img1](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/linux-network-program/half-close-1.png)
图1

![img2](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/linux-network-program/half-close-2.png)
图2

![img3](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/linux-network-program/half-close-3.png)
图3

根据上边几个截图我们可以看出，调用shutdown(fd, SHUT_WR)后，client会给server端发送一个fin，这时的套接字状态如图2，最后server在调用close关闭套接字后，套接字状态图变为图3.

###close vs shutdown-差别

从我们上边的实验看，好像shutdown与close一样，都会是tcp断开连接(发送断开连接的四组分节)，但实际的情况真的是这样吗?或者说，他们之间有有什么区别吗？下边我们来看代码示例。（完整代码看这里：[here](https://github.com/yuxingfirst/snipper/tree/master/dbcs)）  

	server.c
	--------
	
	for (;;) {
        socklen_t clilen = sizeof(child_addr);
        c = accept(s, (struct sockaddr *)&child_addr, &clilen);
	 	if( fork() == 0 ) {
			pid_t child = getpid();
			printf("In child process :%d\n", child);
			//首先调用close, 然后再注释掉close,调用shutdown
			close(c); 
	        //shutdown(c, SHUT_WR);
	        exit(0);
	    }
		//让子进程先执行
	    sleep(5);
		pid_t parent = getpid();
		printf("In parent process:%d\n", parent);
		const char *buf = "server parent process send data";
		int n = write(c, buf, strlen(buf));
		if(n < 0) {
			printf("%d", errno);
		}
		close(c);
	｝  

	client.c
	--------

	void str_cli(FILE *fp, int sfd ) {
	    char recvbuf[1024];	
	    while( read(sfd, recvbuf, 1024) > 0 ) {
	        fputs(recvbuf, stdout);
	    }
		printf("server send fin.\n");
	}

我们首先来看server.c的示例代码，fork调用完成后，实际已经存在两个进程了，并且来自客户端连接的socket（c）在两个进程中都存在了。为了测试close与shutdown的差别，在子进程中我们先后用close、shutdown关闭套接字(c)，然后在父进程中，使用套接字c给客户端发送数据。  

在第一次运行结束后，我们会发现，当子进程调用**close(c)**后，客户端并没有看到**printf("server send fin.\n")**这行代码的输出，说明它仍旧阻塞在read调用上，同时也进一步说明server端并没有发送fin分节，即子进程调用close并没有触发tcp断开连接（4个断开连接的分组没有发送）。  

在父进程sleep结束后，调用write给客户端发送数据，然后调用close关闭套接字，此时我们从client这边可以观察到server发送的字符串被打印在了stdout上，同时输出了：server send fin， 说明server端父进程调用close后，tcp协议栈发送了断开连接的4组分节。  

看到这里，我们会有一个疑问:"**为什么子进程中调用close没有发送断开连接的4组分节，而到父进程中调用close是才发送断开连接的4组分节呢？**"，带着这个疑问，我们先看看调用子进程中调用shutdown的情况。  

重新编译代码并运行，我们会发现，当子进程调用完shutdown后，客户端会马上收到一个fin分节，read返回，然后输出：server send fin；同时服务器这边休眠结束调用write失败，错误码显示32，信号处理函数执行。  

###揭秘  
综合上边的实验，我们不禁要问，造成这种差异的原因是什么呢？  

首先，我们知道，fork函数的特点是“调用一次，两次返回”，在父进程中调用一次，根据不同的返回值，在父进程和子进程中各返回一次。内核(linux/unix)根据父进程复制出一个子进程，父进程和子进程的PCB（Process Control Block）信息相同，用户态代码和数据也相同；当然，有父进程打开的描述符也都会被复制到子进程中，不过这里需要注意的是，父、子进程中相同编号的文件描述符在内核中指向同一个file结构体，也就是说，当复制子进程时，对于同一个文件描述符的file结构体需要将其引用计数增加1。  

对应到我们上边的例子，client连接的套接字描述符c虽然在父子进程中都存在，但是在内核只有一个对应的file结构体。也就是说，父子进程其实是共享一个tcp连接的。  

然后我们在子进程中调用close的时候，由于此时父进程中的描述符还引用了这个结构体，也可以说父进程中还持有这个tcp连接，所以内核不会真正不断开(break)这个连接，只是将对应的那个file结构体的引用计数减1；这个tcp连接还是继续存在的，其他的进程还是继续可以在这个连接上read/write数据的。（***这也就是我们在客户端没有收到fin的原因***）  

与close不同的是，在调用shutdown的时候，内核会真正的去断开(break)这个连接，tcp协议栈会发送断开连接的4组分节，所以此时在client会马上收到fin,此前阻塞的read函数返回。

-EOF-

[参考资料]

1. [What is the difference between close() and shutdown()](http://dev.fyicenter.com/Interview-Questions/Socket-4/What_is_the_difference_between_close_and_shutd.html)
2. [Fork (system call)](http://en.wikipedia.org/wiki/Fork_(system_call))
3. [The fork() System Call](http://www.csl.mtu.edu/cs4411.ck/www/NOTES/process/fork/create.html)
4. [shutdown(3) - Linux man page](http://linux.die.net/man/3/shutdown)

