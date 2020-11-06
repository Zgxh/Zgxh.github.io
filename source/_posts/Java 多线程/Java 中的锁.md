---
title: Java 锁分类
date: 2020-11-06 16:25:25
tags: 多线程与并发
categories: 多线程与并发
---

# Java 中的锁

![](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/0C0EE024968D49839A77182E8FE888E5/12923)

Java 中的并发包： `java.util.concurrent` , 又叫 **JUC**。

## 1. 乐观锁 与 悲观锁

![](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/45FBCEC3BF6643D9A65FAF87FBF23D2A/12618)

### 1.1 乐观锁

乐观锁总是假设对共享资源的访问没有冲突，线程可以不停地执行，无需加锁，也无需等待。

一旦多个线程发生冲突，则是使用 CAS 来解决线程安全性问题。

乐观锁默认不加锁，所以不会出现死锁。

乐观锁适用于 “多读少写” 的情况，避免冲突时频繁加锁，影响性能。

### 1.2 悲观锁

悲观锁总是假设访问共享资源会发生冲突，所以每次都会加锁，以保证临界区同时只有一个线程在执行。
- `synchronized` 和 `Lock` 的实现类 都是悲观锁。

悲观锁适用于 “多写少读” 的情况，避免频繁的失败和重试影响性能。

### 1.3 Java 实现乐观锁与悲观锁

**悲观锁**的调用方式：
- 使用 `synchronized`
- 使用 `ReentrantLock`

```java
// synchronized
public synchronized void testMethod() {
	// 操作同步资源
}

// ReentrantLock
private ReentrantLock lock = new ReentrantLock(); // 需要保证多个线程使用的是同一个锁
public void modifyPublicResources() {
	lock.lock();
	// 操作同步资源
	lock.unlock();
}
```

**乐观锁**的调用方式：
- 使用 CAS 操作：Java 中的原子类

```java
private AtomicInteger atomicInteger = new AtomicInteger();  // 需要保证多个线程使用的是同一个AtomicInteger
atomicInteger.incrementAndGet(); //执行自增 1
```

## 2. 自旋锁 与 适应性自旋锁

### 2.1 自旋锁

