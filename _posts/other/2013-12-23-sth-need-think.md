---
title: Distribution system
layout: post
categories: [技术研究]
tags: []
description: .
---

> 分布式系统简单说就是一组服务器协同工作，主要目的在于均衡负载、均衡数据、冗余等，面对的主要难点在于数据同步、防止宕机造成连锁反应、减少服务器间通讯及依赖、处理任意一个节点缓慢造成任务堆积或失败等方面，还有一点就是不要信任网络，这是最不稳定的部分，即便是同一机房。最近几个月通过改善开发过程、规范单元测试、增加容错机制、消除单点增加冗余、完善监控系统、细化统计分析等方面来提高质量，效果还是明显的，但还有很多路要走。系统设计阶段着眼于大方向上，开发阶段小细节上也需要特别在意，比如日志，系统一旦上线这可能是最丰富稳定的信息源。


Task list:

1. redis 主从同步的详细机制，持久化机制(dumpfile vs aof)。思考如何确保数据的高可用性(宕机不丢失数据)机制。
2. 协程切换时栈是**如何**保存的。
3. 