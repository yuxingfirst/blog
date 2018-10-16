---
title: twemproxy源码分析part3-mbuf设计
layout: post
categories: [技术研究]
tags: [redis, twemproxy]
description: .
---

对于网络server来说，主要的职责就是进行网络I/O, 所以buf的设计对于server性能来说是至关重要的。
通过对twemproxy源码的分析，我将来介绍一下twemproxy中buf的设计。  

buf，其实就是一段内存空间，twemproxy中的buf都有固定的大小chunk_size，这个大小在程序初始
化时就确定了(支持配置)，所有[空闲的buf]都通过一个队列进行管理，当有获取buf的请求时，  
buf管理程序就先检查空闲buf管理队列是否为空，如果不为空，直接从队列中返回一个buf;如果队列  
为空,那么就先申请chunk_size大小的内存，然后将buf初始化之后返回。  

> 注意我上边说的空闲buf管理队列，其中的buf都是没有使用的buf，使用中buf由msg管理，这个后边再介绍。  

对应的文件，参看 [nc_mbuf.h/nc_mbuf.c](https://github.com/twitter/twemproxy/tree/master/src)。  

首先我们来看mbuf的定义(nc_mbuf.h):  

	struct mbuf {
		uint32_t           magic;   /* mbuf magic (const) */
		STAILQ_ENTRY(mbuf) next;    /* next mbuf */
		uint8_t            *pos;    /* read marker */
		uint8_t            *last;   /* write marker */
		uint8_t            *start;  /* start of buffer (const) */
		uint8_t            *end;    /* end of buffer (const) */
	};  

	STAILQ_HEAD(mhdr, mbuf);  

mbuf定义中，magic, start, end为常量，这些属性在创建mbuf时就确定了。pos为数据读取开始索引，last为数据写入开始索引。  

此外，还定义了一个mhdr队列（STAILQ）, 队列中的元素为mbuf。我们看mbuf中的next， 其指向队列中的下一个元素。  

下边，我们再来看几个变量定义：  

	static uint32_t nfree_mbufq;   /* # free mbuf 队列中mbuf的数量 */
	static struct mhdr free_mbufq; /* free mbuf q 空闲mbuf队列  */

	static size_t mbuf_chunk_size; /* mbuf chunk size - header + data (const) */
	static size_t mbuf_offset;     /* mbuf offset in chunk (const) */

下边的是获取一个mbuf结构的底层函数  

	static struct mbuf *
	_mbuf_get(void)
	{
		struct mbuf *mbuf;
		uint8_t *buf;

		if (!STAILQ_EMPTY(&free_mbufq)) {	/* mbuf队列不为空 */
		    ASSERT(nfree_mbufq > 0);

		    mbuf = STAILQ_FIRST(&free_mbufq);
		    nfree_mbufq--;
		    STAILQ_REMOVE_HEAD(&free_mbufq, next);

		    ASSERT(mbuf->magic == MBUF_MAGIC);
		    goto done;
		}

		buf = nc_alloc(mbuf_chunk_size);
		if (buf == NULL) {
		    return NULL;
		}

		/*
		 * mbuf header is at the tail end of the mbuf. This enables us to catch
		 * buffer overrun early by asserting on the magic value during get or
		 * put operations
		 *
		 *   <------------- mbuf_chunk_size ------------->
		 *   +-------------------------------------------+
		 *   |       mbuf data          |  mbuf header   |
		 *   |     (mbuf_offset)        | (struct mbuf)  |
		 *   +-------------------------------------------+
		 *   ^           ^        ^     ^^
		 *   |           |        |     ||
		 *   \           |        |     |\
		 *   mbuf->start \        |     | mbuf->end (one byte past valid bound)
		 *                mbuf->pos     \
		 *                        \      mbuf
		 *                        mbuf->last (one byte past valid byte)
		 *
		 */
		mbuf = (struct mbuf *)(buf + mbuf_offset);
		mbuf->magic = MBUF_MAGIC;

	done:
		/* 设置这个mbuf的后继元素为NULL(可能是新分配的或者是从当前的mbuf队列中取出来的) */
		STAILQ_NEXT(mbuf, next) = NULL;	
		return mbuf;
	}

我们看上边mbuf的结构  

> mbuf_chunk_size 为整个mbuf的大小。 
>  
> mbuf data	为mbuf中数据段。  
> 
> mbuf_offset	数据段的大小，这个跟变量mbuf_offset是一样的，固定大小。 
>  
> mbuf header 为mbuf本身占据的内存空间  

我们看下外部获取mbuf的接口  

	struct mbuf *
	mbuf_get(void)
	{
		struct mbuf *mbuf;
		uint8_t *buf;

		mbuf = _mbuf_get();
		if (mbuf == NULL) {
		    return NULL;
		}

		buf = (uint8_t *)mbuf - mbuf_offset;
		mbuf->start = buf;
		mbuf->end = buf + mbuf_offset;

		ASSERT(mbuf->end - mbuf->start == (int)mbuf_offset);
		ASSERT(mbuf->start < mbuf->end);

		mbuf->pos = mbuf->start;
		mbuf->last = mbuf->start;

		log_debug(LOG_VVERB, "get mbuf %p", mbuf);

		return mbuf;
	}  

我觉得twemproxy中mbuf的设计是十分巧妙的，我们来看，通过调用  

	mbuf = _mbuf_get();

获取一段buf后，实际上， mbuf这个指针，并不是指向这段内存的开头，这段内存的开始位置为：  

	mbuf - mbuf_offset == mbuf.start  

这段内存既作为数据的存储空间，又维护着自身的一些信息。  

下边列出mbuf的一些操作的实现。  
	
	/* reset mbuf中的内存区 */
	void
	mbuf_rewind(struct mbuf *mbuf)
	{
		mbuf->pos = mbuf->start;
		mbuf->last = mbuf->start;
	}  

	/* 返回mbuf中已经存储的数据 */
	uint32_t
	mbuf_length(struct mbuf *mbuf)
	{
		ASSERT(mbuf->last >= mbuf->pos);

		return (uint32_t)(mbuf->last - mbuf->pos);	/* mbuf中已经存储的数据 */
	}  
	
	/*将pos指向的内存区域拷贝n字解到mbuf中*/
	void
	mbuf_copy(struct mbuf *mbuf, uint8_t *pos, size_t n)
	{
		if (n == 0) {
		    return;
		}

		/* mbuf has space for n bytes 检查mbuf中还有n字节的可用空间 */
		ASSERT(!mbuf_full(mbuf) && n <= mbuf_size(mbuf));

		/* no overlapping copy  检查pos指向的内存区没有跟mbuf重叠 */
		ASSERT(pos < mbuf->start || pos >= mbuf->end);

		nc_memcpy(mbuf->last, pos, n);
		mbuf->last += n;
	}  

-EOF-





























