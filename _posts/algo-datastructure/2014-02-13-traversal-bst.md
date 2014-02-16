---
title: 二叉树的遍历(前序，中序，后序，层序)
layout: post
categories: [数据结构]
tags: [二叉树, Interview]
description:  .
---  

二叉树的定义:

	struct BSTreeNode
	{
		int m_nValue;
		struct BSTreeNode *pLeft;
		struct BSTreeNode *pRight;
	}

###### 前序遍历二叉树

        7
	   / \
      5   10
     / \  / \
     1  6 8  12

输出： 7 -> 5 -> 1 -> 6 -> 10 -> 8 -> 12  

	void preOrder(BSTreeNode *pNode) 
	{
		if(!pNode) {
			return;
		}
		cout << pNode->m_nValue << ",";
		preOrder(pNode->pLeft);
		preOrder(pNode->pRight);
	}



######中序遍历

        7
	   / \
      5   10
     / \  / \
     1  6 8  12

输出： 7 -> 5 -> 1 -> 6 -> 10 -> 8 -> 12  

	void preOrder(BSTreeNode *pNode) 
	{
		if(!pNode) {
			return;
		}
		cout << pNode->m_nValue << ",";
		preOrder(pNode->pLeft);
		preOrder(pNode->pRight);
	}
  
######后序遍历

        7
	   / \
      5   10
     / \  / \
     1  6 8  12

输出： 1 -> 6 -> 5 -> 8 -> 12 -> 10 -> 7

	void pastOrder(BSTreeNode *pNode) 
	{
		if(!pNode) {
			return;
		}
		pastOrder(pNode->pLeft);
		pastOrder(pNode->pRight);
		cout << pNode->m_nValue << ",";
	}

###### 从上往下遍历二叉树(层序遍历，广度优先遍历)

        7
	   / \
      5   10
     / \  / \
     1  6 8  12

输出： 7 -> 5 -> 10 -> 1 -> 6 -> 8 -> 12

**分析**：
要从上往下遍历二叉树，同一层的节点按照从左至右的顺序，当访问根节点7时，为了下次访问其子节点，需要将子节点放入一个容器中；并且，当访问根节点的子节点时，为了访问子节点的子节点，同样需要将其放入一个容器中，并且先放入的要先访问，所以可以用队列来保存。

	void SequenceTraversalBST(BSTreeNode *pRoot) {
		if(!pRoot) {
			return;
		}
		deque<BSTreeNode*> tmpDeq;
		tmpDeq.push_front(pRoot);
		while(!tmpDeq.empty()) {
			BSTreeNode *pNode = tmpDeq.back();
			cout << pNode->m_nValue << ",";
			if(pNode->pLeft) {
				tmpDeq.push_front(pNode->pLeft);
			}
			if(pNode->pRight) {
				tmpDeq.push_front(pNode->pRight);
			}
			tmpDeq.pop_back();
		}
		return;
	}
	
	   操作			        队列(对头->队尾)
	push_deq: 7     ->      7
	print: 7
	push_deq: 5 10  ->      10 5 7
	pop_deq         ->      10 5
	print: 5
	push_deq: 1 6   ->      6 1 10 5
	pop_deq         ->      6 1 10
	print: 10
	push_deq:8 12   ->      12 8 6 1 10
	pop_deq         ->      12 8 6 1
	print 1
	pop_deq         ->      12 8 6
	print 6
	pop_deq         ->      12 8
	print 8
	pop_deq         ->      12
	print 12
	pop_deq

-EOF-
