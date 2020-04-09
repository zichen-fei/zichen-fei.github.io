---
layout: post
title: 单例模式的几种实现方式
date: 2019-06-15
tags: 设计模式
---

### **1、懒汉模式**
```
public class Singleton {

    private static Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```
+ 时间换空间，一开始不加载资源或者数据，等到使用的时候在加载，不加同步的懒汉式不是线程安全的  

### **2、饿汉模式**
```
public class Singleton {

    private static Singleton instance = new Singleton();

    private Singleton() {

    }

    public static Singleton getInstance() {
        return instance;
    }
} 
```
+ 空间换时间，当类加载的时候就会创建实例，每次调用的时候不需要判断，节省运行时间，是线程安全的  

### **3、静态内部类**
```
public class Singleton {

    private static class SingletonHolder {
        private static Singleton instance = new Singleton();
    }

    private Singleton() {

    }

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
} 
```
+ 优点：外部类加载时不需要加载内部类，内部类不加载不会去初始化instance，所以不占内存。当Sinlgeton加载时，不需要加载SingletonHolder，只有当getInstance()第一次被调用时，虚拟机才会加载SingletonHolder类，初始化instance
不仅能保证线程安全，也能保证单例的唯一性，也延迟了单例的实例化

+ 为什么能保证线程安全？  
JVM类加载：在类初始化阶段，JVM会保证一个类构造器的< clinit >()方法在多线程环境中被正确的加锁、同步，如果多个线程同时去初始化一个类，只会有一个线程去执行这个类的< clinit >方法，
其他线程都需要阻塞等待，知道执行方法完毕，如果在一个类的< clinit >()方法中有耗时很长的操作，就可能造成多个进程阻塞
(执行< clinit >()方法后，其他线程唤醒之后不会再次进入< clinit >()方法。同一个加载器下，一个类型只会初始化一次。)  

### **4、双重锁校验**
```
public class Singleton {

    private static volatile Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        //第一次校验
        if (instance == null) {
            synchronized (Singleton.class) {
                //第二次校验
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

```
+ 两次校验：  
第一次校验：单例模式只需要创建一次实例，因此同步方法里的代码只执行一次，加上判断会提高性能，如果没有第一次校验，每次获取都需要去竞争锁  
第二次校验：如果没有第二次校验，假设线程t1执行了第一次校验后，判断为null，这时t2也获取了CPU执行权，也执行了第一次校验，判断也为null。接下来t2获得锁，创建实例。这时t1又获得CPU执行权，由于之前已经进行了第一次校验，结果为null（不会再次判断），获得锁后，直接创建实例。结果就会导致创建多个实例。所以需要在同步代码里面进行第二次校验，如果实例为空，则进行创建

+ 在 instance = new Singleton(); 过程中分为三步  
1、在堆中开辟内存空间  
2、实例化类中的各个参数  
3、把对象指向堆  
由于JVM的乱序执行功能，可能在2还没执行时就执行了3，如果有另一个线程执行，instance就不为空，直接使用，就会出现异常，所以instance要用volatile，确保每次都会从主内存中读取