---
title: Unix进程通信-pipes
layout: post
categories: [读书笔记]
tags: [进程通信]
description: Unix进程通信之管道(pipes)
---

pipes是在所有unix系统中均被支持的最老的进程通信方式。同时，它也有两个限制:  

1. 半双工通信(half duplex)。  
2. pipes只能在拥有共同祖先的两个进程之间使用。(例如:  进程A创建了一个管道pa, 然后进程Afork了进程B，那么pipes pa能够在进程A、B之间使用)  

我们可以调用pipe来创建一个管道:  

	#include <unistd.h>
	int pipe(int filedes[2]);  
				//returns 0 ok; -1 on error  

我们注意到pipe的参数: int filedes， pipe内部借助这个参数给调用者返回一对文件描述符，filedes[0]用于读(read)操作；filedes[1]用于写(write)操作。也就是说，filedes[1]的输出是filedes[0]的输入。  

> pipes在4.3BSD, 4.4BSD和Mac OS X 10.3上是基于unix domain socket实现的，但是，虽然unix domain socket是全双工通信的，但是这些系统在实现pipes的时候仍旧只用半双工方式操作。

如下图描绘的半双工管道, 左半图显示了管道的两端在一个进程中相互连接，右半图则说明数据通过内核在管道中流动。

![img1][half-duplex]  

单个进程中的管道几乎没有任何用处。通常，调用pipe的进程接着调用fork，这样就创建了从父进程到子进程(或反向)的IPC通道。下图显示了这种情况:  

![img2][pipe-after-fork]  

调用fork之后做什么取决于我们想要有的数据流的方向。对于从父进程到子进程的管道，父进程关闭管道的读端(fd[0])，子进程则关闭写端(fd[1])。下图显示了在此之后描述符的安排。  

![img3][parent-to-child]  

为了构造从子进程到父进程的管道，父进程关闭fd[1]，子进程关闭fd[0]。  

当管道的一端被关闭后，下列两条规则起作用:   

1. 当读一个写端已被关闭的管道时，在所有数据都被读取后,read返回0，以指示达到了文件结束处。  
2. 写一个读端以被关闭的管道，则产生信号SIGPIPE。如果忽略该信号或者捕获该信号并从其处理程序返回，则write返回-1， errno设置为EPIPE。  

我们知道, 当我们fork一个子线程的时候，父子进程的运行顺序是不确定的。这里，我们通过管道来实现父、子进程同步的例子:  

	#include <stdio.h>
	#include <stdlib.h>
	#include <unistd.h>
	#include <errno.h>

	static int pfd1[2], pfd2[2];

	void TELL_WAIT() 
	{
		    if(pipe(pfd1) < 0 || pipe(pfd2) < 0) {
		            printf("TELL_WAIT errno:%d\n", errno);
		            exit(0);
		    }
	}

	void TELL_PARENT(pid_t pid) 
	{
		    if(write(pfd2[1], "c", 1) != 1) {
		            printf("TELL_PARENT errno:%d\n", errno);
		    }
	}

	void WAIT_PARENT(void)
	{
		    char c;
		    if(read(pfd1[0], &c, 1) != 1) {		//阻塞
		            printf("WAIT_PARENT errno:%d\n", errno);
		    }

		    if(c != 'p') {
		            printf("WAIT_PARENT incorrect data\n");
		            exit(0);
		    }
	}

	void TELL_CHILD(pid_t pid) 
	{
		    if(write(pfd1[1], "p", 1) != 1) {	
		            printf("TELL_CHILD errno:%d\n", errno);
		    }   
	}

	void WAIT_CHILD(void) 
	{
		    char c;
		    if(read(pfd2[0], &c, 1) != 1) {		//阻塞
		            printf("WAIT_CHILD errno:%d\n", errno);
		    }

		    if(c != 'c') 
		    {
		            printf("WAIT_CHILD incorrect data\n");
		            exit(0);
		    }
	}

	static void charatatime(char *str);

	int main(void) 
	{
		pid_t pid; 
		TELL_WAIT();

		if((pid = fork()) < 0) {
		    exit(0); 
		} else if(pid == 0) {   //child
		    //WAIT_PARENT(); 
		    charatatime("output from child\n");
		    TELL_PARENT(pid);
		} else {                //parent
		    WAIT_CHILD();
		    charatatime("output from parent\n");
		    //TELL_CHILD(pid);
		}
		return 0;
	}

	static void charatatime(char *str) {
		char *ptr;
		int c;
		setbuf(stdout, NULL);
		for(ptr = str; (c = *ptr++) != 0;) {
		    putc(c, stdout); 
		}
	}

通过上边的代码就可以控制父子进程的运行顺序。因为在WAIT_XXX函数中调用read会发生阻塞，直到某个进程在相应管道中写入数据。  

不过，有一点需要注意的是，每一个管道都会有一个额外的读取进程，也就是说，子进程从pfd[0]读取，父进程上也这个管道的读端,不过由于父进程并没有执行对该管道的读操作，所以不会有任何影响。不过，比较好的做法是在父子进程中关闭相应的管道的某一端。  
比如，如果父进程在pfd1[0]这端读，子进程在pfd1[1]写，那么就要父进程中关闭pfd1[1]，子进程中关闭pfd1[0];   

[half-duplex]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/read-notes/half-duplex.png  
[pipe-after-fork]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/read-notes/pipe-after-fork.png  
[parent-to-child]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/read-notes/parent-to-child.png  

-EOF-


