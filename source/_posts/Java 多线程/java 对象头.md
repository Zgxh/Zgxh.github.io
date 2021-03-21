---
title: Java 对象头
date: 2020-11-06 16:25:25
tags: 多线程与并发
categories: 多线程与并发
---

# Java 对象头

## 1. Java 对象头的组成

1. `Mark Word`
2. 指向类元信息的指针 `Klass Pointer`
3. 数组的长度

![对象头信息](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/4098CB7903B94C2289BC15C32FFE94BF/8890)

### 1.1 Mark Word

`Mark Word` 在 32 位 JVM 中的长度是 32 bit，在 64 位 JVM 中长度是 64bit。以 32 位为例：
- 使用 1 bit 来指示 是否为偏向锁
- 使用 2 bit 来标志 锁的状态

`Mark Word` 用来存放对象信息或锁信息。`Mark Word` 被设计成可以复用的形式，
- 当对象是**无锁态**时，`Mark Word` 记录对象的 hashCode，锁标志位是 01，是否为偏向锁位为 0；
- 当对象锁为**偏向锁**时，锁标志位依然是 01，是否为偏向锁位为 1，`Mark Word` 的前 23 位标志占有偏向锁的线程 id；
- 当对象锁为**轻量级锁**时，锁标志位是 00，前 30 位指向栈帧中锁记录的指针
- 当对象锁位**重量级锁**时，锁标志位是 10，前 30 位为指向重量级锁的指针

![](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/B7669EB5178041248F525FB44AC284E2/13025)

### 1.2 Klass Pointer

对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。


## References

[深入理解Java并发之synchronized实现原理](https://blog.csdn.net/javazejian/article/details/72828483)