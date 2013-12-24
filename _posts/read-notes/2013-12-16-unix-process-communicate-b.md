---
title: popen和pclose
layout: post
categories: [Unix]
tags: [进程通信, 读书笔记]
description: popen和pclose介绍。  
---

上边介绍了unix进程通信的其中一个方式:pipe， 不过如果直接使用原生的函数的话，需要我们自己
去fork进程，关闭管道的不使用端等等。这里，介绍标准I/O库提供的两个函数:  

	FILE* popen(const char *cmdstring, const char *type);
					//成功返回文件指针,出错则返回NULL
	int pclose(FILE *fp);
					//返回cmdstring的终止状态,出错则返回-1  

函数popen会先执行fork，然后调用exec以执行cmdstring(通常是执行一个shell以运行命令)，并
且返回一个标准I/O文件指针。type标识返回的文件指针可以进行的操作。  

1. 如果type为“r”，则文件指针连接到cmdstring的标准输出;即返回的文件指针是可读的。  
2. 如果type为"w"，则文件指针连接到cmdstring的标准输入;即返回的文件指针是可写的。  

pclose函数关闭标准I/O流，等待命令执行结束，然后返回shell的终止状态。通常，在pclose内部，
会调用waitpid等待子进程执行结束，不过，通过我自己测试发现，如果子进程发生异常，那么则waitpid
不会阻塞而立即返回。

下边我们看一个示例程序:  

	//test.sh
	#!/bin/bash
	echo "before sleep"
	sleep 3

	//popentest.c
	#include <stdio.h>
	#include <sys/wait.h>
	#include <unistd.h>

	int main(int argc, char **argv) {
		char *filename = argv[1];
		char *mode = argv[2];
		printf("execute filename:%s, %s\n", filename, mode);
		FILE *fl = popen(filename, mode);
		int t = pclose(fl);
		if(WIFEXITED(t)) {
		    printf("exit status:%d\n", WEXITSTATUS(t)); 
		}
		return 0;
	} 

当我们编译程序运行时:
	
	$ gcc -o p popentest.c
	$ ./p "sh test.sh" r 

当我们上边这种方式运行的时候，很可能程序会立即返回并不会等待test.sh文件执行结束。在我自
己的机器上(Ubuntu 12.04.3 LTS),输出:  

	exit status:141

141说明子进程接受了SIGPIPE信号，那么为什么会收到SIGPIPE呢?  

首先，我们以读的方式打开管道，则返回的fl指针连接到子进程的标准输出；其次，我们知道，父进程
在fork了子进程后，它们的运行次序是不确定的，在上边的例子中，popen函数返回后，立即执行pclose
，我们知道在pclose会关闭fl指针关联的描述符。很可能此时子进程中执行的shell命令echo，此时的
描述符已被关闭，所以内核会发送SIGPIPE信号然后子进程退出。所以此时在调用waitpid则不会阻塞而立即返回。  

-EOF-


