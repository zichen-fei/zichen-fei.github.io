---
layout: post
title: Kafka-文件存储机制
date: 2019-12-07
tags: 大数据,Kafka
---

Kafka 中消息是以 topic 进行分类的，生产者通过 topic 向 Kafka broker 发送消息，消费者通过 topic 读取数据。topic 在物理层面又能以 partition 为分组，
一个 topic 可以分成若干个 partition，partition 还可以细分为 segment，一个 partition 物理上由多个 segment 组成

### **topic中partition存储分布**

假设只有一个 Kafka 集群，且这个集群只有一个 Kafka broker。在这个 Kafka broker 中配置（KAFKAHOME/config/server.properties中）log.dirs=/tmp/kafka−logs，
以此来设置Kafka消息文件存储目录，与此同时创建一个topic：test，partition的数量为4

KAFKA_HOME/bin/kafka-topics.sh --create --zookeeper localhost:2181 --partitions 3 --topic test --replication-factor 3

```
drwxr-xr-x 2 root root 4096 Apr 10 16:10 test-0
drwxr-xr-x 2 root root 4096 Apr 10 16:10 test-1
drwxr-xr-x 2 root root 4096 Apr 10 16:10 test-2
```

在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，
第一个partiton序号从0开始，序号最大值为partitions数量减1，如果是多broker分布，参考[partitions/replicas分配策略](../../../2019/12/KafkaPartitonPolicy/)

### **partiton中文件存储方式**

下面示意图形象说明了partition中文件存储方式:

![](/images/posts/kafka/a1.png)

1、每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。

2、每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定。

这样做的好处就是能快速删除无用文件，有效提高磁盘利用率。

### **partiton中segment文件存储结构**

+ segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀”.index”和“.log”分别表示为segment索引文件、数据文件.

+ segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。
数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

![](/images/posts/kafka/a5.png)

以上图中一对segment file文件为例，说明segment中index<—->data file对应关系物理结构如下：

![](/images/posts/kafka/kafka_log.png)

![](/images/posts/kafka/a6.png)

索引文件中元数据指向对应数据文件中message的物理偏移地址。 其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、
以及该消息的物理偏移地址为497。

> kafka采取稀疏索引存储的方式，每隔一定字节的数据建立一条索引，它减少了索引文件大小，使得能够把 index 映射到内存，降低了查询时的磁盘IO开销，
>同时也并没有给查询带来太多的时间消耗。因为其文件名为上一个segment 最后一条消息的 offset ，所以当需要查找一个指定 offset 的 message 时，
>通过在所有 segment 的文件名中进行二分查找就能找到它归属的 segment ，再在其 index 文件中找到其对应到文件上的物理位置，就能拿出该 message 。

segment data file由许多message组成，参考[Kafka三个版本的消息格式](../../../)

### **在partition中通过offset查找message**

以上面为例，读取offset=368776的message，需要通过下面2个步骤查找。

+ 第一步查找segment file，其中00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0.第二个文件00000000000000368769.index
的消息量起始偏移量为368770 = 368769 + 1.同样，第三个文件00000000000000737337.index的起始偏移量为737338=737337 + 1，
其他后续文件依次类推，以起始偏移量命名并排序这些文件，只要根据offset ```二分查找``` 文件列表，就可以快速定位到具体文件。 
当offset=368776时定位到00000000000000368769.index|log

+ 第二步通过segment file查找message 通过第一步定位到segment file，当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置
和00000000000000368769.log的物理偏移地址，然后再通过00000000000000368769.log顺序查找直到offset=368776为止