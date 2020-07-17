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







