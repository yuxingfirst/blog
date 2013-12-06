---
title: 解析redis中跳表(skiplist)的运用
layout: post
category: [redis]
tags: [c, redis]
--- 

了解过Redis的都知道，Redis有一个非常有用的数据结构：SortSet，基于它，我们可以很轻松的实现一个Top N的应用。那么，这个SortSet底层到怎么样实现的？怎么样实现才能既保证有序并提供插入，查询的最优时间复杂度呢？这里，便就是我将要给大家介绍的***跳表***

跳表的具体定义，请参考[wikipedia跳表定义](http://zh.wikipedia.org/zh/%E8%B7%B3%E8%B7%83%E5%88%97%E8%A1%A8)，跳表也是链表的一种，只不过它在链表的基础上增加了***跳跃***功能，正是这个跳跃的功能，使得在查找元素时，跳表能够提供O(log n)的时间复杂度。（通常，像红黑树等这样的数据结构查找的时间复杂度也是O(log n)，但是**正确**实现一颗红黑树是比正确实现一个跳表是要复杂很多很多的）

> **跳跃**

那么，这个**跳跃**的功能是怎么实现的呢？为什么能够提供跟查找树一样的O(log n)的时间复杂度呢？下边我将借助Redis中的代码来分析这些原理。（redis的完整代码，参看 [redis](https://github.com/antirez/redis)）

redis代码中，跳表主要实现在 [t_zset.c](https://github.com/antirez/redis/blob/unstable/src/t_zset.c)，跳表节点的定义，则在[redis.h](https://github.com/antirez/redis/blob/unstable/src/redis.h)中，我们首先来看跳表节点的定义：  

    typedef struct zskiplistNode {
        robj *obj;
        double score;
        struct zskiplistNode *backward;
        struct zskiplistLevel {
            struct zskiplistNode *forward;
            unsigned int span;
        } level[];
    } zskiplistNode;  

暂时先不关心robj *obj和unsigned int span这两个属性，robj是redis内部对象的定义， span是redis内部在计算节点的排名使用的。  

在定义普通链表节点的时候（双向链表）

    struct ListNode 
    {
        void *data;
        ListNode *prev;
        ListNode *next;
    };  
prev为前向指针，next为后向指针。  
那么我们再来看redis中跳表节点的前向节点指针的定义，

    struct zskiplistNode *backward;  

backward指针表示其也是一个双向链表，指向某个节点前向节点，这点跟普通链表的prev指针的含义是一样的。  
再看后向节点指针的定义：  

    struct zskiplistLevel {
            struct zskiplistNode *forward;
            unsigned int span;
    } level[];  
很奇怪是不是？后向节点指针变成了一个数组，这正是跳表可以实现**跳跃**的实质，这个数组中的forward元素不仅可以指向这个节点的直接后继元素，也可以指向后继元素的后继元素，或者是后继后继后继元素，依此类推。这也就是说，从当前的这个节点，可以以O(1)的时间复杂度跨过其直接后继节点查找其后的某个节点，这样就实现了跳跃。（普通链表需要查找某个节点的话，必须要进行遍历操作，这对于有查询性能要求的应用场景来说，是不能接受的！）

> **O(log n)**  

现在我们再来看看跳表为什么能提供O(log n)的时间复杂度。  
一般，我们会在插入元素的时候，就保持跳表的有序排列。请看下图:  
 ![enter image description here][1]

可以看到，正式这样一种结构，我们每次在进行查询操作的时候，都会从最高一层开始找，由于链表是有序的，所以期望的时间复杂度是O(log n)，当然最坏情况下的时间复杂度是o(n)，不过这种情况应该不太会遇到。  

需要指出的是，实际的存储结构，并不是像上图这样有这么多节点，跟普通链表相比，跳表额外的存储空间只有那个前向节点数组花费的，当然，这是一种以空间换取时间的策略。与它提供的特性相比，这点额外的空间我们是可以接受的。

> **redis跳表实现解析**  

首先我们看下redis实现中几个我们可以自己拿出来用的api：  

    /** 创建一个skiplist */ 
    zskiplist *zslCreate(void)
    
    /** 创建一个skiplist节点 */
    zskiplistNode *zslCreateNode(int level, double score, robj *obj)；

    /** 释放一个节点的内存 */
    void zslFreeNode(zskiplistNode *node);
    
    /** 释放整个skiplist */
    void zslFree(zskiplist *zsl);
	
	/** 插入一个节点 */
    zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj)

	/** 删除一个节点，根据score删除 */
	int zslDelete(zskiplist *zsl, double score, robj *obj)；
	
	/** 这个函数是查询某个节点对应的排名,其实就是在跳表中位置 */
	unsigned long zslGetRank(zskiplist *zsl, double score, robj *o)；  

上边这些使我们可以从redis源代码中抠出来自己用的代码(当然还需要在封装封装)，对于跳表来说，重点需要关注的就是插入，查找，删除这几个操作。

这几个函数包含了最基本的几种操作，创建、插入、删除、查询；首先，我们看redis中是如何实现跳表的创建的：  

    typedef struct zskiplist {
    	struct zskiplistNode *header, *tail;
    	unsigned long length;
    	int level;
    } zskiplist;

    zskiplist *zslCreate(void) {
        int j;
        zskiplist *zsl;

        zsl = zmalloc(sizeof(*zsl));
        zsl->level = 1;
        zsl->length = 0;
        zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
        for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
            zsl->header->level[j].forward = NULL;
            zsl->header->level[j].span = 0;
        }
        zsl->header->backward = NULL;
        zsl->tail = NULL;
        return zsl;
    }  

首先是跳表结构的定义  

    struct zskiplistNode *header, *tail  

头尾指针，表名是双向链表。  

    unsigned long length;  

length是表示链表中节点的个数。  
  
    int level;  

表示跳表的层数。  
然后我们在看zslCreate函数的定义，其中最关键的就是里边的那个循环，  

    ZSKIPLIST_MAXLEVEL  

宏定义，用于表示这个跳表最多有多少层。在循环内，将指向后边节点的forward指针置为NULL。
zslCreate调用完成后，其实就是创建了跳表的头结点（我们知道所有的链表实现中通常也是需要一个头结点的）。  

然后我们再来看如何给跳表插入一个元素，插入元素代码请看[t_zset.c,第107行zslInsert函数](https://github.com/yuxingfirst/redis/blob/unstable/src/t_zset.c), 这里不贴出完整的代码，我只取其中比较重要的部分分析。  

插入一个元素到跳表中，首先需要确定的是插入元素的位置，插入元素的时候需要保证跳表元素的排列顺序是升序排列；然后确定待插入节点的层数；最后是将节点各层指向正确的后续节点，然后插入元素。  

首先请看下述代码段：  

    
	zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
	x = zsl->header;
	for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }  

这段代码主要的作用就是查找节点所要插入的位置。首先请看下边这幅图：  
![跳表图][2]  

通常，在跳表中查找元素的时候，不像在链表中查找元素那样需要遍历，而是会从头结点(头结点的下一节点才是元素节点)的最顶层开始，即上述代码的for循环，如果level数组的forward指针指向的节点的score值大于要插入的score，那么就下降一层；否则，就把x前进一个节点，指向到下一个节点，继续比较，即上述代码中while循环所做的工作。最后，当结束for循环时，update数组中保存的节点的forward就是将要插入的节点的level数组中的forward需要的值，并且待插入的节点一定是位于**update[0]这个指针所指节点的后边**。  

当新节点需要插入的位置找到后，就需要确定新节点的层数：
    
    level = zslRandomLevel();  

我们知道，跳跃列表是对有序的链表增加上附加的前进链接，增加是以随机化的方式进行的，所以节点层数的确定当然也是随机的，以上这行代码便是确定节点的层数。  

因为在前边查找的时候，是从当前跳表的最高一层开始查找的，如果新节点的层数大于跳表当前的层数，则需要更新跳表的层数并扩展之前的update数组，代码如下：  
	
	if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }  

