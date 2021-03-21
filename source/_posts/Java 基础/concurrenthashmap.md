---
title: ConcurrentHashMap 源码分析
date: 2021-02-21 12:30:43
tags: Java
categories: 开发
---

# ConcurrentHashMap 

## 0. 为什么 HashMap 线程不安全

1. 多线程插入数据导致数据被覆盖
2. 在 JDK 1.7 时，采用的头插法，多线程同时执行 `resize()` 可能造成环形链表，造成死循环
    - JDK 1.8 中代替为尾插法，扩容时会保持链表元素原有的顺序，解决了链表成环的问题

### 0.1 解决方法

- `HashTable`
- `SynchronizedMap`
- `ConcurrentHashMap`

### 0.2 Hashtable 

Hashtable 继承自 Directionary 类。是线程安全的哈希表。

- 底层是 **数组 + 链表** 的实现。
- key 和 value 都不能为 null。
- 其实现并发安全是锁住整个 Hashtable，并发效率低。

### 0.3 SynchronizedMap

把原有的 `Map` 包装成了 `Collections` 的内部类 `SynchronizedMap`，并封装了相应的方法，在执行方法前加了锁，方法的底层实现还是传入的 map 本身。
- 每次执行方法都会对 mutex 对象加锁，效率不高。

```java
// 底层传入的 map
private final Map<K,V> m;     // Backing Map

// 锁对象
final Object      mutex;        // Object on which to synchronize

public int size() {
    synchronized (mutex) {return m.size();}
}
```

`Collections` 中的其他并发包装类：

