---
title: ReentrantLock 与 synchronized
date: 2020-11-06 16:25:25
tags: 多线程与并发
categories: 多线程与并发
---

# ReentrantLock 与 synchronized

## 1. synchronized 与 ReentrantLock 的对比

`synchronized` 与 `ReentrantLock` 都具备线程重入性，但 `synchronized` 由 JVM 实现，`ReentrantLock` 由 JDK 实现。

`synchronized` 在同步代码中发生异常时，会自动释放锁，因此不会导致死锁；而 `Lock` 则必须主动在 `finally` 代码块中显式释放锁。

但 `ReentrantLock` 相比于 `synchronized` 实现了一些更高级的功能：
1. **等待时可中断**：`ReentrantLock` 的 lockInterruptibly() 方法可以使线程在阻塞时响应中断。而 `synchronized` 产生的互斥锁则会一直阻塞，是不能被中断的。
2. **可实现公平锁**：`synchronized` 中的锁一定是非公平锁，`ReentrantLock` 默认情况下也是非公平锁，但可以通过构造方法 `ReentrantLock(true)` 来指明使用公平锁。
3. **锁可以绑定多个条件**：`ReentrantLock` 对象可以同时绑定多个 `Condition` 实例来构造多个等待队列，而在 `synchronized` 中，锁对象的 `wait()` 和 `notify()` 或 `notifyAll()` 方法只可以实现一个隐含条件，但如果要和多于一个的条件关联时，需要额外添加锁，而 `ReentrantLock` 则只需多次调用 `newCondition()` 来创建多个条件对象即可。而且还可以通过绑定 `Condition` 对象来判断当前线程通知的是哪些线程。

