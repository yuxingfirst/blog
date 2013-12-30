---
title: Linux下代码编辑三大件:vim、find、grep
layout: post
categories: [other]
tags: [vim]
description:  . 
---

###1. vim

关于vim的使用技巧，请参考下边的几篇文章:  

[【简明 Vim 练级攻略】][link1]  
[【Vim的分屏功能】][link2]  

如果你能够详细的阅读完上边两篇文章并且能结合vim动手做一些练习，相信你能很快掌握vim的各种用法，进而能在日常的工作中熟练的使用vim进行开发工作。  

我这里需要补充的一点就是在将vim分屏的时候，根据第二篇文章介绍的调整屏幕尺寸的方法不管用(有可能我们是用SecureCRT\Xshell连接到一台linux机器)，此时，我们就得用res命令。  

1. 在垂直分屏时，可以使用 <code>:vertical res n</code>， 来调整屏幕尺寸。
2. 在水平分屏时，使用 <code>:res n</code>，来调整屏幕尺寸。  

vim使用tips:

	0 → 到行头
	^ → 到本行第一个非blank字符
	$ → 到行尾
	g_ → 到本行最后一个不是blank字符的位置

	ctrl + d → 下翻半页
	ctrl + u → 上翻半页


###2. find

find是非常强大的查找命令。在我们日常的开发工作中，特别是在阅读代码的时候，肯定少不了需要查找文件，所以find就是必须要掌握的命令了。  

首先来看看find的用法:  

	find [path] [option] [action]  
	path: 要搜索的路径
	option: 搜索条件
	action: 对搜索结果进行处理  

如果什么参数也不加，find默认搜索当前目录及其子目录，并且不过滤任何结果（也就是返回所有文件），将它们全都显示在屏幕上。

find的使用实例：
	
	$ find . -name 'my*'
		搜索当前目录（含子目录，以下同）中，所有文件名以my开头的文件。

	$ find . -name 'my*' -ls



[link1]: http://coolshell.cn/articles/5426.html  
[link2]: http://coolshell.cn/articles/1679.html   

-EOF-