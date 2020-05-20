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

![](/images/posts/threadLocal/a2.png)

### **为什么会出现内存泄露**

ThreadLocalMap内部Entry中key使用的是对ThreadLocal对象（强引用）的弱引用，如果是强引用，那么即使其他地方没有对ThreadLocal对象的引用，
ThreadLocalMap中的ThreadLocal对象还是不会被回收，而如果是弱引用则这时候ThreadLocal引用是会被回收掉的

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
因此GC是可以回收这部分空间的，也就是key是可以回收的。但是value却存在一条从Current Thread过来的强引用链。
因此只有当Current Thread销毁时，value才能得到释放

ThreadLocalMap提供了```set,get,remove```方法在一些时机下会对这些Entry项进行清理，但是这是不及时的，也不是每次都会执行的，
所以一些情况下还是会发生内存泄露，所以在使用完毕后需要及时调用remove方法避免内存泄露。

### **处理Key为null的Entry**

ThreadLocalMap是**WeakHashMap**，提供了```expungeStaleEntry```方法循环遍历map中的key，当key的值为null是，把value也赋值为null，
在下一次GC时回收

![](/images/posts/threadLocal/a4.png)

在```get,set,remove```时都会调用该方法

![](/images/posts/threadLocal/a6.png)

![](/images/posts/threadLocal/a5.png)

![](/images/posts/threadLocal/a7.png)

