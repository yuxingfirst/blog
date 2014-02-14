---
title: Longest Substring Without Repeating Characters
layout: post
categories: [Leetcode解题报告]
tags: [算法]
description: Given a string, find the length of the longest substring...
---

Given a string, find the length of the longest substring without repeating characters. For example, the longest substring without repeating letters for "abcabcbb" is "abc", which the length is 3. For "bbbbb" the longest substring is "b", with the length of 1.

	class Solution {
	public:
	    int lengthOfLongestSubstring(string s) {
	        int iLength = s.length();
	    	if (iLength <= 1) {
	    		return iLength;
	    	}
	    	int iStartIdx = 0, iEndIdx = 1;
	    	int maxSubstrLength = 0;
	    	while (iEndIdx < iLength) {		
	    		char sCh = s[iEndIdx];
	    		int sChIdx = s.find_first_of(sCh, iStartIdx);
	    		if (sChIdx < iEndIdx) {			
	    			if (maxSubstrLength < iEndIdx - iStartIdx) {
	    				maxSubstrLength = iEndIdx - iStartIdx;
	    			}
	    			iStartIdx = sChIdx + 1;
	    		}
	    
	    		if (iEndIdx == iLength - 1) {		
	    			if (maxSubstrLength < iEndIdx - iStartIdx + 1) {
	    				maxSubstrLength = iEndIdx - iStartIdx + 1;
	    			}
	    			break;
	    		}
	    		iEndIdx++;		
	    	}
	    	return maxSubstrLength;
	    }
	};

-EOF-