这里需要指出的是为什么新加的层节点直接用zsl->header赋值呢? 原因是这样的，因为新加入的这层与zsl->header直接肯定是没有其他节点的层的；而下边我们在给初始化新插入节点的level数组的时候，是把update的每一个元素当作其前一个跳跃节点的。  

	 x = zslCreateNode(level,score,obj);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }  

综合这几段代码，我们便可以知道update这个数组的作用了；然后便是给backward赋值，以及修改zsl的tail指针:
  
	 x->backward = (update[0] == zsl->header) ? NULL : update[0];
    	if (x->level[0].forward)
        	x->level[0].forward->backward = x;
    	else
    	    zsl->tail = x;
    zsl->length++;  

到这里，基本跳表的创建以及插入元素都分析完毕了，下边，我们看看如何删除一个元素。删除元素的代码位于[t_zset.c 第185行zslDelete定义](https://github.com/yuxingfirst/redis/blob/unstable/src/t_zset.c)  

同插入节点类似，删除节点的时候，同样也是需要先找到需要删除节点的位置，代码如下：  

	zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0)))
            x = x->level[i].forward;
        update[i] = x;
    }  

这里，需要特别注意的是，到for循环结束时，update[0]处的指针所指向的是要删除的节点的前一个节点，到这里为止，要删除的节点是否存在并不知道。  

	x = x->level[0].forward;  

这里便是将节点往后前进一个位置，便到了我们要寻找的节点的位置，到这里为止，要删除的节点是否存在仍然不知道，下边便是做节点的比较：  

	if (x && score == x->score && equalStringObjects(x->obj,obj)) {
        zslDeleteNode(zsl, x, update);
        zslFreeNode(x);
        return 1;
    } else {
        return 0; /* not found */
    }  

只有当score值与节点元素的值全部相等时，才说明要删除的节点是存在的，否则就是不存在的。  

最后，节点元素的查找跟之前插入跟删除节点的查找过程是一样的，具体请看redis的实现中下边这个函数:   

[t_zset.c unsigned long zslGetRank(zskiplist *zsl, double score, robj *o)][4]    


[1]: https://raw.github.com/yuxingfirst/Blogs/master/images/redis-redis-skiplist-1.png
[2]: https://raw.github.com/yuxingfirst/Blogs/master/images/redis-redis-skiplist-2.png 
[3]: https://github.com/yuxingfirst/Blogs/blob/master/redis/%E6%B5%85%E6%9E%90redis%E4%B8%AD%E8%B7%B3%E8%A1%A8%E7%9A%84%E8%BF%90%E7%94%A8(%E4%B8%80).md
[4]: https://github.com/yuxingfirst/redis/blob/unstable/src/t_zset.c#zslDelete

-EOF-