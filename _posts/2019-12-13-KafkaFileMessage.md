---
layout: post
title: Kafka-message格式
date: 2019-12-13
tags: Kafka
---

Kafka根据topic（主题）对消息进行分类，发布到Kafka集群的每条消息都需要指定一个topic，每个topic将被分为多个partition（分区）。
每个partition在存储层面是追加log（日志）文件，任何发布到此partition的消息都会被追加到log文件的尾部

![](/images/posts/kafka/a7.png)

每一条消息被发送到Kafka中，会根据一定的规则选择被存储到哪一个partition中。如果规则设置的合理，所有的消息可以均匀分布到不同的partition里，
这样就实现了水平扩展。如上图，每个partition由其上附着的每一条消息组成，如果消息格式设计的不够精炼，那么其功能和性能都会大打折扣。
比如有冗余字段，势必会使得partition不必要的增大，进而不仅使得存储的开销变大、网络传输的开销变大，也会使得Kafka的性能下降；
又比如缺少字段，在最初的Kafka消息版本中没有timestamp字段，对内部而言，其影响了日志保存、切分策略，对外部而言，其影响了消息审计、端到端延迟等功能的扩展，
虽然可以在消息体内部添加一个时间戳，但是解析变长的消息体会带来额外的开销，而存储在消息体（参考下图中的value字段）前面可以通过指针偏量获取其值而容易解析，
进而减少了开销（可以查看v1版本），虽然相比于没有timestamp字段的开销会差一点。

### **V0版本**

对于Kafka消息格式的第一个版本，称之为v0，在Kafka 0.10.0版本之前都是采用的这个消息格式。

![](/images/posts/kafka/a8.png)

上左图中的“RECORD”部分就是v0版本的消息格式，包括offset和message size字段都看成是消息，因为每个Record（v0和v1版）必定对应一个
offset和message size。每条消息都一个offset用来标志它在partition中的偏移量，这个offset是逻辑值，而非实际物理偏移值，
message size表示消息的大小，这两者的一起被称之为日志头部（LOG_OVERHEAD），固定为12B。LOG_OVERHEAD和RECORD一起用来描述一条消息。
与消息对应的还有消息集的概念，消息集中包含一条或者多条消息，消息集不仅是存储于磁盘以及在网络上传输（Produce & Fetch）的基本形式，
而且是kafka中压缩的基本单元，详细结构参考上右图。

从crc32开始算起，各个字段的解释如下：

+ crc32（4B）：crc32校验值。校验范围为magic至value之间。

+ magic（1B）：消息格式版本号，此版本的magic值为0。

+ attributes（1B）：消息的属性。总共占1个字节，低3位表示压缩类型：0表示NONE、1表示GZIP、2表示SNAPPY、3表示LZ4（LZ4自Kafka 0.9.x引入），其余位保留。

+ key length（4B）：表示消息的key的长度。如果为-1，则表示没有设置key，即key=null。

+ key：可选，如果没有key则无此字段。

+ value length（4B）：实际消息体的长度。如果为-1，则表示消息为空。

+ value：消息体。可以为空，比如tomnstone消息。

v0版本中一个消息的最小长度（RECORD_OVERHEAD_V0）为crc32 + magic + attributes + key length + value length = 4B + 1B + 1B + 4B + 4B =14B，
也就是说v0版本中一条消息的最小长度为14B，如果小于这个值，那么这就是一条破损的消息而不被接受。

### **V1版本**

kafka从0.10.0版本开始到0.11.0版本之前所使用的消息格式版本为v1，其比v0版本就多了一个timestamp字段，表示消息的时间戳。v1版本的消息结构图如下所示：

![](/images/posts/kafka/a9.png)

v1版本的magic字段值为1。v1版本的attributes字段中的低3位和v0版本的一样，表示压缩类型，而第4个bit也被利用了起来：0表示timestamp类型为CreateTime，
而1表示tImestamp类型为LogAppendTime，其他位保留。v1版本的最小消息（RECORD_OVERHEAD_V1）大小要比v0版本的要大8个字节，即22B。
如果像v0版本介绍的一样发送一条```key="key"，value="value"```的消息，那么此条消息在v1版本中会占用42B，

### **消息压缩**

常见的压缩算法是数据量越大压缩效果越好，一条消息通常不会太大，这就导致压缩效果并不太好。而kafka实现的压缩方式是将多条消息一起进行压缩，
这样可以保证较好的压缩效果。而且在一般情况下，**生产者发送的压缩数据在kafka broker中也是保持压缩状态进行存储，消费者从服务端获取也是压缩的消息，
消费者在处理消息之前才会解压消息**，这样保持了端到端的压缩。

当消息压缩时是将整个消息集进行压缩而作为内层消息（inner message），内层消息整体作为外层（wrapper message）的value，其结构图如下所示：

![](/images/posts/kafka/a11.png)

