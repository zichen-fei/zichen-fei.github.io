---
layout: post
title: HiveSQL注释问题排查
date: 2023-03-02
tags: Hive
---

现象：

```
--CREATE EXTERNAL TABLE ads_meta_dev.ads_meta_flow_aa_index_pd
set hive.merge.tezfiles = true;
```

这条SQL用beeline查询没问题

```
beeline -u 'jdbc:hive2://hive-adhoc:10000/default?mapreduce.job.queuename=tc_infra_tsn_dev;auth=noSasl' -n tc_warehouse -p hive_PY8fJ70p --hiveconf hive.stats.autogather=false --hiveconf hive.strict.checks.large.query=false --hiveconf hive.mapred.mode=nostrict -e "--CREATE EXTERNAL TABLE ads_meta_dev.ads_meta_flow_aa_index_pd \n set hive.merge.tezfiles = true;"
 
echo '--CREATE EXTERNAL TABLE ads_meta_dev.ads_meta_flow_aa_index_pd \n set hive.merge.tezfiles = true;' > /tmp/test.sql
beeline -u 'jdbc:hive2://hive-adhoc:10000/default?mapreduce.job.queuename=tc_infra_tsn_dev;auth=noSasl' -n tc_warehouse -p hive_PY8fJ70p --hiveconf hive.stats.autogather=false --hiveconf hive.strict.checks.large.query=false --hiveconf hive.mapred.mode=nostrict -f /tmp/test.sql
```

但是在即席查询平台就会报错

![](/images/posts/HiveSQLComment/img01.png)

平台把异常做了封装，看不到具体的堆栈日志

所以用正式环境的hive源码编译在本地搭建hiveserver服务，复现问题

hiveserver日志

![](/images/posts/HiveSQLComment/img02.png)

![](/images/posts/HiveSQLComment/img03.png)

只看堆栈日志也看不出啥问题，很正常的一条sql为啥用HiveStatement方式查询回报错？

即席查询执行SQL的方式

![](/images/posts/HiveSQLComment/img04.png)

debug跟踪下代码

从哪个地方入手？看堆栈信息报错的地方是 PareserDriver.parse 方法，在这个地方打上断点

![](/images/posts/HiveSQLComment/img05.png)

![](/images/posts/HiveSQLComment/img06.png)

再往下就是parse解析器，应该不会是解析器的问题

但是set命令为啥会走到 解析 SQL的 ParseDriver 呢？应该是前面就出问题了

往上翻堆栈

![](/images/posts/HiveSQLComment/img07.png)

发现 这个地方的 operation (ExecuteStatementOperation) 是个抽象类

而实现是 HiveCommandOperation 和 SQLOperation

![](/images/posts/HiveSQLComment/img08.png)

更加印证了之前的猜想，set 语句应该是 command命令，应该用 HiveCommandOperation，但是现在却是用的 SQLOperation

看operation具体是怎么生成的

![](/images/posts/HiveSQLComment/img09.png)

具体逻辑在  CommandProcessorFactory.getForHiveCommand(tokens, parentSession.getHiveConf())  这个方法的参数 tokens 是将 SQL 按空格分割成的数组，依据tokens数组的第一个字符串是不是特殊命令来判断是不是command命令

![](/images/posts/HiveSQLComment/img10.png)

这条SQL分割出来的结果是 

```
["--CREATE", "EXTERNAL", "TABLE", "ads_meta_dev.ads_meta_flow_aa_index_pd", "set", "hive.merge.tezfiles", "=", "true"]，返回的是 SQLOperation，所以用到了ParseDriver
```

感觉是 Hive 的一个bug

看Hive的高版本有没有修复这个问题

![](/images/posts/HiveSQLComment/img11.png)

翻到 ExecuteStatementOperation 这个文件的修改历史，果然找到一个patch [HIVE-16935](https://issues.apache.org/jira/browse/HIVE-16935) 修复了这个问题，但是在3.0.0版本才上线

只是加了一行处理  HiveStringUtils.removeComments(statement)  就是把SQL的注释都去掉

在公司hive-2.1.1的源码加上这段逻辑，在本地重新编译打包，运行同样的SQL没有再报错

但是要改线上的hive要重启服务，影响比较大，把HiveStringUtils类复制放到zue上，在下发查询的时候处理SQL，同样也能解决问题

