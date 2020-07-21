---
layout: post
title: Kafka-数据可可靠性和一致性
date: 2019-12-09
tags: Kafka
---

### **Partition Recovery机制**

每个Partition会在磁盘记录一个RecoveryPoint, 记录已经flush到磁盘的最大offset。当broker fail 重启时,会进行loadLogs。 
首先会读取该Partition的RecoveryPoint,找到包含RecoveryPoint的segment及以后的segment, 这些segment就是可能没有 完全flush到磁盘segments。
然后调用segment的recover,重新读取各个segment的msg,并重建索引

优点：

+ 以segment为单位管理Partition数据,方便数据生命周期的管理,删除过期数据简单

+ 在程序崩溃重启时,加快recovery速度,只需恢复未完全flush到磁盘的segment

+ 通过index中offset与物理偏移映射,用二分查找能快速定位msg,并且通过分多个Segment,每个index文件很小,查找速度更快。

### **Partition Replica同步机制**

+ Partition的多个replica中一个为Leader,其余为follower

+ Producer只与Leader交互,把数据写入到Leader中

+ Followers从Leader中拉取数据进行数据同步

+ Consumer只从Leader拉取数据

ISR:所有不落后的replica集合, 不落后有两层含义:距离上次FetchRequest的时间不大于某一个值或落后的消息数不大于某一个值, Leader失败后会从ISR中选取一个Follower做Leader

### **数据可靠性保证**

当Producer向Leader发送数据时,可以通过acks参数设置数据可靠性的级别

+ acks = 0：意味着如果生产者能够通过网络把消息发送出去，那么就认为消息已成功写入 Kafka 。在这种情况下还是有可能发生错误，
比如发送的对象无能被序列化或者网卡发生故障，但如果是分区离线或整个集群长时间不可用，那就不会收到任何错误。
在 acks=0 模式下的运行速度是非常快的（这就是为什么很多基准测试都是基于这个模式），你可以得到惊人的吞吐量和带宽利用率，不过如果选择了这种模式， 一定会丢失一些消息。

+ acks = 1：意味若 Leader 在收到消息并把它写入到分区数据文件（不一定同步到磁盘上）时会返回确认或错误响应。在这个模式下，如果发生正常的 Leader 选举，
生产者会在选举时收到一个 LeaderNotAvailableException 异常，如果生产者能恰当地处理这个错误，它会重试发送悄息，最终消息会安全到达新的 Leader 那里。
不过在这个模式下仍然有可能丢失数据，比如消息已经成功写入 Leader，但在消息被复制到 follower 副本之前 Leader发生崩溃。

acks = all（和acks = -1 含义一样）：意味着 Leader 在返回确认或错误响应之前，会等待所有同步副本都收到悄息。
如果和 min.insync.replicas 参数结合起来，就可以决定在返回确认前至少有多少个副本能够收到悄息，生产者会一直重试直到消息被成功提交。
不过这也是最慢的做法，因为生产者在继续发送其他消息之前需要等待所有副本都收到当前的消息。

另外，Producer 发送消息还可以选择同步（默认，通过 producer.type=sync 配置） 或者异步（producer.type=async）模式。
如果设置成异步，虽然会极大的提高消息发送的性能，但是这样会增加丢失数据的风险。如果需要确保消息的可靠性，必须将 producer.type 设置为 sync

> 使用异步模式的时候，当缓冲区满了，如果配置为0（还没有收到确认的情况下，缓冲池一满，就清空缓冲池里的消息），数据就会被立即丢弃掉。

#### **Leader 选举**

每个分区的 leader 会维护一个 ISR 列表，ISR 列表里面就是 follower 副本的 Borker 编号，只有跟得上 Leader 的 follower 副本才能加入到 ISR 里面，
这个是通过 replica.lag.time.max.ms 参数配置的，具体参见[Kafka副本同步机制理解]()。只有 ISR 里的成员才有被选为 leader 的可能。

当 Leader 挂掉了，而且 unclean.leader.election.enable=false 的情况下，Kafka 会从 ISR 列表中选择第一个 follower 作为新的 Leader，
因为这个分区拥有最新的已经 committed 的消息。通过这个可以保证已经 committed 的消息的数据可靠性。

综上所述，为了保证数据的可靠性，我们最少需要配置一下几个参数：

+ producer 级别：acks=all（或者 request.required.acks=-1），同时发生模式为同步 producer.type=sync

+ topic 级别：设置 replication.factor>=3，并且 min.insync.replicas>=2；

+ broker 级别：关闭不完全的 Leader 选举，即 unclean.leader.election.enable=false；

### **数据一致性**

数据一致性主要是说不论是老的 Leader 还是新选举的 Leader，Consumer 都能读到一样的数据。

![](/images/posts/kafka/a13.png)

假设分区的副本为3，其中副本0是 Leader，副本1和副本2是 follower，并且在 ISR 列表里面。虽然副本0已经写入了 Message4，但是 Consumer 只能读取到 Message2。
因为所有的 ISR 都同步了 Message2，只有 High Water Mark 以上的消息才支持 Consumer 读取，而 High Water Mark 取决于 ISR 列表里面偏移量最小的分区，
对应于上图的副本2，类似于木桶原理。

这样做的原因是还没有被足够多副本复制的消息被认为是“不安全”的，如果 Leader 发生崩溃，另一个副本成为新 Leader，那么这些消息很可能丢失了。
如果我们允许消费者读取这些消息，可能就会破坏一致性。试想，一个消费者从当前 Leader（副本0） 读取并处理了 Message4，这个时候 Leader 挂掉了，
选举了副本1为新的 Leader，这时候另一个消费者再去从新的 Leader 读取消息，发现这个消息其实并不存在，这就导致了数据不一致性问题。

引入了 High Water Mark 机制，会导致 Broker 间的消息复制因为某些原因变慢，那么消息到达消费者的时间也会随之变长（因为我们会先等待消息复制完毕）。
延迟时间可以通过参数 replica.lag.time.max.ms 参数配置，它指定了副本在复制消息时可被允许的最大延迟时间。


