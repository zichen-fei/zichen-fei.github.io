---
layout: post
title: Kafka-数据可可靠性和一致性
date: 2019-12-07
tags: 大数据,Kafka
---

### **Partition Recovery机制**

每个Partition会在磁盘记录一个RecoveryPoint, 记录已经flush到磁盘的最大offset。当broker fail 重启时,会进行loadLogs。 
首先会读取该Partition的RecoveryPoint,找到包含RecoveryPoint的segment及以后的segment, 这些segment就是可能没有 完全flush到磁盘segments。
然后调用segment的recover,重新读取各个segment的msg,并重建索引
