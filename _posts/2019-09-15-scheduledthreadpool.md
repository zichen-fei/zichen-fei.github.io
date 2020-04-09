---
layout: post
title: 多线程问题排查
date: 2019-06-15
tags: 线程
---

线上服务监控报警，有个服务占用CPU一直100%，记录一下排查过程

1、还是第一次遇见这个问题，先想到的是内存占用过高，导致一直Full GC，但是看监控CPU一直保持在100%，内存占用也不高。

服务部署在k8s中，先进入pod
```
kubectl exec -it pre-bit-saas-ras-5f7bbd545f-2wtdx -n saas-pro -- /bin/sh
```
2、执行命令
```
jstat -gcutil pid
```
![](/images/posts/ScheduleThreadPoolExecutor/a5.png)

并没有发生频繁GC，想到项目中用了多线程，怀疑会不会是有死循环

3、查看进程下的线程信息
```
top -H -p pid
```
![](/images/posts/ScheduleThreadPoolExecutor/a1.png)

按CPU占用排序后，发现的确有一个线程CPU的占用率一直在99%

3、将线程对应的pid转换成16进制并获取当前的线程快照
```
echo 'obase=16;pid' | bc

jstack -l pid > jstack.log
```
![](/images/posts/ScheduleThreadPoolExecutor/a4.png)

5、在log中搜索 34bf，找到对应线程信息，定位到 ScheduledThreadPoolExecutor 的 pool 方法，jdk的方法也会有死循环？

![](/images/posts/ScheduleThreadPoolExecutor/a2.png)

6、找到相关的问题，在初始化ScheduledThreadPoolExecutor时，如果指定corePooSize=0，会造成CPU占用100%的问题，

[https://bugs.openjdk.java.net/browse/JDK-8129861]()  
[https://bugs.openjdk.java.net/browse/JDK-8022642]()

为什么会造成这个问题？

7、检查代码，schedulePoolSize用static修饰，**在类加载时会在堆上分配内存空间并赋初始值0，使用@Value注解读取配置文件中的配置，但是在set schedulePoolSize方法前多加了一个static，变成了类的静态方法，只有在显示调用该方法的时候才会进行赋值。**

![](/images/posts/ScheduleThreadPoolExecutor/a3.png)

8、修改代码重新上线后问题解决