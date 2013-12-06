---
title: Gprof使用介绍
layout: post
category: [程序工具]
tags: [c++, gprof]
--- 

根据wikipedia的介绍，Gprof是一个Unix下的性能分析工具，当然也适用于Linux。使用Gprof, 我们可以分析出来我们的程序在所花费的时间以及在程序的执行过程中，函数之间的调用关系。使用Gprof在分析程序的CPU占用时间方面是非常有效的，通过导出的调用图，可以很直观的看出来我们的程序在什么地方花费了比较多的CPU时间，藉此我们便能高效的优化我们的程序。下边我们看看如何使用Gprof。  

为了得到Gprof的分析结果图，有如下几个步骤：  
1. 增加编译参数 -pg 重新编译我们的c/c++参数。  
2. 启动程序并且在程序运行一段事件后，停止程序。这样在程序的执行路径下会产生一个gmon.out的文件。  
3. 以生成的gmon.out作为输入文件，执行gprof命令(一般linux都自带了)，用来分析profile data。  

通常直接使用gprof分析profile data得出的结果是不直观的，好在有这么两个工具： 
 
> gprof2dot.py  
> dot  

我们可以使用这两个工具生成一个png图片，这个图片中就包含了我们程序中函数的调用关系结构图以及每个函数调用所占用的CPU时间。

参考地址：  
1. [https://sourceware.org/binutils/docs/gprof/](https://sourceware.org/binutils/docs/gprof/)  
2. [https://code.google.com/p/jrfonseca/wiki/Gprof2Dot](https://code.google.com/p/jrfonseca/wiki/Gprof2Dot)  

-EOF-