---
title: map,set与hash_map,hash_set对比
layout: post
categories: [linux-c-cpp]
tags: [stl]
description: .
---
 

我们知道,c++的stl给我们提供了两个非常有用的数据结构: map, set，他们在查询指定元素方面提供了不错的性能;另外在标准之外，还有两个同样在查询元素方面有不错表现的数据结构: hash_map, hash_set;那么这两者之间有什么区别呢？在某一场景下，选择使用那种才是最合适的呢? 下边我们来探讨一下。

###map, set  

map与set都是基于RB-Tree实现的,所以在提供较好查询性能的同时，它们也能够按照key值的大小排列元素，所以map与set内的元素都是有序的，不允许插入key值相同的元素。  

###hash_map, hash_set

hash_map与hash_set是基于hashtable实现的(以开链法解决冲突)，鉴于hashtable的特性，不保证元素的有序性，另外也不能插入key值相同的元素  

***对于set来说，key就是value***

下边我们看一个示例程序,观察下hash_map和map的查询性能对比  

	#include <sys/time.h>
	#include <map>
	#include <ext/hash_map>
	#include <stdio.h>
	
	const static int ELE_NUM = 10000;
	
	using namespace std;
	using namespace __gnu_cxx;
	
	long get_curr_time() {
	    struct timeval tv;
	    gettimeofday(&tv, NULL);
	    return tv.tv_sec * 1000 * 1000 + tv.tv_usce;
	}
	
	int main() {
	    hash_map<int, int> _hashmap;
	    map<int, int> _map;
	
	    for(int i=0; i<ELE_NUM; ++i) {
	        _hashmap[i] = i;
	        _map.insert(make_pair(i, i));
	    }
	
	    long being = get_curr_time();
	    _map.find(1);
	    _map.find(100);
	    _map.find(8888);
	    _map.find(9999);
	    _map.find(2654);
	    long end = get_curr_time();
	    long map_cost = end - begin;
	
	    being = get_curr_time();
	    _hashmap.find(1);
	    _hashmap.find(100);
	    _hashmap.find(8888);
	    _hashmap.find(9999);
	    _hashmap.find(2654);
	    end = get_curr_time();
	    long hashmap_cost = end - begin;
	
	    printf("map cost:%ld, hashmap cost:%ld \n", map_cost, hashmap_cost);
	
	    return 0;
	}  

通过运行上边这段程序我们可以发现, 通过指定不同的ELE_NUM, hash_map的查询效率是要优于map的; 总体来说大概有3~6倍的差异. 

-EOF-
