---
title: 程序员面试题精选100-二叉树相关
layout: post
categories: [Program-Interview-Series]
tags: [面试，二叉树]
description:  二叉树相关面试题.
---  

###01. 二叉查找树转换为排序的双向链表
**题目**： 输入一棵二叉查找树，将该二叉查找树转换成一个排序的双向链表。要求不能创建任何新的结点，只调整指针的指向。 比如:

		10
	   /  \
      6    14
     / \   / \
    4   8 12 16 

转换成双向链表: 4=6=8=10=12=14=16。

**分析**：很多与树有关的题都可以用递归的思想来解决，这里也不例外。  
**思路一**：要运用递归来解决，我们可以先转换某个节点的左子树，将其转换为一个排好序的左子链表；然后再转换该节点的右子树，将其转换为一个排好序的右子链表；最后再将左子链表，父节点，右子链表链接起来。  
**思路二**：可以按照中序遍历这颗树。这样较小的节点先访问，每当遍历到一个节点时，假设在它前边访问的节点已经调整成了一个双向链表，只需要将这个节点链接到链表的尾部，这样，当所有节点都访问过后，整棵树也转换成了一个排好序的双向链表。  
**代码**：  

	struct BSTreeNode
	{
		int m_nValue;
		struct BSTreeNode *pLeft;
		struct BSTreeNode *pRight;
	}

	//思路一
	BSTreeNode* BSTreeSortedDoubleListConvert(struct BSTreeNode *pNode, bool isRight)
	{
		if(!pNode) {
			return NULL;
		}
		struct BSTreeNode *left = NULL;
		struct BSTreeNode *right = NULL;
		if(!pNode->pLeft) {
			left = BSTreeSortedDoubleListConvert(pNode->pLeft, false);
		}
		if(left) {
			left->pRight = pNode;
			pNode->pLeft = left;
		}
		if(!pNode->pRight) {
			right = BSTreeSortedDoubleListConvert(pNode->pRight, true);
		}
		if(right) {
			right->pLeft = pNode;
			pNode->pRight = right;
		}
		struct BSTreeNode *pTemp = pNode;
		if(!isRight) {
			while(pTemp->pRight) {
				pTemp = pTemp->pRight;
			}
		} else {
			while(pTemp->pLeft) {
				pTemp = pTemp->pLeft;
			}
		}
		return pTemp;
	}	
	
	//思路二
	

###02. 二叉树中和为某个给定值的所有路径
**题目**：输入一个整数和一棵二叉树。从树的根结点开始往下访问一直到叶结点所经过的所有结点形成一条路径。打印出和与输入整数相等的所有路径。  
**分析**： 这道题考查对树这种基本数据结构以及递归函数的理解。当访问到某一结点时，把该结点添加到路径上，并累加当前结点的值。如果当前结点为叶结点并且当前路径的和刚好等于输入的整数，则当前的路径符合要求，我们把它打印出来。如果当前结点不是叶结点，则继续访问它的子结点。当前结点访问结束后，递归函数将自动回到父结点。因此我们在函数退出之前要在路径上删除当前结点并减去当前结点的值，以确保返回父结点时路径刚好是根结点到父结点的路径。我们不难看出保存路径的数据结构实际上是一个栈结构，因为路径要与递归调用状态一致，而递归调用本质就是一个压栈和出栈的过程。  
**代码**：
	
	struct BSTreeNode
	{
		int m_nValue;
		struct BSTreeNode *pLeft;
		struct BSTreeNode *pRight;
	}

	void FindPath(struct BSTreeNode *pNode, deque<int> &pathDeq, int sum)
	{
		if(!pNode) {
			return;
		}
		pathDeq.push_back(pNode->m_nValue);
		if(!pNode->pLeft && !pNode->pRight) {			
			if(DeqSum(pathDeq) == sum) {
				DeqPrint(pathDeq);
			}
		}
		if(pNode->pLeft) {
			FindPath(pNode->pLeft, pathDeq, sum);
		}
		if(pNode->pRight) {
			FindPath(pNode->pLeft, pathDeq, sum);
		}
		pathDeq.pop_back();
	}

###03. 判断整数序列是不是某二叉树的后序遍历结果
**题目**：输入一个整数数组，判断该数组是不是某二元查找树的后序遍历的结果。如果是返回
true，否则返回false。例如输入5、7、6、9、11、10、8，由于这一整数序列是如下树的后序遍历结果：  
	
			8
		  /   \
         6    10 
        / \   / \
       5   7 9   11
因此返回true。如果输入:7、4、6、5，没有哪棵树的后序遍历的结果是这个序列，因此返回false。  
**分析**： 在后续遍历得到的序列中，最后一个元素为树的根结点。从头开始扫描这个序列，比根结点
小的元素都应该位于序列的左半部分；从第一个大于根结点开始到根结点前面的一个元素为止，所有元素都应该大于根结点，因为这部分元素对应的是树的右子树。根据这样的划分，把序列划分为左右两部分，我们递归地确认序列的左、右两部分是不是都是二叉查找树。  
**代码**：

	bool IsSequenceBST(int sequence[], int length) 
	{
		if(!sequence || length <= 0) {
			return false;
		}
		int root = sequence[length-1];
		int i=0;
		for(;i<length-1;++i) {
			if(sequence[i] > root) {
				break;
			}
		}
		int j=i;
		for(; j<length - 1; ++j) {
			if(sequence[j] < root) {
				return false;
			}
		}
		bool left = true;
		if(i > 0) {
			left = IsSequenceBST(sequence, i);
		}
		bool right = true;
		if(i < length - 1) {
			right = IsSequenceBST(sequence+i, length - i - 1);
		}
		return left && right;
	}

###从上往下遍历二叉树(层序遍历，广度优先遍历)