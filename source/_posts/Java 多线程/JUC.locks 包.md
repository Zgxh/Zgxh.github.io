---
title: JUC.locks 统览
date: 2020-11-06 16:25:25
tags: 多线程与并发
categories: 多线程与并发
---

# JUC.locks 

## 1. AQS 抽象类

JUC.locks 包下有三个抽象类:
- AQS
- AQLS
- AOS

AQS 是 `ReentrantLock` 等锁的基础，具体看 AQS 部分。

## 2. Condition 接口

`Condition` 条件对象提供了 `await()`，`signal()` 等方法，提供了类似于 `Object` 类的监视器方法，它与 `Lock` 配合可以实现 等待/通知 模式。

`Condition` 的实例是通过 `Lock` 接口实现类的 `newCondition()` 方法来创建的。

### 2.1 等待队列

AQS 中的 `ConditionObject` 类是 `Condition` 接口的实现类。

```java
public class ConditionObject implements Condition, java.io.Serializable {
    // 头结点
    private transient Node firstWaiter;
    // 尾结点
    private transient Node lastWaiter;
}
```

![](http://note.youdao.com/yws/public/resource/03e442b43c9103c3bcce438252939ad9/xmlnote/B202E4EC01A84A7C8548FDA4676780B8/16953)

一个 `Condition` 实例对象对应一个**单向等待队列**，队列的结点是 `AQS.Node` 类，每个结点对应一个线程。
- 如果线程调用 `condition.await()` 方法，则会以当前线程构造一个 `Node` 结点，然后加入该等待队列尾部，并释放锁，从而使该线程进入等待状态。
    - `Condition` 对象持有队列的尾指针，新加入一个对象只需要连接在当前尾结点的后面即可。

### 2.2 await() 与 notify() 的过程

![](http://note.youdao.com/yws/public/resource/03e442b43c9103c3bcce438252939ad9/xmlnote/E0B31346E36F462188FCADE31B9A336C/16888)

在一个 `Lock` 对象中，同时拥有**一个同步队列**和**多个等待队列**。如 `ReentrantLock`。

从方法调用者的角度来看，一个线程如果调用了 `Condition.await()` 方法，则意味着该线程一定正持有锁，所以该线程结点位于同步队列的头结点。

#### 2.2.1 等待

当调用某个 `Condition` 实例的 `await()` 方法后，会把当前线程结点从**同步队列的头结点**移动到该 `Condition` 实例对应的**等待队列的尾结点**，然后释放已获得的锁，并唤醒同步队列中的下一个节点，然后当前线程会进入等待状态。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 把当前的线程加入等待队列尾部
    Node node = addConditionWaiter();
    // 释放当前线程持有的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

#### 2.2.2 通知

调用某个 `Condition` 实例对应的 `signal()` 之类的方法，会唤醒当前 `Condition` 实例对应的等待队列中已等待时间最长的线程（即等待队列中的头结点），会将结点移入同步队列的尾部。

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 拿到等待队列的头结点
    Node first = firstWaiter;
    if (first != null)
        // 从等待队列中删除该结点，并放入同步队列中
        doSignal(first);
}
```

[Condition 等待通知的使用与对应的同步队列与等待队列状态分析](https://blog.csdn.net/MakeContral/article/details/78135531)

### 2.2 Object 的监视器与 Condition 对象的对比

![](http://note.youdao.com/yws/public/resource/03e442b43c9103c3bcce438252939ad9/xmlnote/BCC66A93BCBE4CBEB9CB3C673A88ACDE/16933)

AQS 中的 `ConditionObject` 是 `Condition` 接口的实现类。

`Condition` 对象是依赖于 `Lock` 对象的。
- 调用 `Condition` 的方法，首先需要获得 `Condition` 对象所关联的锁。
 
## 3. Lock 接口

`Lock` 接口的几个实现类：`ReentrantLock`，`ReentrantReadWriteLock.ReadLock`，`ReentrantReadWriteLock.WriteLock`

```java
public interface Lock {
    // 加锁
    void lock();
    // 支持中断响应
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    // 释放锁
    void unlock();

    // 创建一个 Condition 实例
    Condition newCondition();
}
```

### 3.1 Lock 获取锁的方法

`Lock` 中获取锁的方法对比：
- `lock()`：获取锁，如果获得了锁，则继续进行；如果没有获得锁，则一直等待。
- `tryLock()`：尝试获取锁，无论是否获取到锁，都会立即返回，不会在没有获得锁时一直等待。如果获取到了锁则返回 true，否则返回 false。
    - `tryLock(time, unit)`：尝试获取锁，并在没有拿到锁的时候尝试等待一段时间，如果在此期间获得了锁，则返回 true，否则返回 false。
- `lockInterruptibly()`：获取锁，但**等待锁的过程中**可以响应中断。一旦线程获取到了锁，则不会被中断。

### 3.2 获取锁的代码

`Lock` 必须显式地加锁和解锁，并在 `finally` 块中释放锁，从而保证锁一定会被释放而防止死锁

```java
Lock lock = new ReentrantLock();
lock.lock();  
try {  
    // 互斥区
finally {  
    lock.unlock(); // 锁必须在 finally 块中释放 
```

### 3.3 ReentrantLock

详见 ReentrantLock 章节

## 4. ReadWriteLock 接口

`ReadWriteLock` 分别维护了读锁和写锁。该接口被 `ReentrantReadWriteLock` 实现。

```java
public interface ReadWriteLock {
    // 读锁
    Lock readLock();

    // 写锁
    Lock writeLock();
}
```

### 4.1 ReentrantReadWriteLock

## 5. StampedLock

Java 8 中加入的新锁，性能比之前的锁更高。