![](http://note.youdao.com/yws/public/resource/03e442b43c9103c3bcce438252939ad9/xmlnote/3601C68F630C44439CBB2310D9A766C6/14012)

ReentrantLock 中 state 是锁的重入次数。

## 2. ReentrantLock 公平锁 与 非公平锁

- **公平锁**：按照申请锁的顺序来获得锁
- **非公平锁**：不按照申请锁的顺序来获得锁，存在锁的抢占

`ReentrantLock` 默认是 **非公平锁**。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```

公平锁与非公平锁在加锁的时候有区别，在释放锁的时候没有区别。

### 2.1 加锁

#### 2.1.1 非公平锁加锁 NonfairSync （独占模式下）

非公平锁在新来线程后，首先尝试 CAS 直接获取锁（抢占），如果获取不到，才排队。
- 抢占成功的情况发生在：线程 A 刚好释放了锁，这时线程 B 刚好来了，此时 B 可能直接抢占到锁，同步队列中的线程结点只好继续等待。

```java
final void lock() {
    // 使用 CAS 操作尝试将 state 状态变量设置为 1
    if (compareAndSetState(0, 1))
        // 设置成功后，表示当前线程获取到了锁，然后将独占锁的拥有者设置为当前线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 进行后续处理，会涉及到重入性、创建 Node 节点加入队列等
        acquire(1);
}
```

`AQS` 抽象类中的 `acquire(int)`:
1. 使用 `tryAcquire(int)` 方法，尝试直接获取同步锁，获取成功则设置锁的占有者为当前线程，则 `&&` 短路返回。
2. 获取失败，`addWaiter()` 方法创建线程结点加入到同步队列中，并执行 `acquireQueued()` 方法，死循环遍历同步队列，直到创建的结点对应的线程获取到锁。
3. 最后一直成功获取到锁，若需要中断，则执行中断。

```java
public final void acquire(int arg) {
    /**
     * acquireQueued() 返回的 true 代表当前线程存在中断标志，这样在获取到同步锁后，对当前线程进行中断。
     */
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`NonfairSync` 中重写了 `AQS` 中的 `tryAcquire(int)` 方法：

```java
final boolean nonfairTryAcquire(int acquires) {
    // 获取当前线程
    final Thread current = Thread.currentThread();
    // 获取当前 state 同步状态变量值
    int c = getState();
    // 如果状态值为 0，则没有线程占用该锁，尝试获取该锁
    if (c == 0) {
        // 使用 CAS 算法尝试将state同步状态变量设置为 1，获取同步锁
        if (compareAndSetState(0, acquires)) {
            // 若获取成功，则将独占锁的拥有者设置为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果拥有独占锁的的线程是当前线程的话,表示当前线程需要重复获取锁（重入锁）
    else if (current == getExclusiveOwnerThread()) {
        // 同步状态 state 变量值加 1
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 更新 state 同步状态变量值
        setState(nextc);
        return true;
    }
    // 其他线程占有锁，直接返回false
    return false;
}
```

`AQS` 中的 `addWaiter(mode)` 方法，创建一个指定模式的线程结点，并插入同步队列的尾部:

```java
private Node addWaiter(Node mode) {
    // 新建指定模式的结点
    Node node = new Node(Thread.currentThread(), mode); // 若 mode 参数是独占模式，则值为 null
    // 把 node 连接到队列的尾部
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        // 使用 CAS 算法将当前节点设置为尾节点
        if (compareAndSetTail(pred, node)) {
            // 尾节点设置成功，将pre旧尾节点的后继结点指向新尾节点node
            pred.next = node;
            return node;
        }
    }
    // 如果尾节点为null，表示同步队列是空队列，将 node 直接加入队列
    enq(node);
    
    return node;
}
```

`AQS` 中的 `acquireQueued()`，进入死循环，一直尝试获取锁，直到获取成功才返回，返回值是一个中断标志位：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true; // 标志获取锁是否成功，决定着 cancelAcquire() 方法是否执行
    try {
        boolean interrupted = false; // 标志是否中断
        // 进入死循环，循环尝试获取锁，直到成功获取
        for (;;) { 
            // 获取当前节点的前驱结点
            final Node p = node.predecessor();
            /**
             * 如果当前节点的前驱结点是同步队列的头结点，则头结点已经获得了锁，则有两种情况：
             * 1、前驱结点的锁还没释放
             * 2、前驱结点的锁已经释放了
             *
             * 然后使用tryAcquire()方法去尝试获取同步锁，如果前驱结点已经释放了锁，那么就会获取成功，
             * 否则同步锁获取失败，继续循环
             */
            if (p == head && tryAcquire(arg)) {
                // 获取锁成功，则将当前节点设置为同步队列的头结点
                setHead(node);
                // 然后将当前节点的前驱结点的后继结点置为null，帮助进行垃圾回收
                p.next = null; // help GC
                failed = false;
                // 返回中断的标志
                return interrupted;
            }
            /**
             * 1. shouldParkAfterFailedAcquire() 如果前驱结点 p 的状态为 SIGNAL 的话，则返回 true
             * 2. parkAndCheckInterrupt() 会使当前线程进入 waiting 状态，并查看当前线程是否被中断
             * interrupted() 同时会将中断标志清除。
             */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 中断标志置为true
                interrupted = true;
        }
    } finally {
        /**
         * 如果循环中出现异常，并且 failed=false 没有执行的话,cancelAcquire() 会将当前线程
         * 的状态置为 node.CANCELLED 已取消状态，并且将当前节点 node 移出同步队列。
         */
        if (failed)
            cancelAcquire(node);
    }
}
```

#### 2.1.2 公平锁的加锁 fairSync 

公平锁加锁时，不会发生抢占，先判断同步队列是不是空，如果为空则获取锁；如果其他线程正在占有锁，则直接排队。

```java
final void lock() {
    acquire(1);
}
```

`acquire(int)` 方法与非公平锁是一样的，这个方法是 `AQS` 提供的。

```java
public final void acquire(int arg) {
    /**
     * acquireQueued() 返回的 true 代表当前线程存在中断标志，这样在获取到同步锁后，对当前线程进行中断。
     */
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

但 `fairSync` 类对 `tryAcquire()` 进行了重写：
1. 先判断当前同步资源是否有线程占用
2. 如果没有线程持有锁，则判断当前同步队列中是否有线程正在等待获取锁，
3. 如果同步队列为空，则尝试通过 CAS 获取锁，并在成功获取锁后返回
4. 如果当前同步资源被当前线程占有，则锁可以被重入，修改锁资源状态 `state` 即可
5. 如果不能获取到锁，则返回 false。

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 公平锁与非公平锁的区别在这：先在同步队列中排队，再尝试获取锁
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 2.2 锁的释放

`unlock()` 函数用来释放锁。
- 因为锁可重入，所以释放当前一层的锁。

```java
public void unlock() {
    // 释放锁时，需要将state同步状态变量值进行减 1，传入参数 1
    sync.release(1);
}
```

它使用了 `release(int)` 方法，该方法由父类 `AQS` 提供：
1. 尝试释放锁
2. 若释放锁成功则尝试唤醒同步队列中的下一个可用结点

```java
public final boolean release(int arg) {
    // tryRelease() 尝试释放锁，对 state 变量的值进行修改
    if (tryRelease(arg)) {
        Node h = head;
        // 头结点不为空并且头结点的 waitStatus 不是初始化结点,则唤醒队列下一个可用结点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    
    return false;
}
```

`tryRelease()` 方法会对 `state` 变量进行更改，因为锁是可重入的，只有当 `state == 0` 后，该线程才真正地释放了锁。

```java
protected final boolean tryRelease(int releases) {
    // 当前state状态值进行减少
    int c = getState() - releases;
    // 判断权限：确定当前独占锁的占有者是当前线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // 当 state 变成 0 时，才表示该同步资源无线程正在占用
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 更新 state 同步状态值，对 volatile 变量的更改会立即被其他线程可见
    setState(c);
    
    return free;
}
```

`unparkSuccessor(Node)` 方法用于唤醒 同步队列中第一个可用的正在阻塞的线程：
- 寻找目标线程时是**从队尾向队首找**
    - 这是因为结点的入队不是一个原子操作，在 `addWaiter()` 方法中，连接一个结点需要修改 prev 和 next 指针，首先进行的是 prev 指针的修改，若此时执行了 `unparkSuccessor(Node)` 方法，则可能不会检测到 next 指针的存在，即可能忽略了这个队尾结点。所以要从后往前找。

```java
private void unparkSuccessor(Node node) {
    // 获取头结点 waitStatus
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    // 在同步队列中找下一个节点
    Node s = node.next;
    // 如果下个节点为空或不可用，就查找队列中第一个可用的结点
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从队尾到队首查找第一个 waitStatus < 0 的线程结点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果找到了有效的线程结点，则将对应的处于阻塞状态的线程唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

被唤醒的线程在 acquireQueued() 中继续执行，设置当前结点为同步队列的头结点，即获取到了锁。

## 3. synchronized 关键字

`synchronized` 同步锁，利用锁机制来实现同步。
- **互斥性**：同一时间只允许一个线程持有某个对象锁
- **可见性**：锁释放前，对共享变量的修改会从本地内存刷新到主内存，其他线程获取锁时，会强制从主内存中获取最新的值。因此一个线程对共享变量的修改对其他线程可见。
- **可重入性**：`synchronized` 锁天然可重入

### 3.1 对象锁 与 类锁

- **对象锁**：对**实例对象**加锁，作用范围是该实例对象的所有的同步方法和同步代码块。
    - 当某个同步方法被一个线程占用时，其他该实例对象的所有其他同步方法都无法进入。
- **类锁**：对 `Class` 对象加锁，作用对象是**所有该类的实例**。

设计锁的目的是让多个线程共享同一把锁，于是：

- **对象锁**：当多个线程贡献一个对象实例时，可以通过实例锁来加锁实现线程互斥；
- **类锁**：当多个线程各自拥有自己的对象实例时，要通过类锁来实现线程互斥。

但是，对象实例锁与类锁不相互影响。因为对象锁锁的是对象实例，类锁锁的是 `Class` 对象，二者不是同一个对象，所以不占用同一把锁。

锁只影响需要锁的同步方法和同步代码块。当对象锁或类锁被占用时，对应的其他非同步方法不会受影响。

#### 3.1.1 锁的释放

锁会在同步代码运行结束时，或者同步方法抛异常时，释放锁。因此，抛异常时不会出现死锁。

### 3.2 synchronized 作用体

![](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/12837D509E594F4893A2432DCA9A887A/12906)

### 3.3 synchronized 的注意事项

- `synchronized` 关键字不能被继承
    - 子类覆盖父类方法时，必须显式指明 `synchronized`
- 接口的方法不能被 `synchronized` 修饰
- 构造器不能被 `synchronized` 修饰，但可以在同步代码块中执行同步操作

### 3.4 synchronized 的实现原理与优化

#### 3.4.1 加锁的原理：重量级锁的底层实现

每个对象关联一个 Monitor 对象，当 Monitor 被某个线程持有时，它便处于锁定状态。

Monitor 在虚拟机中由 `ObjectMonitor` 对象实现。

![ObjectMonitor 运行机制](http://note.youdao.com/yws/public/resource/03e442b43c9103c3bcce438252939ad9/xmlnote/B4E904274E46400CB12569CD1300BC2F/19711)

#### ObjectMonitor

实现 Object 类的 `wait()`, `notify()` 和 对 `synchronized` 的支持。
- 如果调用了 `wait()`, `notify()`，则首先将锁膨胀为重量级锁

```java
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; // 记录该线程获取锁的次数
    _waiters      = 0,
    _recursions   = 0; // 记录锁的重入次数
    _object       = NULL;
    _owner        = NULL; // 指向目前正持有锁的线程
    _WaitSet      = NULL; // 处于 WAITING 状态的线程，会被加入到 _WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; // 最新参与竞争的线程队列
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 处于等待锁而被阻塞 BLOCKED 的线程，会被加入到该队列
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
}
```

#### 多线程争夺重量级锁的过程

`ObjectMonitor` 通过队列来管理多线程竞争，队列中存放的是 `ObjectWaiter` 对象，维护了三个队列 _cxq，_ENtryLIst，_WaitSet。

1. 当某个线程获取到对象的 `Monitor` 之后，把 `_owner` 指向当前获取到锁的线程，计数器 `_count` 加一，表示当前对象锁被一个线程获取。
2. 如果一个线程要获取锁，首先会自旋尝试获取锁，获取到则直接返回；获取不到则被封装成 `ObjectWaiter` 对象，然后进入竞争队列 _cxq，并挂起当前线程。
3. 当有线程释放锁时，会设置 _owner=null，然后唤醒阻塞的线程，此时会根据具体的唤醒策略，来决定是从 _cxq 中取出线程，或者是把 _cxq 中的线程放入 _entryList 中，然后唤醒 _entryList 中的线程。
4. 当线程调用 `wait()` 方法时，释放持有的 `Monitor` 对象，然后 `_owner` 设为 `null`，`_count` 减一，该线程进入 `_WaitSet` 中，此时该线程处于 WAITING 状态，直到有线程调用 `notify()` 来唤醒它。
5. 若锁重入时， `_recursions++` 来记录锁重入的次数。

![](https://note.youdao.com/yws/public/resource/03e442b43c9103c3bcce438252939ad9/xmlnote/6E901A23E67F46A7A05E1AE987E6BE08/25288)

##### 为什么 synchronized 锁不公平

1. 等待锁的线程在进入 _cxq 队列前，首先会尝试自旋获取锁，如果获取到了就直接拿到锁，获取不到再进入 _cxq，这对其他正在等待锁的线程是不公平的。
2. 当_cxq中等待锁的线程向 _entryList 中迁移时，在某种策略下，会导致后进入 _cxq 的线程先获得锁。

#### 3.4.2 从字节码层面看 synchronized

##### synchronized 作用于代码块

对于 `synchronized` 锁定的代码块，字节码层面通过 `monitorenter`，`moinitorexit` 来实现。

字节码层面通过**一次** `monitorenter` 和 **两次** `monitorexit` 来实现对 Monitor 的互斥访问。
- 两次 `monitorexit` 是为了处理异常情况，保证锁的退出。
- 获得 `monitor` 的线程可以**重入**这个锁，每重入一次执行一次 `monitorenter`，每一个 `monitorenter` 都对应 2 个 `monitorexit`。

##### synchronized 作用于方法

synchronized 作为方法修饰符时，没有利用 `monitorenter` 和 `moinitorexit`。而是通过标记方法为 `ACC_SYNCHRONIZED` 来声明该方法是个同步方法。

#### 3.4.3 synchronized 的优化

操作系统想要实现线程之间的切换，必须从用户态转换到核心态，转换的成本很高，因此 `synchronized` 在**重量级锁**状态下的效率很低。

##### 1. 锁升级

- 在 JDK 1.6 之前，`synchronized` 默认为重量级锁，即利用操作系统的互斥锁来实现。
- JDK 1.6 时，对 `synchronized` 进行了优化。锁会经历从偏向锁，到轻量级锁，再到重量级锁，的锁升级过程。

锁只会升级，不会降级。

##### 2. 锁消除

当虚拟机执行 JIT 编译时，扫描运行上下文，自动去除不存在锁竞争的同步语句。
- 比如，在方法中，局部变量只会被单个线程所访问的，且生命周期随着方法运行结束而消亡，所以局部变量 `StringBuffer` 调用 `append()` 方法时，会执行同步消除。

### 3.5 synchronized 与 中断

执行中断方法 `thread.interupt()` 并不会影响同步代码块的正常执行，除非在代码中显式地去判断中断状态并处理中断。

[synchronized 底层分析--这个人写的巨他妈好](https://github.com/farmerjohngit/myblog/issues/15)


