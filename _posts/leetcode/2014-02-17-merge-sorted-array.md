---
title: Merge Sorted Array
layout: post
categories: [Leetcode解题报告]
tags: [算法]
description: Given two sorted integer arrays A and B, merge...
---

**topic**  
Given two sorted integer arrays A and B, merge B into A as one sorted array.

**Note**：  
You may assume that A has enough space to hold additional elements from B. The number of elements initialized in A and B are m and n respectively.

	//
	class Solution {
	public:
	    void merge(int A[], int m, int B[], int n) {
	        int tail = m + n - 1;
	    	int tailA = m - 1;
	    	int tailB = n - 1;
	    
	    	while (tailA >= 0 && tailB >= 0)
	    	{
	    		if(A[tailA] > B[tailB])  {
	    			A[tail] = A[tailA];
	    			tailA--;
	    		}
	    		else if (A[tailA] < B[tailB]) {
	    			A[tail] = B[tailB];
	    			tailB--;
	    		}
	    		else {
	    			A[tail] = A[tailA];
	    			tailA--;
	    			tail--;
	    			A[tail] = B[tailB];
	    			tailB--;
	    		}
	    		tail--;
	    	}
	    	if (tailA < 0) {
	    		while (tail >= 0) {
	    			A[tail] = B[tailB];
	    			tailB--;
	    			tail--;
	    		}
	    	}    
	    }
	};

-EOF-