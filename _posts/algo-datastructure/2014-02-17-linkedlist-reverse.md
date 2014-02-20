---
title: 单链表逆置
layout: post
categories: [数据结构]
tags: [链表]
description:  .  
--- 

关于单链表的逆置，整体来说，可以利用三个指针(包含一个临时指针)来完成单链表的逆置。需要注意的是边界条件的判定。  

	//
	struct ListNode {
		int val;
		ListNode *next;
		ListNode(int x) : val(x), next(NULL) {}
	};

	void list_reverse(ListNode *&head)
	{
		if (!head) {
			return;
		}
		if (!head->next) {
			return;
		}
		ListNode *p = head;
		ListNode *q = p->next;
		p->next = NULL;
		while(q) {
			ListNode *m = q;
			q = q->next;
			m->next = p;
			p = m;
		}
		head = p;
	}

-EOF-
