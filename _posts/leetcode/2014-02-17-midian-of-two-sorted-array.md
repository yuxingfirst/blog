---
title: Median of Two Sorted Arrays
layout: post
categories: [Leetcode解题报告]
tags: [算法]
description: There are two sorted arrays A and B of ...
---

**topic**  
There are two sorted arrays A and B of size m and n respectively. Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).

找出两个排序数组的中位数，一个非常直观的方法，先合并两个数组，这里的时间复杂度是O(m+n)，然后可以以O(1)的时间复杂度求的其中位数。但这里要求的时间复杂度是O(log(m+n)), 所以需要思考其他的方案。

我们知道，二分查找的时间复杂度是O(log n)，所以如果能够把二分的思想运用到这里，那么我们就可以在指定的时间复杂度下解决问题。  

由于两个数组已经是有序的了，那么，我们可以先比较两个数组的中位数， Mid(A/2)和Mid(B/2)；如果这两个数相等，则可以直接求中位数；如果Mid(A/2)小于Mid(B/2)，则中位数必定在: A/2 ~ A, 0 ~ B/2这两个区间之内。反之， 如果Mid(A/2)大于Mid(B/2)，则中位数必定在: B/2 ~ B, 0 ~ A/2这两个区间之内。  

按照这个方法，我们不断以二分的方式缩小范围区间，直到最后缩小到只包含1个元素时结束。

下边，我们以递归方式来实现这个算法:

	//

-EOF-