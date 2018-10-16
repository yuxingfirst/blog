---
title: twemproxy源码分析part2-freeBSD queue
layout: post
categories: [技术研究]
tags: [redis, twemproxy]
description: .
---

这篇中我将来介绍twemproxy中队列的应用。  

首先，我们先定位到nc_queue.h这个文件， 这个是FreeBSD提供的一些数据结构，这里边主要是队列的一些实现。在twemproxy中用到的主要是两种队列，一种是双向队列，一种的单向队列，但是可以直接访问队列末尾元素。我们看这两种队列的定义：  

	/*
	 * Singly-linked Tail queue declarations.
	 * 1：以O(n)的时间复杂度移除任意元素。
	 * 2：新元素可以插入到指定元素的后边，也可以插入到队列头或者队列尾。
	 * 3：为了最佳效率，需要以提供的指定宏移除队列头部的元素。
	 * 4：只能正向遍历。
	 * 5：是一种LIFO(后进先出)队列，适合很少(或没有)移除操作的大数据集应用。
	 */
	#define STAILQ_HEAD(name, type)                                         \
	struct name {                                                           \
	    struct type *stqh_first; /* first element 指向队列头 */                        \
	    struct type **stqh_last; /* addr of last next element 指向队列尾 */            \
	}

	/*
	 * Tail queue declarations.
	 * 1：双向队列。
	 * 2：移除元素无需进行遍历操作。
	 * 3：新元素可以插入到指定元素的任意位置(前/后)，队列头，队列尾。
	 */
	#define TAILQ_HEAD(name, type)                                          \
	struct name {                                                           \
	    struct type *tqh_first; /* first element */                         \
	    struct type **tqh_last; /* addr of last next element */             \
	    TRACEBUF                                                            \
	}  

	/* 这个宏用来定义一个匿名结构体，它包含指向下一个元素的指针 */
	#define STAILQ_ENTRY(type)                                              \
	struct {                                                                \
		struct type *stqe_next;    /* next element */                       \
	}  

	/** 
	 * 将elm插入到tqelm的后边 
	 * 我们来分析,如下
	 * 		tqelm --> tqelm_next --> tqelm_next_next
	*/
	#define STAILQ_INSERT_AFTER(head, tqelm, elm, field) do {               \
		/* 如果tqelm为队列最后一个元素,直接将elm放到队列尾 */
		if ((STAILQ_NEXT((elm), field) = STAILQ_NEXT((tqelm), field)) == NULL)\
		    (head)->stqh_last = &STAILQ_NEXT((elm), field);                 \
		/* 插入到tqelm后边 */
		STAILQ_NEXT((tqelm), field) = (elm);                                \
	} while (0)
	
	/* 插入elm到队列头 */
	#define STAILQ_INSERT_HEAD(head, elm, field) do {                       \
		if ((STAILQ_NEXT((elm), field) = STAILQ_FIRST((head))) == NULL)     \
		    (head)->stqh_last = &STAILQ_NEXT((elm), field);                 \
		STAILQ_FIRST((head)) = (elm);                                       \
	} while (0)
	
	/* 插入elm到队列尾 */
	#define STAILQ_INSERT_TAIL(head, elm, field) do {                       \
		STAILQ_NEXT((elm), field) = NULL;                                   \
		*(head)->stqh_last = (elm);                                         \
		(head)->stqh_last = &STAILQ_NEXT((elm), field);                     \
	} while (0)  
	
	/* 取elm的下一个元素*/
	#define STAILQ_NEXT(elm, field)    ((elm)->field.stqe_next)  
	
	/* 取队列最后的一个元素,如果队列为空则返回NULL */
	#define STAILQ_LAST(head, type, field)                                  \
    (STAILQ_EMPTY((head)) ?                                             \
        NULL :                                                          \
            ((struct type *)(void *)                                    \
        ((char *)((head)->stqh_last) - __offsetof(struct type, field))))  
	
	/* 遍历队列,遍历过程中不可进行元素的删除操作 */
	#define STAILQ_FOREACH(var, head, field)                                \
    for ((var) = STAILQ_FIRST((head));                                  \
         (var);                                                         \
         (var) = STAILQ_NEXT((var), field))

	/* 遍历队列,遍历过程中可进行元素的删除操作 */
	#define STAILQ_FOREACH_SAFE(var, head, field, tvar)                     \
    for ((var) = STAILQ_FIRST((head));                                  \
        (var) && ((tvar) = STAILQ_NEXT((var), field), 1);               \
        (var) = (tvar))  

具体相关的使用示例，请点击: [queue example](https://github.com/yuxingfirst/snipper/tree/master/freeBSD_queue)

		

