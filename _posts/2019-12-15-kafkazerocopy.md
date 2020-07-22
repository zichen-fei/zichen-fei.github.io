---
layout: post
title: Kafka-零拷贝
date: 2019-12-15
tags: Kafka
---

### **传统IO流程**

![](/images/posts/kafka/a14.png)

比如：读取文件，再用socket发送出去，传统方式实现是先读取、再发送，实际经过1~4四次copy。

+ 1、操作系统将数据从磁盘文件中读取到内核空间的页面缓存；

+ 2、应用程序将数据从内核空间读入用户空间缓冲区；

+ 3、应用程序将读到数据写回内核空间并放入socket缓冲区；

+ 4、操作系统将数据从socket缓冲区复制到网卡接口，此时数据才能通过网络发送。

### **网络数据持久化到磁盘 (Producer 到 Broker)**

传统方式实现：

```
data = socket.read()// 读取网络数据 
File file = new File() 
file.write(data)// 持久化到磁盘 
file.flush()
```

先接收生产者发来的消息，再落入磁盘。实际会经过四次copy

![](/images/posts/kafka/a15.png)

数据落盘通常都是非实时的，kafka生产者数据持久化也是如此。Kafka的数据并不是实时的写入硬盘，它充分利用了现代操作系统分页存储来利用内存提高I/O效率。

对于kafka来说，Producer生产的数据存到broker，这个过程读取到socket buffer的网络数据，其实可以直接在OS内核缓冲区，完成落盘。并没有必要将socket buffer的网络数据，读取到应用进程缓冲区；在这里应用进程缓冲区其实就是broker，broker收到生产者的数据，就是为了持久化。

在此特殊场景下：接收来自socket buffer的网络数据，应用进程不需要中间处理、直接进行持久化时。——可以使用mmap内存文件映射。

### **内核态和用户态**

从系统安全和保护的角度出发，在进行计算机体系结构设计时，处理机的执行模式一般设定为两种：分别称为内核模式(内核态)和用户模式(用户态)。

当处理机处于内核模式执行时，意味着系统除了可以执行一般指令外，还可以执行特权指令，即可以执行访问各种控制寄存器的指令、I/O指令以及程序状态字。

当处理机处于用户模式执行时，只能执行一般指令，而不允许执行特权指令。这样做可以保护核心代码不受用户程序有意和无意的攻击。

显然，处理机在运行期间需要在内核模式和用户模式之前进行切换。

### **为什么Kafka快**

1、顺序读写 

kafka将来自Producer的数据，顺序追加在partition，partition就是一个文件，以此实现顺序写入。

Consumer从broker读取数据时，因为自带了偏移量，接着上次读取的位置继续读，以此实现顺序读。 

顺序读写，是kafka利用磁盘特性的一个重要体现。

2、零拷贝 sendfile(in,out)

数据直接在内核完成输入和输出，不需要拷贝到用户空间再写出去。

kafka数据写入磁盘前，数据先写到进程的内存空间。

3、mmap文件映射 

虚拟映射只支持文件，在进程 的非堆内存开辟一块内存空间，和OS内核空间的一块内存进行映射， 

kafka数据写入、是写入这块内存空间，但实际这块内存和OS内核内存有映射，也就是相当于写在内核内存空间了，且这块内核空间、内核直接能够访问到，直接落入磁盘。 

### **sendfile**

Linux 2.4+ 内核通过 sendfile 系统调用，提供了零拷贝。磁盘数据通过 DMA 拷贝到内核态 Buffer 后，
直接通过 DMA 拷贝到 NIC Buffer(socket buffer)，无需 CPU 拷贝。除了减少数据拷贝外，因为整个读文件 - 网络发送由一个 sendfile 调用完成，
整个过程只有两次上下文切换，因此大大提高了性能。零拷贝过程如下图所示。

![](/images/posts/kafka/a17.png)

### **Java NIO对sendfile的支持**

```
FileChannel.transferTo()/transferFrom()。
fileChannel.transferTo( position, count, socketChannel);
```

把磁盘文件读取OS内核缓冲区后的fileChannel，直接转给socketChannel发送；底层就是sendfile。

transferTo 和 transferFrom 并不保证一定能使用零拷贝。实际上是否能使用零拷贝与操作系统相关，如果操作系统提供 sendfile 这样的零拷贝系统调用，
则这两个方法会通过这样的系统调用充分利用零拷贝的优势，否则并不能通过这两个方法本身实现零拷贝。

### **mmap(Memory Mapped Files)**

将磁盘文件映射到内存, 用户通过修改内存就能修改磁盘文件。

它的工作原理是直接利用操作系统的Page来实现文件到物理内存的直接映射。完成映射之后你对物理内存的操作会被同步到硬盘上（操作系统在适当的时候）。

通过mmap，进程像读写硬盘一样读写内存（当然是虚拟机内存），也不必关心内存的大小有虚拟内存为我们兜底。使用这种方式可以获取很大的I/O提升，省去了用户空间到内核空间复制的开销。

mmap也有一个很明显的缺陷——不可靠，写到mmap中的数据并没有被真正的写到硬盘，操作系统会在程序主动调用flush的时候才把数据真正的写到硬盘。

Kafka提供了一个参数——producer.type来控制是不是主动flush；如果Kafka写入到mmap之后就立即flush然后再返回Producer叫同步(sync)；写入mmap之后立即返回Producer不调用flush叫异步(async)。

### **Java NIO对文件映射的支持**

Java NIO，提供了一个 MappedByteBuffer 类可以用来实现内存映射。
MappedByteBuffer只能通过调用FileChannel的map()取得
FileChannel.map()是抽象方法，具体实现是在 FileChannelImpl.c 其map0()方法就是调用了Linux内核的mmap的API。

使用 MappedByteBuffer类要注意的是：mmap的文件映射，在full gc时才会进行释放。当close时，需要手动清除内存映射文件，可以反射调用sun.misc.Cleaner方法。

DirectByteBuffer继承于MappedByteBuffer，可以开辟了一段直接的内存，并不会占用jvm的内存空间；
通过Filechannel映射出的MappedByteBuffer其实际也是DirectByteBuffer，除了这种方式，也可以手动开辟一段空间：




