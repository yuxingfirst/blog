---
title: 标准库printf函数族介绍
layout: post
categories: [linux-c]
tags: [linux-c]
description: the functions in the printf() family.
---  

	#include <stdio.h>
	
	int printf(const char *format, ...);
	int fprintf(FILE *stream, const char *format, ...);
	int sprintf(char *str, const char *format, ...);
	int snprintf(char *str, size_t size, const char *format, ...);  

	#include <stdarg.h>
	
	int vprintf(const char *format, va_list ap);
	int vfprintf(FILE *stream, const char *format, va_list ap);
	int vsprintf(char *str, const char *format, va_list ap);
	int vsnprintf(char *str, size_t size, const char *format, va_list ap);  

1. printf和vprintf写到标准输出流(stdout)。
2. fprintf和vfprintf写到指定的输出流。  
3. sprintf、vsprintf、snprintf、vsnprintf写到指定的缓冲区str。  

执行成功时，这些函数返回打印出的字符的数目(不包括结尾的'\0')。（**对于snprintf和vsnprintf，最多写size-1个字符到str中，并自动追加'\0'到str**）；输出发生错误时返回负数。  