![](http://note.youdao.com/yws/public/resource/bfce0e3d92cf4516094fe684a07f9b39/xmlnote/E1A0ECBB49F34F7EA606228899DAA7F8/12961)

#### 2.1.1 定义

当临界资源被其他线程占用时，先让当前线程自旋，如果自旋结束前临界资源被释放，则避免了线程阻塞来直接获得同步资源，避免了线程切换的开销。

#### 2.1.2 适用情形

在某些情况下，同步资源的**锁定时间很短**，若线程频繁阻塞或唤醒会消耗更多资源，使用自旋锁来避免线程的阻塞和唤醒。

反之，如果线程持有锁的时间很长，则忙等待会白白浪费处理器资源。
- 使用适应性自旋锁来优化自旋锁。

自旋锁不是公平的，等待时间最长的线程不会优先获得锁，会造成线

#### 2.1.3 自旋锁的原理

利用 CAS 操作，如果失败则一直循环来执行自旋，直到成功。

JVM 开启关闭自旋锁：
```
-XX:+UseSpinning
```

[面试必备之深入理解自旋锁](https://blog.csdn.net/weixin_33995481/article/details/88771814?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

### 2.2 自适应自旋锁

JDK 6 中默认开启自旋锁，并且引入了**自适应自旋锁**。
- 自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。
    - 如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。
    - 如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

## 3. 无锁，偏向锁，轻量级锁，重量级锁

详情看 [9. 对象锁] 的 【9.2 锁分类】 部分。

## 4. 公平锁 与 非公平锁

### 4.1 公平锁

公平锁是指多个线程按照**申请锁的顺序**来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。

公平锁的**优点**是等待锁的线程不会饿死。

**缺点**是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU 唤醒阻塞线程的开销比非公平锁大。

### 4.2 非公平锁

非公平锁是多个线程加锁时**直接尝试获取锁，获取不到才会到等待队列的队尾等待**。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。

非公平锁的**优点**是可以减少唤起线程的开销，整体的**吞吐效率高**，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。

**缺点**是处于等待队列中的线程可能会发生线程饥饿。

### 4.3 Java 中的公平锁与非公平锁

`ReentrantLock` 可以实现公平锁与非公平锁。

`ReentrantLock` 里面有一个内部类 `Sync`  继承 AQS（AbstractQueuedSynchronizer），添加锁和释放锁的大部分操作实际上都是在 `Sync` 中实现的。
- 它有公平锁 `FairSync` 和非公平锁 `NonfairSync` 两个子类。
- `ReentrantLock` 默认使用非公平锁，也可以通过构造器来显示的指定使用公平锁。

公平锁加锁，先判断当前线程是否位于等待队列的第一个位置：
```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && // 判断当前线程是否位于等待队列的第一个
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

非公平锁加锁，直接先抢夺锁：
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

## 5. 可重入锁 VS 非可重入锁

可重入锁又名递归锁，是指在**同一个线程**在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者 class），不会因为之前已经获取过锁还没释放而阻塞。
- Java 中 `ReentrantLock` 和 `synchronized` 都是可重入锁
- 可重入锁的优点是可一定程度避免死锁。

### 5.1 非可重入锁导致死锁

```java
public class Widget {
    public synchronized void doSomething() {
        System.out.println("方法1执行...");
        doOthers();
    }

    public synchronized void doOthers() {
        System.out.println("方法2执行...");
    }
}
```

对于不可重入锁来讲，当一个线程执行 doSomething() 时，已经获得了该类实例锁，当 doSomething() 调用 doOthers() 时，因为锁不可重入，需要释放当前的锁。但当前的锁被当前线程所占有，且无法释放，所以会出现死锁。

## 6. 独占锁 VS 共享锁

**独享锁**也叫排他锁，互斥锁 **mutex**，是指该锁一次只能被一个线程所持有。如果线程 T 对数据 A 加上独享锁后，则其他线程不能再对 A 加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。
- JDK 中的 `synchronized` 和 JUC 中 `Lock` 的实现类就是互斥锁。

**共享锁**是指该锁可被多个线程所持有。如果线程 T 对数据 A 加上共享锁后，则其他线程只能对 A 再加共享锁，不能加排它锁。
- 获得共享锁的线程只能读数据，不能修改数据。

### 6.1 Java 中的独占锁与共享锁

`ReentrantReadWriteLock` 可重入的读写锁，分离了**读锁**和**写锁**：其中读锁是共享锁，写锁是独占锁。
- 读锁可以重入，存在读锁时，写锁不能被获取
- 写锁一旦被获取，则其他的读写线程都会被阻塞

读写锁分离，可以在“**读多写少**”的情况下大幅提升效率。

[详细理解 ReentrantReadWriteLock](https://my.oschina.net/adan1/blog/158107)

## 7. JVM 中对锁的优化

### 7.1 锁粗化 Lock Coarsening

锁粗化就是把多个相邻的加锁、解锁操作合并为一次加锁、解锁，从而扩展成一个更大范围的锁。

比如利用线程安全的 `StringBuffer` 类来追加字符时，JVM 会自动把对同一个对象的相邻的加锁-解锁操作来合并，即对第一次 `append()` 进行加锁，对最后一次 `append()` 进行解锁：

```java
public class StringBufferTest {
    StringBuffer stringBuffer = new StringBuffer();

    public void append(){
        // 锁粗化
        stringBuffer.append("a");
        stringBuffer.append("b");
        stringBuffer.append("c");
    }
}
```

### 7.2 锁消除 Lock Elimination

锁消除是指 Java **编译器**会自动删除不必要的加锁操作。例如当编译器分析的值某个变量一定不存在多线程访问时，会自动消除同步操作。

即时编译技术（JIT）可以对代码的编译过程进行优化，在**逃逸分析**过程中，编译器会对代码做优化，包括：
1. 同步省略
2. 标量替换
3. 栈上分配

```
// 开启逃逸分析（从JDK1.7开始默认开启）
-XX:+DoEscapeAnalysis 
// 关闭逃逸分析
-XX:-DoEscapeAnalysis 
```

#### 7.2.1 同步省略

如果发现某个变量不会被多线程访问，即一定是线程安全的，则会执行**锁消除**。

#### 7.2.2 标量替换

标量 Scalar 是指无法被分解为更小粒度的数据，例如原始数据类型 int 等。否则该数据被称为聚合量 Aggregate，例如对象。

**标量替换**是指 JIT 编译过程中，如果经过逃逸分析后，确定一个对象不会被其他线程或其他方法访问，则会将对象的创建替换为成员变量的创建。

举例，我们要创建一个 User 对象，但只在一个线程中调用了 `getUser()` 方法访问了 `name` 和 `age` 属性，则会进行标量替换，不会创建对象 User，只会创建它的两个成员变量。

```java
public class EscapeObject {
    private static void getUser() {
        User user = new User("张三", 18);
        System.out.println("user name is " + user.name + ", age is " + user.age);
    }

    public static void main(String[] args) {
        getUser();
    }
}

class User {
    String name;
    int age;
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

标量替换后的等价代码：

```java
private static void getUser() {
    String name = "张三";
    int age = 18;
    System.out.println("user name is " + name + ", age is " + age);
}

public static void main(String[] args) {
    getUser();
}
```

#### 7.2.3 栈上分配

栈上分配是指变量不创建在堆上，而是创建在栈中，会随着方法的结束而自动销毁。
- 实际上，在 hotspot 虚拟机中，不会实现栈上分配，所有的对象都会创建在堆上。但会采用**标量替换**来代替栈上分配来实现优化。



## Reference 

[美团技术团队-不可不说的锁事](https://tech.meituan.com/2018/11/15/java-lock.html)


