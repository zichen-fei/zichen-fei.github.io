---
layout: post
title: Java内存模型
date: 2019-11-15
tags: JVM
---

### **硬件的效率与一致性**

大多数的运算任务都不可能只靠处理器“计算”就能完成，处理器要与内存交互，如读取运算数据、存储运算结果等，这个I/O操作很难消除（无法仅靠寄存器来完成所有运算任务），
由于计算机的存储设备与处理器的运算速度有几个数量级的差距，所以加入一层读写速度尽可能接近处理器运算速度的高速缓存（Cache）来作为内存与处理器之间的缓冲：
将运算需要使用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就无须等待缓慢的内存读写了

处理器、高速缓存、主内存间的交互关系如下图

![](/images/posts/JMM/a1.png)

每一个CPU都有一个自己的Cache，而它们又共享同一主内存（Main Memory），所以当多个CPU的运算任务涉及到同一块主内存区域时，就会出现缓存一致性的问题，

### **Java内存模型**

**Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这3个特征来建立的**

Java虚拟机规范中定义了Java内存模型（Java Memory Model，JMM），用于屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的并发效果，
JMM规范了Java虚拟机与计算机内存是如何协同工作的：规定了一个线程如何和何时可以看到由其他线程修改过后的共享变量的值，以及在必须时如何同步的访问共享变量
（包括了实例字段、静态字段和构成数组对象的元素，但不包括局部变量与方法参数）

Java内存模型规定了所有的变量都存储在主内存（Main Memory）中，每条线程还有自己的工作内存（Working Memory），线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝，
线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的变量，不同的线程之间也无法直接访问对方工作内存中的变量，
线程间变量值的传递均需要通过主内存来完成

#### **原子性**

JMM中定义了8中操作：虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的

+ lock（锁定）：作用于主内存的变量，它把一个变量标识为一条线程独占的状态

+ unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定

+ read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用

+ load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中

+ use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作

+ assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作

+ store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的write操作使用

+ write（写入）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中

如果要把一个变量从主内存复制到工作内存，那就要**顺序**地执行read和load操作，如果要把变量从工作内存同步回主内存，就要**顺序**地执行store和write操作  

**这两个操作必须按顺序执行，而没有保证是连续执行，操作指令之间可插入其他指令**

#### **可见性**

提供volatile关键字来保证可见性。

当一个共享变量被volatile修饰时，它会保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值。

#### **有序性**

在Java内存模型中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

同样可以通过volatile关键字来保证一定的"有序性"，

Java内存模型具备一些先天的"有序性"，不需要通过任何手段就能够得到保证的有序性，这个通常也称为 happens-before(先行发生) 原则。
如果两个操作的执行次序无法从happens-before原则推导出来，那么它们就不能保证它们的有序性，虚拟机可以随意地对它们进行重排序

+ 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作

+ 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作

+ volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作

+ 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C

+ 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作

+ 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生

+ 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行

+ 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始

这8条规则中，前4条规则是比较重要的，后4条规则都是显而易见的。

第一条规则：一段程序代码的执行在单个线程中看起来是有序的，顺序是按照代码顺序执行的，这个规则是用来保证程序在单线程中执行结果的正确性，但无法保证程序在多线程中执行的正确性。

第二条规则：论在单线程中还是多线程中，同一个锁如果出于被锁定的状态，那么必须先对锁进行了释放操作，后面才能继续进行lock操作。

第三条规则：如果一个线程先去写一个变量，然后一个线程去进行读取，那么写入操作肯定会先行发生于读操作。

第四条规则：体现happens-before原则具备传递性。

以上三个特征都可以通过synchronized和Lock来实现，synchronized和Lock能保证同一时刻只有一个线程获取锁然后执行同步代码，
并且在释放锁之前会将对变量的修改刷新到主存当中。

### **volatile关键字**

#### **volatile关键字的两层语义**

一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：

+ 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。

+ 禁止进行指令重排序。

#### **可见性**

当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量不能做到这一点，普通变量的值在线程间传递均需要通过主内存来完成，
线程A经过```assign, store, write```操作之后，把修改的变量写入主内存，线程B要经过```read, load```操作之后才能读到，而且什么时候被写入主存是不确定的，
当线程B去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性。

**volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能立刻反应到其他线程之中，但是基于volatile变量的运算在并发下并不一定是安全的**

以自增运算```a++```为例，分为三个操作：

1、读取到工作内存
2、值加1
3、写回主内存

变量a是volatile类型的，在多线程下，步骤1每次都会从主内存中读取，可以保证读取到的值是正确的，但当写回主内存时，可能会有两个线程同时写，
导致最后的值不一致，所以volatile只在**原子操作**下是线程安全的

volatile怎么保证线程间的可见性?

+ 使用volatile关键字会强制将修改的值立即写入主存

+ 使用volatile关键字的话，当线程2进行修改时，会导致线程1的工作内存中缓存变量的缓存行无效（反映到硬件层的话，就是CPU的L1或者L2缓存中对应的缓存行无效）

+ 由于线程1的工作内存中缓存变量的缓存行无效，所以线程1再次读取变量的值时会去主存读取


#### **指令重排序**

使用volatile变量的第二个语义是禁止指令重排序优化，普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，
而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致

使用volatile修饰的变量，相当于加了一个内存屏障（Memory Barrier或Memory Fence，指重排序时不能把后面的指令重排序到内存屏障之前的位置），
只有一个CPU访问内存时，并不需要内存屏障，但如果有两个或更多CPU访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性

对应的CPU指令是```lock```前缀，提供三个功能

+ 确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成

+ 强制将对缓存的修改操作立即写入主存

+ 如果是写操作，它会导致其他CPU中对应的缓存行无效