压缩后的外层消息（wrapper message）中的key为null，所以图右部分没有画出key这一部分。当生产者创建压缩消息的时候，
对内部压缩消息设置的offset是从0开始为每个内部消息分配offset，详细可以参考下图右部

![](/images/posts/kafka/a12.png)

每个从生产者发出的消息集中的消息offset都是从0开始的，当然这个offset不能直接存储在日志文件中，对offset进行转换时在服务端进行的，
客户端不需要做这个工作。外层消息保存了内层消息中最后一条消息的绝对位移（absolute offset），绝对位移是指相对于整个partition而言的。
参考上图，对于未压缩的情形，图右内层消息最后一条的offset理应是1030，但是被压缩之后就变成了5，而这个1030被赋予给了外层的offset。
当消费者消费这个消息集的时候，首先解压缩整个消息集，然后找到内层消息中最后一条消息的inner offset，
然后根据如下公式找到内层消息中最后一条消息前面的消息的absolute offset（RO表示Relative Offset，IO表示Inner Offset，而AO表示Absolute Offset）：

```
RO = IO_of_a_message - IO_of_the_last_message 
AO = AO_Of_Last_Inner_Message + RO
```

这里RO是前面的消息相对于最后一条消息的IO而言的，所以其值小于等于0，0表示最后一条消息自身。

v1版本比v0版的消息多了个timestamp的字段。对于压缩的情形，外层消息的timestamp设置为：

+ 如果timestamp类型是CreateTime，那么设置的是内层消息中最大的时间戳（the max timestampof inner messages if CreateTime is used）

+ 如果timestamp类型是LogAppendTime，那么设置的是kafka服务器当前的时间戳

内层消息的timestamp设置为：

+ 如果外层消息的timestamp类型是CreateTime，那么设置的是生产者创建消息时的时间戳。

+ 如果外层消息的timestamp类型是LogAppendTime，那么所有的内层消息的时间戳都将被忽略。

### **V2版本**

kafka从0.11.0版本开始所使用的消息格式版本为v2，参考了Protocol Buffer而引入了变长整型（Varints）和ZigZag编码。
Varints是使用一个或多个字节来序列化整数的一种方法，数值越小，其所占用的字节数就越少。ZigZag编码以一种锯齿形（zig-zags）的方式来回穿梭于正负整数之间，
以使得带符号整数映射为无符号整数，这样可以使得绝对值较小的负数仍然享有较小的Varints编码值，比如-1编码为1,1编码为2，-2编码为3。

kafka v0和v1版本的消息格式，如果消息本身没有key，那么key length字段为-1，int类型的需要4个字节来保存，而如果采用Varints来编码则只需要一个字节。
根据Varints的规则可以推导出0-63之间的数字占1个字节，64-8191之间的数字占2个字节，8192-1048575之间的数字占3个字节。
而kafka broker的配置message.max.bytes的默认大小为1000012（Varints编码占3个字节），如果消息格式中与长度有关的字段采用Varints的编码的话，
绝大多数情况下都会节省空间，而v2版本的消息格式也正是这样做的。不过需要注意的是Varints并非一直会省空间，一个int32最长会占用5个字节（大于默认的4字节），
一个int64最长会占用10字节（大于默认的8字节）。

v2版本中消息集为Record Batch，而不是先前的Message Set了，其内部也包含了一条或者多条消息，消息的格式参见下图中部和右部。在消息压缩的情形下，
Record Batch Header部分（参见下图左部，从first offset到records count字段）是不被压缩的，而被压缩的是records字段中的所有内容。

![](/images/posts/kafka/a10.png)

v2版本对于消息集（RecordBatch）做了彻底的修改，参考上图左部，除了刚刚提及的crc字段，还多了如下字段：

+ first offset：表示当前RecordBatch的起始位移。

+ length：计算partition leader epoch到headers之间的长度。

+ partition leader epoch：用来确保数据可靠性，详细可以参考KIP-101

+ magic：消息格式的版本号，对于v2版本而言，magic等于2。

+ attributes：消息属性，注意这里占用了两个字节。低3位表示压缩格式，可以参考v0和v1；第4位表示时间戳类型；第5位表示此RecordBatch是否处于事务中，0表示非事务，1表示事务。第6位表示是否是Control消息，0表示非Control消息，而1表示是Control消息，Control消息用来支持事务功能。

+ last offset delta：RecordBatch中最后一个Record的offset与first offset的差值。主要被broker用来确认RecordBatch中Records的组装正确性。

+ first timestamp：RecordBatch中第一条Record的时间戳。

+ max timestamp：RecordBatch中最大的时间戳，一般情况下是指最后一个Record的时间戳，和last offset delta的作用一样，用来确保消息组装的正确性。

+ producer id：用来支持幂等性，详细可以参考KIP-98。

+ producer epoch：和producer id一样，用来支持幂等性。

+ first sequence：和producer id、producer epoch一样，用来支持幂等性。

+ records count：RecordBatch中Record的个数。