![](http://note.youdao.com/yws/public/resource/8a56a5bf48dece1894fbd2bd605453b1/xmlnote/36001029F12447F39BE506FF59564912/14738)

## 1. JDK 1.7 版本

![](http://note.youdao.com/yws/public/resource/8a56a5bf48dece1894fbd2bd605453b1/xmlnote/D63790F8493343CF98B97FAB8C8531B4/13935)

1.7 中的 `ConcurrentHashMap` 采用了**分段锁**的策略，先分为多个 `segment`，每个 `segment` 包含一个 `HashEntry` 数组，每个 `HashEntry` 都相当于一个加锁的 `HashMap`。
- 对数据进行读写的时候，只需对当前所在的 `segment` 进行加锁，分段锁提高了并发能力
- 对数据存放位置的查找需要经过两次 hash, 第一次确定在 `segment` 数组中的位置，第二次确定 在 `HashEntry` 中的位置。

`Segement` 类继承了 `ReentrantLock`，支持加锁。

```java
static class Segment<K,V> extends ReentrantLock implements Serializable {}
```

当 segment 越来越大时，锁的粒度会不断变大。同时，使用 reentrantlock 是jdk层面的实现，会增加额外的内存开销。

## 2. JDK 1.8 版本

![](http://note.youdao.com/yws/public/resource/8a56a5bf48dece1894fbd2bd605453b1/xmlnote/28E6682926BC4B8B889538F5765A738B/14396)

1.8 中，取消了 `segment` 数组，把桶换成了 `Node` 结点, 在每个 `Node` 结点通过 CAS + synchronized 来实现并发的安全性。
- `ConcurrentHashMap` 的 key 和 value 都不能为 null.

#### 为什么同步容器的键和值都不能为 null

`ConcurrentHashMap` 和 `HashTable` 都不允许键值对为 null。

在多线程情况下，无法辨别是 key 对应的值为 null，还是根本没找到对应的键。
- `HashMap` 在非并发的情况下，可以通过 `contains(key)` 来判断是否有对应的键，但并发下，可能 `get()` 后键被删除或更改，再调用 `contains(key)` 会得到修改或删除后的结果。

### 2.0 基本属性

```java
// Node 数组，默认大小为 16，每次扩容为 2 倍大小
transient volatile Node<K,V>[] table;

// Node 数组的扩容版本，扩容时使用
private transient volatile Node<K,V>[] nextTable;
```

`sizeCtl` 控制标识符，控制 `table` 数组的初始化和扩容操作
- `sizeCtl = -1` : 正在初始化
- `sizeCtl = -N` : 有 **M - 1** 个线程正在进行扩容操作
- `sizeCtl = 0` : 表示没有指定初始容量
- `sizeCtl > 0`：
    1. 如果没初始化，则代表初始化的大小
    2. 如果已经初始化过，则代表 table 的扩容阈值，默认为 0.75 * size

```java
private transient volatile int sizeCtl;
```

[sizeCtl含义纠正-- M-1](https://blog.csdn.net/Unknownfuture/article/details/105350537)

其他重要的属性值：

```java
// 默认的初始化哈希表数组的大小
private static final int DEFAULT_CAPACITY = 16;

// 转化为红黑树的阈值
static final int TREEIFY_THRESHOLD = 8

// 树化是要判断桶数组容量是否大于等于64
```

### 2.1 桶的 4 种类型

`table[]` 结点有 4 类，对应 4 种不同的桶：
1. `Node` : 链表
2. `TreeBin` : 红黑树的代理结点，控制加锁与解锁。
    - `TreeBin` 代替 `TreeNode` 挂载在桶上，指向红黑树的根结点。
    - 真正实现红黑树的结点是 `TreeNode` 结点。
3. `ForwardingNode` : 扩容时使用
4. `ReservationNode` : 保留结点，用于其他特殊方法。

#### 2.1.1 Node 链表结点

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; // 结点的 hash 字段，不是真的 hash 值
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}
```

对 `Node` 的 `hash` 字段的定义，对应不同的哈希值：

```java
// 代表正在扩容
// hash 值为 MOVED 的结点就是 ForwardingNode 结点
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
```

`Node` 结点也是其他结点，如 `TreeBin`, `ForwardingNode` 的父类，在继承后，各自重写了 `Node` 的 `find()` 方法，执行各自的查找。

#### 2.1.2 TreeBin 红黑树代理

当桶中是红黑树时，桶里存放的就是 `TreeBin` 结点。
- `TreeBin` 不存放键值对，但指向红黑树的根结点
- 保存了旗下的红黑树组成的 list
- 维护了读写锁，负责根节点的加锁和解锁

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock
    
    // methods...
}
```

#### 2.1.3 TreeNode 红黑树结点

`ConcurrentHashMap` 中的 `TreeNode` 与 `HashMap` 中的 `TreeNode` 节点类似，用于维护红黑树结构。

#### 2.1.4 ForwardingNode 

`ForwardingNode` 结点用来**标识扩容**。通过该结点来把新表和旧表联系起来，用于保证扩容时结点访问的线程安全性。

当进行扩容时，要把之前的链表迁移到新的 `nextTable` 中。当旧数组中的全部结点都被迁移到新数组后，旧数组原来的桶中会放置一个 `ForwardingNode` 结点。当原 `table[]` 数组的对应桶为空时，也会放置一个 `ForwardingNode` 结点。把节点变成 `ForwardingNode` 是为了并发地扩容，当多线程合作扩容时，某个线程碰到 `ForwardingNode` 就说明这个桶已经处理完了，可以接着遍历后面的桶。
- 该对象不保存 key 和 value，只保存扩容后哈希表的引用。
- 读操作时，若遇到了 `ForwardingNode` 结点，则转到 `nextTable` 数组去读。
- 写操作遇到 `ForwardingNode` 结点，则尝试去帮助扩容。

```java
 static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        // hash 值为 MOVED（-1）的节点就是 ForwardingNode
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
    // 到 nextTable 中查找对应元素
    Node<K,V> find(int h, Object k) {
        // 使用循环，避免多次碰到 ForwardingNode 导致递归过深
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                // 第一个节点就是要找的节点
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) { // hash < 0, 表示特殊结点
                    // 继续碰见 ForwardingNode 的情况，这里相当于是递归调用一次本方法
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else // 其他特殊节点，调用对应的 find() 进行查找
                        return e.find(h, k);
                }
                // 普通节点直接循环遍历链表
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

### 2.2 重要操作

#### 2.2.1 初始化 initTable()

`table[]` 数组是**懒初始化**的。`ConcurrentHashMap` 进行初始化的时候，不会立即执行 `initTable()` 方法，而是在第一次插入的时候进行初始化，初始化为 2 的整数倍。
- `put`, `merge`, `compute` 等
- `sizeCtl` 会指示是否存在线程正在执行初始化操作，从而保证只能同时存在一个线程对 `table` 数组进行初始化。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // 如果 table 数组没被初始化过，则进行初始化
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0 表示有其他线程在初始化，该线程要让出时间片
        if ((sc = sizeCtl) < 0)
            Thread.yield();
        // 如果该线程获取了初始化的权利，则用 CAS 设置 sizeCtl = -1，表示本线程正在初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 进行初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 下次扩容的大小
                    sc = n - (n >>> 2); ///相当于0.75*n 设置一个扩容的阈值  
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

#### 2.2.2 put()

`put()` 的流程：
0. 先保证key和value不为null，否则抛异常
1. 如果桶数组 `table[]` 没初始化，则初始化
    - 初始化会使用CAS设置sizeCtl=-1，标识正在初始化，保证只有一个线程去初始化数组
2. 计算 hash 值，确定桶的位置
3. 如果桶中还没结点，则 CAS 插入结点，然后退出
4. 如果桶不为空，分为两种情况：
    - 如果正在扩容，则先放弃插入，然后去帮助扩容
    - 否则把桶中的结点加锁，然后插入到适当的位置，或者替换已有结点的 value
        - 链表结点则插入到链表末尾
        - 红黑树结点则按红黑树进行插入
5. 如果是新插入的话，判断是否要进行树化。（长度）与 `HashMap` 相同，树化的另一个先决条件是桶数组 `table` 的长度大于 64。
6. 新插入的话，结点数+1，判断是否需要扩容。

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key, value 均不能为 null
    if (key == null || value == null) throw new NullPointerException();
    // 计算 hash 值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; // 首次赋值为桶中链表的头结点
        int n, i, fh;
        // 如果 table 数组还没初始化，则进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 如果对应的桶还没有节点，则 CAS 插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 有线程正在进行扩容操作，则先帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 把对应桶中的结点加锁，然后把结点插入到合适的位置
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // fh > 0 表示桶中储存的是链表，将节点插入到链表尾部，或者替换已有的结点
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果在链表中遍历时发现相同的 hash 和 key，则替换 value
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //putIfAbsent()
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 链表尾部，插入结点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    // 树节点的插入
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 如果链表长度已经达到临界值 8，把链表转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }

    // 如果是新插入的haul， size + 1
    addCount(1L, binCount);
    return null;
}

```

#### 2.2.3 get()

`get()` 过程没用锁。

`get()` 方法的执行步骤：
1. 计算 hash 值，然后找目标桶
2. 如果桶中的结点（头结点）是目标结点，则返回该结点
3. 否则，若 hash 为负，则分别采用 `ForwardingNode` 和 `TreeBin` 的方法进行查找
4. 否则，按正常链表进行遍历查询

```java
public V get(Object key) {
    Node<K,V>[] tab; 
    Node<K,V> e, p; 
    int n, eh; K ek;
    // 计算 hash 值
    int h = spread(key.hashCode());
    // 首先保证目标桶存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        // 如果桶中的结点就是目标结点，则直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // hash < 0 分两种情况：
        // -1 ： ForwardingNode 结点，调用 find() 去 nextTab 读
        // -2 : TreeBin 结点，按照红黑树的方法 find()
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 正常链表的遍历查询
        while ((e = e.next) != null) {
            if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    
    return null;
}
```

#### 2.2.4 transfer() 扩容

扩容会在两种情况下发生：
1. put() 插入了一个新元素，元素个数+1，触发扩容
2. 当桶中链表长度大于8，判断是否需要树化，如果桶数组长度小于64，则触发扩容

`ConcurrentHashMap` 是多线程无锁扩容。

步骤：
1. 计算每个线程处理的桶个数，最小为16个桶；
2. 如果传入的nextTab为空，则对它进行初始化，变为原数组大小的2倍
3. 多线程扩容。通过标记桶中结点为forwardingNOde来标识这个结点已经被处理过，advance变量标识该线程是否可以移动到下一个桶，finishing标识扩容结束，原数组可以清除，新数组来代替原数组。
4. 迁移数据时，对桶加锁，
    - 如果是链表的话，根据结点的hash值&原数组长度，把结点分为高位结点和低位结点两组（高位与低位的位置相差原数组长度大小），插入到新数组中
    - 如果是树的话，分割完后，判断树结点个数是否小于6，如果是，则反树化

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 控制每个线程处理的桶数量：最小16个
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 保证 nextTab[] 数组已经分配了 2 倍容量的内存
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            // 构建扩容后的数组 nextTable[]，其容量为原来容量的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];        
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // ForwardingNode 来标志扩容（ fwd.hash = -1，fwd.nextTable = nextTab ）
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true; // advance == true，表明可以移动到下一个桶
    boolean finishing = false; // 标识扩容可以结束，设置数组指向新数组
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 控制 --i ,遍历原 hash 表中的桶
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // CAS 计算得到的 transferIndex
            else if (U.compareAndSwapInt
                    (this, TRANSFERINDEX, nextIndex,
                            nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 如果当前线程是最后一个退出扩容的线程，则执行收尾工作
            // 已经完成所有节点复制了，则替换原 table 数组，并设置新的扩容阈值
            if (finishing) {
                nextTable = null;
                table = nextTab; // 更新 table 为 nextTable
                sizeCtl = (n << 1) - (n >>> 1);     // sizeCtl 阈值为原来的 0.75 * 2 = 1.5 倍
                return;
            }
            // 当前线程要退出前，设置参与扩容的线程数-1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 如果不是最后一个退出扩容的线程，则退出，保证最后留下一个线程做收尾工作
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 遍历的节点为 null，则放入到 ForwardingNode 指针节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // f.hash == -1 表示遍历到了ForwardingNode节点，意味着该节点已经处理过了
        // 这里是控制并发扩容的核心
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 节点加锁
            synchronized (f) {
                // 节点复制工作
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // fh >= 0 ,表示为链表节点
                    if (fh >= 0) {
                        // 构造两个链表  一个是原链表  另一个是原链表的反序排列
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            // 把结点高位、低位
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 把两个链表分别加入高位、低位的新数组中
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 在table i 位置处插上ForwardingNode 表示该节点已经处理过了
                        setTabAt(tab, i, fwd);
                        // advance = true 可以执行--i动作，遍历节点
                        advance = true;
                    }
                    // 如果是TreeBin，则按照红黑树进行处理，处理逻辑与上面一致
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 扩容后树节点个数若<=6，将树转链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

#### 2.2.5 spread(hashcode) 重哈希

重哈希，把高 16 位与低 16 位进行异或运算，让 hash 值更分散。

异或后的值又与 `HASH_BITS = 0x7fffffff` 取与，是为了保证 hash 值为正数。因为在 `ConcurrentHashMap` 中负的 hash 值有特殊的含义。
- `hash = -1` 代表 `ForwardingNode` 结点，表示扩容
- `hash = -2` 代表 `TreeBin` 结点，表示桶下面维护了红黑树

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```








