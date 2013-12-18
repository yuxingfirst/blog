---
title: APUE读书笔记-Chapter8-进程控制
layout: post
categories: [读书笔记]
tags: [c, unix]
description: 进程控制.
---

###进程标识符  
每个进程都有一个非负整数表示的唯一进程ID。虽然是唯一的，但是进程ID可以重用。当一个进程终
止后，其进程ID就可以再次使用了。大多数UNIX系统实现延迟重用算法，使得赋予新建进程的ID不同
于最近终止进程所使用的ID。这防止了将新进程误认为是使用同一ID的某个已终止的先前进程。  

系统中有一些专用的进程。ID为0的进程通常是调度进程。进程ID1通常是init进程，在自举过程结束
时内核调用。

###fork  
由fork创建的新进程被称为子进程。fork函数被调用一次，但返回两次，两次返回的唯一区别是子进
程的返回值是0，而父进程的返回值则是新子进程的进程ID。  

子进程和父进程继续执行fork调用之后的指令。子进程是父进程的副本。例如：子进程获得父进程数
据空间、堆和栈的副本。父子进程并不共享这些存储空间部分。父、子进程共享正文段。

由于在fork之后经常跟随着exec，所以现在的很多实现并不执行一个父进程数据段、栈和堆的完全复
制。作为替代，使用了写时复制(copy-on-write)技术。  

下边我们通过一个例子看下fork的使用方式。  
	
	//include...
	
	int glob = 6;
	char buf[] = "a write to stdout\n";
	
	int main(void) {
		int var;
		pid_t pid;
		var = 88;
		if(write(STDOUT_FILENO, buf, sizeof(buf)-1) != sizeof(buf)-1) {
			err_sys("write error.");
		}	
		printf("before fork\n");
		
		if((pid = fork()) < 0) {
			err_sys("fork error");
		} else if(pid == 0) {	//child
			glob++;
			var++;
		} esle {
			sleep(2);
		}
		printf("pid=%d, glob=%d, var=%d\n", getpid(), glob, var);
		exit(0);
	}

调用fork后是父进程先执行还是子进程先执行是不确定的。着取决于内核所使用的调度算法。  

当写到标准输出时，将buf长度减去1作为输出字节数，这是为了避免将终止null字节写出。strlen计算不包含终止null字节的字符串长度，而sizeof则计算包含终止null字节的缓冲区长度。两者之间的另一个差别是，使用strlen需进行一次函数调用，而对于sizeof而言，因为缓冲区已用已知字符串进行的初始化，其长度是固定的，所以sizeof在编译时计算缓冲区长度。  

上面的程序需要注意的是fork于I/O函数之间的交互关系。并且需要注意的是：  
	
> write是不带缓冲的，标准I/O函数是带缓冲的。  

> 如果标准输出连到终端设备，则它是行缓冲的，否则是全缓冲的。此外，我们还是通过设置让标准输出无缓冲。

> note: 可以对上述代码稍作修改然后编译，以两种不同的方式运行，观察输出的不同。

	$ ./a.out
	$ ./a.out > temp.out  

###exit
在这篇中[进程环境](http://yuxingfirst.github.io/posts/process-terminal.html)我介绍
了进程终止的几种方式。  不管进程如何终止，最后都会执行内核中同一段代码。这段代码为相应进程关闭所有打开的描述符，
释放它所使用的存储器等。   
对于任何一种终止情形(正常/异常)，我们都希望终止进程能够通知其父进程它是如何终止的。对于三个终止函数(exit, _exit, _Exit)，实现这一点的方法是，将其退出状态(exit status)作为参数传送给函数。在异常终止的情况下，内核(不是进程本身)产生
一个指示其异常终止原因的终止状态(termination status)。在任意一种情况下，该终止进程的父进程都能用wait或waitpid函数取得其终止状态。   

> 如果父进程在子进程之前终止，则这些父进程的所有子进程的父进程都变为init进程，我们称这些进程由init领养。  

###wait、waitpid
当一个进程正常或者异常终止时，内核就向其父进程发送SIGCHLD信号。因为子进程终止是个异步事件，所以这种信号也是内核向
父进程发的异步通知。调用wait和waitpid的进程会有以下三种情况发生：  

1. 如果其所有子进程都还在运行，则阻塞。  
2. 如果一个子进程已终止，正等待父进程获取其终止状态，则取得该子进程的终止状态立即返回。    
3. 如果没有任何 ，则立即出错返回。  

我们看下这两个函数的原型:  

    #include <sys/wait.h>
	pid_t wait(int *statloc);
	pid_t waitpid(pid_t pid, int *statloc, int options);

其区别如下:  

1. 在一个子进程终止前，wait使其调用者阻塞，而waitpid有一个选项，可使调用者不阻塞。  
2. waitpid并不等待在其调用之后的第一个终止子进程，他有若干个选项，可以控制他所等待的进程。  

如果一个子进程已经终止，并且是一个僵死进程，则wait立即返回并取得该子进程的状态，否则wait使其调用者阻塞直到
一个子进程终止。如调用者而且他有多个子进程，则在其一个子进程终止时，wait就立即返回。因为wait返回终止子进程的进程ID，所以它总能了解是哪一个子进程终止了。  

在我的这篇文章中“[linux僵尸进程产生的原因及如何避免产生僵尸进程](http://yuxingfirst.github.io/posts/linux-zombile-process-analysis.html)”介绍了避免产生僵死进程的方法，这篇中的方法是通过信号处理的方式来解决的，这里我将在介绍一种方式，看下述代码：  
	
	//include...
	#include<sys/wait.h>  
	
	int 
	main(void) {
		pid_t pid;
		
		if((pid = fork()) < 0) {
			err_sys("fork error");		
		} else if(pid == 0) {	//first child
			if((pid = fork()) < 0) {
				err_sys("fork error");		
			} else if(pid > 0) {
				exit(0);	/*这里退出第一个子进程*/
			}
			/*到这里是第二个子进程开始执行，当第二个子进程的父进程(first child)调用exit后，
				第二个子进程的父进程将变成init，当这个进程执行完推出后，init会获取其状态。
			*/
			sleep(2);//确保其父进程(first child)退出
			exit(0);
		}
		if(waitpid(pid, NULL, 0) != pid) {	//等待第一个子进程, first child
			err_sys("waitpid error");		
		}
		exit(0);
	}  

如上，我们调用fork两次，利用这一技巧就可以避免产生僵尸进程。  

###进程时间

我们可以测量的三种时间，墙上时钟时间、用户CPU时间和系统CPU时间。任一进程都可以调用times函数以获得它自己及已终止子进程的这三种时间。  

	#include <sys/time.h>
	
	clock_t times(struct tms *buf);
				返回值：若成功则返回流逝的墙上时钟时间(单位：时钟滴答数)，若出错则返回-1.
	
	//tms结构定义如下
	
	struct tms {
		clock_t tms_utime;	//user CPU time
		clock_t tms_stime;	//sys CPU time
		clock_t tms_cutime;	//user CPU time, terminated children
		clock_t tms_cstime;	//system CPU time, terminated children
	};  

> note 所有此函数返回的clock_t值都可以用_SC_CLK_TCK变换成秒数。  

-EOF-















