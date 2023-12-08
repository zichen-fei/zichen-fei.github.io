---
layout: post
title: 使用连接池导致metastore连接被打满
date: 2022-09-19
tags: Hive
---

现象：

![](/images/posts/HiveMetastore/img01.png)

GenericObjetPool 配置的5个连接，为什么会创建这么多连接？
看日志在不断的在init和close连接

![](/images/posts/HiveMetastore/img02.png)

先看下连接池配置参数

maxActive                     最大活跃连接数  
 maxIdle                       链接池中最大空闲的连接数  
 minIdle                       连接池中最少空闲的连接数  
 maxWait                       当连接池资源耗尽时，调用者最大阻塞的时间  
 timeBetweenEvictionRunsMillis 空闲链接检测线程，检测的周期，毫秒数，-1 表示关闭空闲检测  
 minEvictableIdleTimeMillis    空闲链接超过时间后被销毁，-1 表示不检查空闲时间  
 testWhileIdle                 空闲时是否进行连接有效性验证，如果验证失败则移除，默认为 true  
 testOnBorrow                  判断这条连接是否是可用  
 testOnReturn                  连接池回收连接的时候会判断该连接是否还可用

关键在 minEvictableIdleTimeMillis 这个参数，空闲连接时长，默认30分钟，timeBetweenEvictionRunsMillis 配置的5s，就是每隔5s检测连接的有效性和空闲时间，如果空闲时间超过三十分钟，就会调用 destroyObject 方法，关闭连接，从连接池中移除

minIdle 连接池中最少的空闲链接数，配置的是5个，destory一个，在重新 init 一个加到池子里，所以会反复的 close，init

![](/images/posts/HiveMetastore/img03.png)

但是为啥调用了 destroyObject 方法，链接没有关闭？

```
public void destroyObject(ThriftHiveMetastore.Client client) {
    logger.info("close metastore client");
    try {
        client.shutdown();
    } catch (TException e) {
        logger.error("关闭连接失败!", e);
    }
}
```

看下 client.shutdown() 方法，最后执行的是

![](/images/posts/HiveMetastore/img04.png)

pm 就是 PersistenceManager

只是关闭了数据库的连接

而client初始化方式是

```
public ThriftHiveMetastore.Client makeObject() throws Exception {
    logger.info("init metastore client");
    for (String host: hosts) {
        TTransport transport = new TSocket(host, port);
        TProtocol protocol = new TBinaryProtocol(transport);
        ThriftHiveMetastore.Client client = new ThriftHiveMetastore.Client(protocol);
        transport.open();
        return client;
    }
    return null;
}
```

没有关闭底层的 TSocket，连接没有真正关闭，所以会导致连接数不断增多

修改关闭连接的方法，参考了 HiveMetaStoreClient.close() 方法

```
public void destroyObject(ThriftHiveMetastore.Client client) {
    logger.info("close metastore client");
    try {
        client.shutdown();
    } catch (TException e) {
        logger.error("关闭连接失败!", e);
    }
 
    try {
        //HiveMetaStoreClient.close() 方法
        // Transport would have got closed via client.shutdown(), so we dont need this, but
        // just in case, we make this call.
        TTransport transport = client.getOutputProtocol().getTransport();
        if ((transport != null) && transport.isOpen()) {
            transport.close();
            logger.info("Closed connection to metastore");
        }
    } catch (Exception e) {
        logger.error("transport close error!", e);
    }
}
```

HiveMetaStoreClient.close() 方法

![](/images/posts/HiveMetastore/img05.png)

为啥没直接用封装好的 HiveMetaStoreClient？原因：[https://issues.apache.org/jira/browse/HIVE-23700](https://issues.apache.org/jira/browse/HIVE-23700)

测试：连接池配置空闲的连接数为 1 个

在 hivedev01.dev 这台机器上看当前的连接数

```
netstat -na |grep 9083|grep -v LISTEN | wc -l
```

![](/images/posts/HiveMetastore/img06.png)

修改close方法前，运行一段时间

![](/images/posts/HiveMetastore/img8.png)

修改close方法后，运行一段时间

![](/images/posts/HiveMetastore/img07.png)

修改后连接数没有一直增加，问题修复

