---
title: 二叉树的深度
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


分析：  
要计算二叉树的深度，首先我们可以想到的是，从树根节点开始，得到树的所有路径计算长度并保存，最后去最长的那条路径即为树的深度。  
按照这个思想，代码写起来可能会比较麻烦，我们不妨从另外一个角度来思考。  

如果为空树，则深度为0；  
如果只有一个节点，则深度为1；  
如果根节点只有左子树没有右子树，则其深度为左子树的深度加1；  
如果根节点只有右子树没有左子树，则其深度为右子树的深度加1。  
如果左右子树都有，则树的深度为左右子树中深度较大的那颗子树的深度加1。  

显然，按照上述的这个方法，采用递归的方式，可以很容易的写出求树深度的代码：  

	//
	int bstree_depth(BSTreeNode *phead) {
		if(!phead) {
			return 0;
		}
		int left_depth = bstree_depth(phead->pLeft);
		int right_depth = bstree_depth(phead->pRight);

		return left_depth > right_depth ? left_depth + 1: right_depth + 1;
	}


-EOF-
