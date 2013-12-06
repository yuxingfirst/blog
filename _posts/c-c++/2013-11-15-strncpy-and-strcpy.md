---
title: strncpy与strcpy
layout: post
category: [c&c++]
tags: [c]
--- 

C语言使用空字符来结束字符串.  

printf()和 strcpy()都把这一点作为默认的前提条件。此外,编译器也会在字符串常量的末尾自动加上空字符。不过strncpy()却打破了这个规则。strncpy()使用时可以按如下这种方式调用：  

    strncpy(dest, src, len);  

从src复制最大长度为len的字符到dest中。当 src 的长度大于 len 时,dest 不会以空字符结束。反过来,如果 src 比len 短,就会使用空字符补足剩余的长度。  

因此, 如果使用strncpy()拷贝字符串后,正巧src又比len长，那么则就会导致dest没有结尾的null字符, 之后再使用 printf()、sprintf()、strcpy()等处理 dest,由于其末尾可能没有空字符,进而可能会发生处理越界,以至于破坏大片内存区域的数据。因此,strncpy()是危险的。

> 如果使用 strncpy(),请注意它可能会产生没有空字符结尾的字符串。

所以在我们使用strncpy的时候, 一定要注意在字符串dest的末尾加上null('\0')。

下边总结下几个常用字符串的处理函数: 

    //将src所指的字符串拷贝到dest处，以('\0')作为结束符,需要确保dest有足够的空间
    char *strcpy(char *dest, const char *src);

    //将src所指的字符串拷贝n个字符到dest处,注意当src大于n时，dest不会以'\0'结束
    char *strncpy(char *dest, const char *src, size_t n);

    //计算字符串s的长度,不包括结尾的'\0'
    size_t strlen(const char *s);  

-EOF-
