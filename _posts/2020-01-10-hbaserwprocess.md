---
layout: post
title: HBase读写流程
date: 2020-01-10
tags: HBase
---

### **WAL机制**

WAL(Write-Ahead Logging)，基本原理是在数据写入之前首先顺序写入日志，然后再写入缓存，等到缓存写满之后统一落盘。之所以能够提升写性能，
是因为WAL将一次随机写转化为了一次顺序写加一次内存写。提升写性能的同时，WAL可以保证数据的可靠性，即在任何情况下数据不丢失。
假如一次写入完成之后发生了宕机，即使所有缓存中的数据丢失，也可以通过恢复日志还原出丢失的数据。

WAL(Write-Ahead-Log)预写日志是HBase的RegionServer在处理数据插入和删除的过程中用来记录操作内容的一种日志。
只有当WAL日志写成功以后，客户端才会被告诉提交数据成功，如果写WAL失败会告知客户端提交失败。

### **读流程**

+ 1、客户端从zookeeper中获取meta表所在的regionserver节点信息

+ 2、客户端访问meta表所在的regionserver节点，获取到region所在的regionserver信息

+ 3、客户端访问具体的region所在的regionserver，找到对应的region及store

+ 4、首先从memstore中读取数据，如果读取到了那么直接将数据返回，如果没有，则去blockcache读取数据
 
+ 5、如果blockcache中读取到数据，则直接返回数据给客户端，如果读取不到，则遍历storefile文件，查找数据

+ 6、如果从storefile中读取不到数据，则返回客户端为空，如果读取到数据，那么需要将数据先缓存到blockcache中（方便下一次读取），然后再将数据返回给客户端

+ 7、blockcache是内存空间，如果缓存的数据比较多，满了之后会采用LRU策略，将比较老的数据进行删除

如何保证client读取blockcache内为新数据？

cache与HFile共同维护一索引表，索引表内指向最新版本数据源，当client读取指定rowkey数据时会访问索引，索引给出给出最新版本数据，
如blockcache不是最新的，那本次会读取hfile并更新blockcache数据，从而使client访问最新数据

### **写流程**

+ 1、客户端从zookeeper中获取meta表所在的regionserver节点信息
 
+ 2、客户端访问meta表所在的regionserver节点，获取到region所在的regionserver信息

+ 3、客户端访问具体的region所在的regionserver，找到对应的region及store

+ 4、开始写数据，写数据的时候会先向hlog中写数据（方便memstore中数据丢失后能够根据hlog恢复数据，向hlog中写数据的时候也是优先写入内存，
后台会有一个线程定期异步刷写数据到hdfs，如果hlog的数据也写入失败，那么数据就会发生丢失）

+ 5、hlog写数据完成之后，再将数据写入到memstore，memstore默认大小是128M，当memstore满了之后会进行统一的溢写操作，将memstore中的数据持久化到hdfs中

+ 6、频繁的溢写会导致产生很多的小文件，因此会进行文件的合并（Compact），文件在合并的时候有两种方式，minor和major， minor表示小范围文件的合并，major表示将所有的storefile文件都合并成一个

+ 7、当文件达到一定数量（默认3）就会触发Compact操作，将多个HFile合并成一个的HFile文件，同时进行版本合并和数据删除

+ 8、当HFile越来越大，导致单个HFile大小超过一定阈值（默认10G）后，触发Split操作，把当前Region Split成2个Region，Region会下线，
新Split出的Region会被HMaster分配到相应的HRegionServer 上，使得原先1个Region的压力得以分流到2个Region上

