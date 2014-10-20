---
title: mysql事务和锁
layout: post
categories: [DB]
tags: [mysql]
description: .
--- 

我们知道，数据库事务，有4个特性：

	原子性
	一致性
	隔离性
	持久性


事务必须是原子工作单元，对于事务中的每一个操作序列，要么都执行、要么都不执行。
