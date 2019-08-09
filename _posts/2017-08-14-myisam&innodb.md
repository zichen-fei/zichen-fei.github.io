---
layout: post
title: MySQL存储引擎InnoDB与Myisam的区别
date: 2017-08-14
tags: 数据库
---

### MySQL存储引擎InnoDB与Myisam的区别  

MyISAM存储引擎的特点：  
&emsp;&emsp;表级锁、不支持事务和全文索引，适合一些CMS内容管理系统作为后台数据库使用，但是使用大并发、重负荷生产系统上，表锁结构的特性就显得力不从心。  

InnoDB存储引擎的特点是：  
&emsp;&emsp;行级锁、事务安全（ACID兼容）、支持外键、不支持FULLTEXT类型的索引(5.6.4以后版本开始支持FULLTEXT类型的索引)。InnoDB存储引擎提供了具有提交、回滚和崩溃恢复能力的事务安全存储引擎。InnoDB是为处理巨大量时拥有最大性能而设计的。它的CPU效率可能是任何其他基于磁盘的关系数据库引擎所不能匹敌的

### 区别：

 + InnoDB支持事务，MyISAM不支持。事务是一种高级的处理方式，如在一些列增删改中只要哪个出错还可以回滚还原，而MyISAM就不可以了。
 + MyISAM适合查询以及插入为主的应用，InnoDB适合频繁修改以及涉及到安全性较高的应用
 + InnoDB支持外键，MyISAM不支持
 + MyISAM是默认引擎，InnoDB需要指定
 + InnoDB不支持FULLTEXT类型的索引
 + InnoDB中不保存表的行数，如select count(*) from table时，InnoDB需要扫描一遍整个表来  计算有多少行，但是MyISAM只要简单的读出保存好的行数即可。注意的是，当count(*)语句包含where条件时MyISAM也需要扫描整个表
 + 对于自增长的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中可以和其他字段一起建立联合索引
 + 清空整个表时，InnoDB是一行一行的删除，效率非常慢。MyISAM则会重建表
 + InnoDB支持行锁（某些情况下还是锁整表，如 update table set a=1 where user like '%lee%'
 + MyISAM的索引和数据是分开的，并且索引是有压缩的，内存使用率就对应提高了不少。能加载更多索引，而Innodb是索引和数据是紧密捆绑的，没有使用压缩从而会造成Innodb比MyISAM体积庞大不小
