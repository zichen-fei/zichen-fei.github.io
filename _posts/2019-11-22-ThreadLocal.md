---
layout: post
title: ThreadLocal内存泄漏问题
date: 2019-11-21
tags: Java
---

### **实现原理**

![](/images/posts/threadLocal/a1.jpg)

实线代表强引用，虚线代表弱引用，[Java中的引用](../../../2019/11/JavaReference/)

每个Thread维护一个ThreadLocalMap映射表，这个映射表的key是ThreadLocal实例本身，value是真正需要存储的Object。

内部是一个Entry数组,Entry继承自WeakReference，Entry内部的value用来存放Object

比如：
```
ThreadLocal<User> userThreadLocal = new ThreadLocal<>();
userThreadLocal.set(user);
```

实际存储为:

```
//ThreadLocal values pertaining to this thread. This map is maintained by the ThreadLocal class.
ThreadLocal.ThreadLocalMap threadLocals;

ThreadLocalMap map = Thread.currentThread().threadLocals;

map.put(this.userThreadLocal, new User()); 
```

即： ```Map<currentThread, Map<this.Object, ObjectValue>>```

![](/images/posts/threadLocal/a2.png)

### **为什么会出现内存泄露**

ThreadLocal的存储结构为 ```Map<currentThread, Map<this.Object, ObjectValue>>```

以上面代码为例

在使用线程池的情况下，会出现线程复用，Thread会长时间存在，只要线程不销毁，以userThreadLocal对象为Key的Entry(this.userThreadLocal, new User())就无法被回收， 
即使userThreadLocal在其他地方没有使用

而ThreadLocalMap内部Entry做了优化，key使用的是对userThreadLocal对象的弱引用，如果其他地方没有对userThreadLocal对象的引用，
userThreadLocal引用会在下一次GC时被回收掉

比如：

```
C c = new C(b);

b = null;
```

即便b被置为null，但是c仍然持有对b的引用，而且还是强引用，所以GC不会回收b原先所分配的空间！既不能回收利用，又不能使用，这就造成了内存泄露。

```
c = null;
WeakReference w = new WeakReference(b);
```

使用以上两种方法可以解决这个问题，但是如果把ThreadLocal置为null，那么意味着Heap中的ThreadLocal实例不在有强引用指向，只有弱引用存在，
因此GC是可以回收这部分空间的，也就是key是可以回收的。

但是value却是一个强引用，只有当Current Thread销毁时，value才能得到释放

针对key为null的value，ThreadLocalMap提供了```set,get,remove```方法在一些时机下会对这些Entry项进行清理，但是这是不及时的，也不是每次都会执行的，
所以一些情况下还是会发生内存泄露，所以在使用完毕后需要及时调用remove方法避免内存泄露。

### **处理Key为null的Entry**

ThreadLocalMap是**WeakHashMap**，提供了```expungeStaleEntry```方法循环遍历map中的key，当key的值为null是，把value也赋值为null，
在下一次GC时回收

![](/images/posts/threadLocal/a4.png)

在```get,set,remove```时都会调用该方法

![](/images/posts/threadLocal/a6.png)

![](/images/posts/threadLocal/a5.png)

![](/images/posts/threadLocal/a7.png)

