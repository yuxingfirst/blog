---
title: 字节序
layout: post
categories: [网络编程]
tags: [network]
description: Byte Order
---

字节序是一种处理器架构的特性，在表示大的数据类型时用于指示字节是如何排序的，比如整型数(integer)。  
可以看一下一个32位整数的字节是如何排序的：  

![img1](https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/byte-order-1.gif)  

如上图所示，展示了两种字节排序的方式：  

1. 大端字节序(big-endian)  
2. 小端字节序(little-endian)  

如果处理器架构支持大端字节序，那么高字节地址就出现在最低有效字节；小端字节序是相反的：最低有效字节包含
最低字节地址。  

如图所示，在不关心字节顺序的情况下，最高有效字节总是出现在左边，而最低有效字节总是出现在右边。举例来说，
如果我们将0x04030201赋值给一个32位的整数，那么最高有效位将包含那4，最低有效位将包含1。此时，如果我们将一个
字符指(cp)针转换为一个整数的地址，在不同的字节序的条件下，我们会看到两者的差异。  

在小端字节序的处理器中，cp[0]将指向最低有效字节处的值1;cp[3]指向最高有效字节处的值4。但是如果在大端字节序的处理器，
cp[0]包含4，引用了最高有效字节；cp[3]包含1，引用了最低有效字节。   

下图是一些平台支持的字节序列：   

![img2](https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/byte-order-platforms.png)

我自己机器上边的测试结果:
	
	#include <stdio.h>

	int main(void) {
		int a = 0x04030201;
		char *cp = (char*)&a;
		return 0;
	}  

要查看自己的机器是哪种字节序，我们可以借助gdb查看上边这段程序中变量a的二进制内容：  

	$ gdb -g -o byteorder byteorder.c
	$ gdb byteorder
	(gdb) x/t cp
	(gdb) 000001000000001100000010000000001  

结合上边的介绍和上述输出结果，我们可以看到我机器是大端字节序。因为cp[0]和cp[3] (int为4个字节)的内容分别是:
	
	cp[0]=00000100=4  
  	cp[3]=00000001=1

通常，网络协议会指定自己的字节序，比如TCP/IP协议一般按大端字节序。这种情求下，就有可能跟cpu
使用的字节序不一样。所以，在网络编程中，系统为我们提供了一组api用于在网络字节序和主机字节序之间相互转换:  

	#include <arpa/inet.h>
	uint32_t htonl(uint32_t hostint32);
					//Returns:32-bit integer in network byte order

	uint16_t htons(uint16_t hostint16);
					//Returns:16-bit integer in network byte order

	uint32_t ntohl(uint32_t netint32);
					//Returns:32-bit integer in host byte order

	uint32_t ntohs(uint16_t netinit16);
					//Returns:16-bit integer in host byte order

在上边这些函数中，h代表主机(host)字节序，n代表网络(network)字节序；l代表长(long)整数。  
s代表短(short)整型。  

关于gdb如何查看内存内容参考这篇文章:[gdb查看内存](http://coderworm.com//c/2013/09/28/gdb.html)

-EOF-



