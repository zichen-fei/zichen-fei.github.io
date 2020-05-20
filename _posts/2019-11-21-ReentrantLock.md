---
layout: post
title: 可重入锁
date: 2019-11-21
tags: Java
---

指的是可重复可递归调用的锁，在外层使用锁之后，在内层仍然可以使用，并且不发生死锁（前提得是同一个对象或者class），这样的锁就叫做可重入锁。
**ReentrantLock**和**synchronized**都是可重入锁，可重入锁的最大作用就是避免死锁

### **简单锁**

```
public class Counter{
    private Lock lock = new Lock();
    private int count = 0;
    public int inc(){
        lock.lock();
        this.count++;
        lock.unlock();
        return count;
    }
}
```

上面的程序中，由于```this.count++```这一操作分多步执行，在多线程环境中可能出现结果不符合预期的情况，这段代码称之为**临界区**，所以需要使用lock来保证其原子性

Lock实现

```
public class Lock{
    private boolean isLocked = false;
    public synchronized void lock() throws InterruptedException{
        while(isLocked){    //不用if，而用while，是为了防止假唤醒
            wait();
        }
        isLocked = true;
    }
    public synchronized void unlock(){
        isLocked = false;
        notify();
    }
}
```

当isLocked为true时，调用lock()的线程在wait()阻塞，为防止该线程虚假唤醒，程序会重新去检查isLocked条件。 如果isLocked为false，
当前线程会退出while(isLocked)循环，并将isLocked设回true，让其它正在调用lock()方法的线程能够在Lock实例上加锁。
当线程完成了临界区中的代码，就会调用unlock()。执行unlock()会重新将isLocked设置为false，并且唤醒 其中一个 处于等待状态的线程。

### **不可重入锁**

```
public class UnReentrant{
    Lock lock = new Lock();
    public void outer(){
        lock.lock();
        inner();
        lock.unlock();
    }
    public void inner(){
        lock.lock();
        //do something
        lock.unlock();
    }
}
```

outer中调用了inner，outer先锁住了lock，这样inner再获取lock就会产生死锁，调用outer的线程已经获取了lock锁，
但是不能在inner中重复利用已经获取的锁资源，这种锁即称之为**不可重入**。而实际上同一个线程不必每次都去释放锁再来获取锁，这样的调度切换是很耗资源的

**可重入就意味着：线程可以进入任何一个它已经拥有的锁所同步着的代码块。**

### **可重入锁**

```
public class Lock{
    boolean isLocked = false;
    Thread  lockedBy = null;
    int lockedCount = 0;
    public synchronized void lock()
        throws InterruptedException{
        Thread callingThread = Thread.currentThread();
        while(isLocked && lockedBy != callingThread){
            wait();
        }
        isLocked = true;
        lockedCount++;
        lockedBy = callingThread;
  }
    public synchronized void unlock(){
        if(Thread.curentThread() == this.lockedBy){
            lockedCount--;
            if(lockedCount == 0){
                isLocked = false;
                notify();
            }
        }
    }
}
```

+ lockBy：保存已经获得锁实例的线程，在lock()判断调用lock的线程是否已经获得当前锁实例，如果已经获得锁，则直接跳过while，无需等待

+ lockCount：记录同一个线程重复对一个锁对象加锁的次数。否则，一次unlock就会解除所有锁，即使这个锁实例已经加锁多次了，
即重入了多少次，就要释放多少次，不然也会导入锁不被释放




