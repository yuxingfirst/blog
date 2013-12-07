---
title: APUE读书笔记-Chapter7-进程环境
layout: post
categories: [读书笔记]
tags: [c, unix]
description: 进程环境.
---

###Section7.2-main函数
C程序总是从main函数开始执行。main函数的原型是  

    int main(int argc, char *argv[]);  

其中，argc是命令行参数的数目，argv是指向参数的各个指针所构成的数组。当内核执行C程序时(
使用一个exec函数)，在调用main前先调用一个特殊的启动例程。可执行程序文件将此启动例程指定
为程序的起始地址---这是由连接编辑器设置的。而连接编辑器则由C编译器(通常是cc)调用。启动
例程从内核取得命令行参数和环境变量值，然后为按上述方式调用main函数做好安排。  

###Section7.3-进程终止
有8种方式使进程终止(termination)，其中5种为正常终止，它们是：  

1. 从main返回  
2. 调用exit。  
3. 调用_exit或_Exit。  
4. 最后一个线程从其启动例程返回。  

异常终止有3种方式，它们是：  

1. 调用abort。  
2. 接到一个信号并终止。  
3. 最后一个线程对取消请求做出响应。  

######7.3.1-exit函数
有三个函数用于正常终止一个程序:_exit和_Exit立即进入内核，exit则先执行一些清理处理(包括
调用执行各终止处理程序，关闭所有标准I/O流等)，然后进入内核。  

	#include <stdlib.h>
	
	void exit(int status);

	void _Exit(int status);

	#include <unistd.h>
	
	void _exit(int status);  

由于历史原因，exit函数总是执行一个标准I/O库的清理关闭操作：为所有打开流调用fclose函数(
会造成所有缓冲的输出数据都被冲洗)。  
三个exit函数都带一个整型参数，称之为终止状态(或退出状态，exit status)。大多数UNIX SHE
LL 都提供检查进程终止状态的方法。如果(a)若调用这些函数时不带终止状态，或(b)main执行了一
个无返回值的return语句，或(c)main没有声明返回类型为整型，则该进程的终止状态是未定义的。
当时，若main的返回类型是整型，并且main执行到最后一条语句时返回(隐式返回)，那么该进程的
终止状态是0.  

main函数返回一整型值与用该值调用exit是等价的。即：  

	exit(0);
	==
	return 0;  

######7.3.1-atexit函数
按照ISO C的规定，一个进程可以登记多达32个函数，这些函数将有exit自动调用。我们称这些函数
为终止处理程序，并调用atexit函数来登记这些函数。

	#include <stdlib.h>
	int atexit(void (*func)(void));  

atexit的参数是一个函数地址，当调用此函数时无需向他传送任何参数，也不期望它返回一个值。
exit调用这些函数的顺序与它们等级时候的顺序相反。同一函数如果登记多次，则也会被调用多次。

图7-1显示了一个C程序是如何启动的，以及它可以终止的各种方式。  

![img1](https://raw.github.com/yuxingfirst/yuxingfirst.github.io/master/_images/read-notes/read-notes-apue-c7-img1.jpg)
图7-1  

注意，内核使程序执行的唯一方法是调用一个exec函数。进程自愿终止的唯一方法是显示或隐士地(通
过调用exit)调用_exit或_Exit。进程也可非自愿地由一个信号使其终止。  

###Section7.4-命令行参数
	
	$ ./exceprog arg1 arg2 agr3  

当我们这样执行一个程序的时候，main函数中我们可以这样访问:

	argv[0]: ./exceprog
	argv[1]: arg1
	argv[2]: arg2
	argv[3]: arg3  

###Section7.6-C程序的存储空间布局

1. 正文段  
2. 初始化数据段  
3. 非初始化数据段  
4. 栈  
5. 堆  










