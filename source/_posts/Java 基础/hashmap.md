---
title: HashMap 源码分析
date: 2021-02-21 12:30:43
tags: Java
categories: 开发
---

# HashMap 源码分析

## 1. HashMap 概述

##  HashMap 与 HashTable 对比

1. 继承关系：HashTable 继承自 Dictionary 类，HashMap 继承自 AbstractMap 类
2. 线程安全性：HashMap 非线程安全，HashTable 方法为 sychronized 修饰，是线程安全的。
3. 允许null值: HashMap 的 key 和 value 都支持 null，HashTable 不支持 null。
4. 容量与扩容：HashMap 默认初始容量是 16，每次扩容大小为 2 倍；HashTable 默认初始容量 11，每次扩容为 2 * old + 1。
5. 哈希方式：HashMap 是对 hashCode() 进行再次 hash；而 HashTable 则是直接使用了 hashCode()

### 1.1 继承体系

1. AbstractMap 类
2. Map 接口
3. Cloneable 接口，可以被克隆
4. Serializable 接口，可以被序列化

![](http://note.youdao.com/yws/public/resource/8a56a5bf48dece1894fbd2bd605453b1/xmlnote/BE905D83CB06493B9668B3E8BEAF28E7/14242)

### 1.2 存储结构

#### 1.2.1 JDK 1.7

1. HashMap 在 Java7 之前是采用 **数组+链表** 的存储结构。
2. 链表结点的插入采用的是**头插法**
    - 并发的插入可能会覆盖其他线程插入的结点
    - 头插法在扩容时会调转元素原本的顺序，当并发扩容时，如果一个线程已经移动完某些节点，但另一个线程不知道，则会在获得时间片后再次修改指针，从而导致链表成环的问题。
3. 在扩容的时候会重新计算哈希值和索引位置。

#### 1.2.2 JDK 1.8

1. Java8 开始，HashMap 采用 **数组 + 链表 + 红黑树** 的结构来存储。
    - 数组又被称为 `hash 桶`，根据 hash 值来计算桶的位置。
    - 链表用来解决 hash 冲突。在链表长度达到一定数目时，会转化为红黑树来存储。
2. 把头插法改为**尾插法**，在扩容是会保持链表元素本来的顺序，避免了链表成环的问题。
3. 在扩容时不会重新计算哈希值，通过判断 `(e.hash & oldCap) == 0` 操作来计算新的索引位置。

## 2. 属性

- 默认初始数组容量是 `16`，默认装载因子是 `0.75f`，容量总是 `2 的 n 次幂`。
    - 在使用 hashMap 时，尽量提前指定容量，避免多次扩容
        - 容量设置成 `expectedSize/0.75 + 1`，因为有负载因子0.75的影响
    - 默认装载因子是 0.75，是权衡了查找效率与空间占用。
        - 负载因子太大，会导致过多的哈希冲突；负载因子太小，会浪费空间。
- 扩容问题：扩容时每次变为原来的两倍，为了保证容量 2 的n次幂。
    - 保证2的n次幂是为了在确定桶的时候可以使用**位与运算**来加快效率。正常来说求桶的位置应该是把hash对桶个数取余，当桶个数为2^N时，可以通过位与来实现。
    - 扩容的条件是大于临界值
        - 临界值 `threshold = loadFactor * capacity`
- 树化 与 链表化：
    - 当桶的数量小于 64 时不会直接进行树化，要先尝试扩容。
    - 当桶数量大于 64 并单个桶中结点个数大于 8 时，进行树化。
    - 当单个桶中元素数量小于 6 时，进行反树化。
- 线程安全性：`HashMap` 是非线程安全的。

### 2.1 默认属性

```java
// 默认的数组长度为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// 最大的数组长度为2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 当一个桶中的元素个数大于等于8时进行树化
static final int TREEIFY_THRESHOLD = 8;

// 当一个桶中的元素个数小于等于6时把树转化为链表
static final int UNTREEIFY_THRESHOLD = 6;

// 当桶的个数达到64的时候才进行树化
static final int MIN_TREEIFY_CAPACITY = 64;
```

### 2.2 属性

```
// 数组，又叫作桶（bucket）
transient Node<K,V>[] table;

// 作为entrySet()的缓存
transient Set<Map.Entry<K,V>> entrySet;

// 所有结点的数量
transient int size;

// 修改次数，用于在迭代的时候执行fail-fast策略
transient int modCount;

// 当桶的使用数量达到多少时进行扩容，threshold = capacity * loadFactor
int threshold;

// 装载因子
final float loadFactor;
```

## 3. 内部类

Map 结点的继承关系：

- Map.Entry<K, V> 内部接口
    - HashMap.Node<K, V> 内部类
        - LinkedHashMap.Entry<K, V> 内部类
            - HashMap.TreeNode<K, V> 内部类

### 3.1 Node 内部类
Node 内部类用于链表结点的实现。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    ...
    ...
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
    
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```



### 3.2 TreeNode 内部类

TreeNode 结点作为红黑树实现时的结点。

HashMap 类中的TreeNode内部类：
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
}
```
继承自LinkedHashMap中的Entry内部类：
```java
// 位于LinkedHashMap中，典型的双向链表节点
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

## 4. 构造方法

### 4.1 空构造器

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

### 4.2 HashMap(int initialCapacity) 构造器

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

### 4.3 HashMap(int initialCapacity, float loadFactor)

```java
public HashMap(int initialCapacity, float loadFactor) {
    // 检查传入的初始容量是否合法
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 检查装载因子是否合法
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    // 计算扩容门槛
    this.threshold = tableSizeFor(initialCapacity);
}
```

**下一次扩容大小的计算规则**：

```java
// 计算扩容门槛，初始容量往上取最近的2的n次方
static final int tableSizeFor(int cap) {
    // 先 -1 为了避免 cap 本来就是2的整数次方
    int n = cap - 1;
    // 把最高位及之后全填充为 1
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## 5. HashMap 常用 API

### 5.1 hash() 扰动函数

hash() 方法用与在 HashMap 中计算 hash 值。

(n - 1) & hash 用来确定桶的位置，n是桶的数目，是2的整数次方。
- 采用与运算可以提高计算效率
- 之所引 HashMap 容量是 2 的n次幂，一方面也有这个原因，因为这样n-1所有位都是1，能保证取到桶的所有位置

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

1. `hashCode()` 方法若不进行重写则会根据内存地址来计算得到
2. null 的 hash 值为 0
3. 让int值的 低16位 与 高16位 进行异或操作，是为了让计算出来的 hash 值更加分散，这样可以同时保留高16位与低16位的信息。采用异或而不是与和或，是因为与计算值向0靠拢，或计算值向1靠拢，而异或能更好保持各部分特征。


### 5.2 get()

1. 首先计算 key 的hash值，然后调用 getNode() 方法进行查找
2. 找到对应的桶
    - 检查桶中元素是不是目标key，是就直接返回
    - 检查结点类型
        - 如果是树，调用getTreeNode()
        - 如果是链表，则遍历链表查找

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

#### 5.2.1 getNode() 链表检索

真正的get方法：

```java
final Node<K, V> getNode(int hash, Object key) {
    Node<K, V>[] tab;
    Node<K, V> first, e;
    int n;
    K k;
    // 检查目标桶是否存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
        // 检查第一个元素是不是要查的元素，如果是直接返回
        if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 如果第一个元素是树节点，则按树的方式查找
            if (first instanceof TreeNode)
                return ((TreeNode<K, V>) first).getTreeNode(hash, key);

            // 否则就遍历整个链表查找该元素
            do {
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### 5.2.2 TreeNode.getTreeNode() 树检索

1. 找到树的根，从根开始查找
2. 

```java
final TreeNode<K, V> getTreeNode(int h, Object k) {
    // 先找到树的根，然后从树的根节点开始查找
    return ((parent != null) ? root() : this).find(h, k, null);
}

final TreeNode<K, V> find(int h, Object k, Class<?> kc) {
    TreeNode<K, V> p = this;
    do {
        int ph, dir;
        K pk;
        TreeNode<K, V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h) // 左子树
            p = pl;
        else if (ph < h) // 右子树
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk))) // 找到了直接返回
            return p;
        else if (pl == null) // hash相同但key不同，左子树为空查右子树
            p = pr;
        else if (pr == null) // 右子树为空查左子树
            p = pl;
        else if ((kc != null ||
                (kc = comparableClassFor(k)) != null) &&
                (dir = compareComparables(kc, k, pk)) != 0)
            // 通过compare方法比较key值的大小决定使用左子树还是右子树
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            // 如果以上条件都不通过，则尝试在右子树查找
            return q;
        else
            // 都没找到就在左子树查找
            p = pl;
    } while (p != null);
    return null;
}
```

### 5.3 put()

put() 方法在放入新的 key-value 对，并返回旧值。

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```
#### 5.3.1 putVal() 方法

**过程**：首先对 key 进行 hash，并找到对应的桶 `(n - 1) & hash`：
- 如果是树，则在红黑树中查找该key
    - 如果找到了对应结点，则返回该节点，并在`putVal()`进行value的更改
    - 如果没找到，则在`putTreeVal()`中新建结点，并返回null给`putVal()`
- 如果是链表，则从前至后进行遍历，查找该key
    - 如果没有找到目标key，则新建结点连在链表最后
    - 如果找到了目标key，则把value进行替换，并返回旧的value
- 插入元素之后，判断是否需要扩容

真正的 `put()` 底层：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K, V>[] tab;
    Node<K, V> p;
    int n, i;
    // 如果桶的数量为0，则调用resize()初始化，即 扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // (n - 1) & hash 计算元素在哪个桶中
    // 如果这个桶中还没有元素，则新建一个节点放在桶中的第一个位置
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else { // 如果桶中已经有元素存在了
        Node<K, V> e;
        K k;
        // 判断桶中第一个元素的key与待插入元素的key是否相同
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode) // 如果是树节点，则调用树的处理方法
            e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
        else { // 对于链表，遍历这个链表，寻找目标 key值
            for (int binCount = 0; ; ++binCount) {
                // 如果链表遍历完了都没有找到相同key的元素，则在链表最后新插入
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 判断是否需要树化，因为第一个元素没有算到binCount中，所以这里 -1
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果待插入的key在链表中找到了，则退出循环
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 如果在桶中找到了对应 key的元素
        if (e != null) { // existing mapping for key
            // 记录下旧值
            V oldValue = e.value;
            // 判断是否需要替换旧值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 在节点被访问后做点什么事，在 LinkedHashMap中用到
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 前边没 return，则说明没有找到元素
    ++modCount; // 修改次数加 1
    // 元素数量加 1，判断是否需要扩容
    if (++size > threshold)
        resize();
    // 在节点插入后做点什么事，在LinkedHashMap中用到
    afterNodeInsertion(evict);
    return null;
}
```

#### 5.3.2 putTreeVal() 加入树结点

`putTreeVal()` 方法是用来寻找红黑树中对应key结点的位置，其中在红黑树中的比较是通过比较key的hash值的。

**查找过程**：
1. 寻找树的根结点
2. 从根结点开始查找对应的key
    - 在树中查找key时，采用 `dir` 来标志向左查找还是向右查找 
3. 若找到了（hash值相同且key相同）则返回地址，没找到则自建结点并返回null。
    - 若返回null，则返回后其调用者`putVal()`不需再处理该节点
    - 若返回了结点的地址，则还需在调用者`putVal()`中对value进行更改

> 若新建结点之后，需要对红黑树进行平衡处理，并把新的root放到table数组中。

```java
final TreeNode<K, V> putTreeVal(HashMap<K, V> map, Node<K, V>[] tab,
                                int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false; // 标记是否找到这个key的节点
    TreeNode<K, V> root = (parent != null) ? root() : this; // 树的根节点
    // 遍历红黑树
    for (TreeNode<K, V> p = root; ; ) {
        // dir=direction，标记是在左边还是右边
        int dir, ph;
        K pk;
        if ((ph = p.hash) > h) { // 当前hash比目标hash大，说明在左边
            dir = -1;
        }
        else if (ph < h) // 当前hash比目标hash小，说明在右边
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            // 两者hash相同且key相等，说明找到了节点，直接返回该节点
            return p;
        else if ((kc == null &&
                // 如果k是Comparable的子类则返回其真实的类，否则返回null
                (kc = comparableClassFor(k)) == null) ||
                // 如果k和pk不是同样的类型则返回0，否则返回两者比较的结果
                (dir = compareComparables(kc, k, pk)) == 0) {
            // 这个条件表示两者hash相同但是其中一个不是Comparable类型或者两者类型不同
            // 比如key是Object类型，这时可以传String也可以传Integer，两者hash值可能相同
            // 在红黑树中把同样hash值的元素存储在同一颗子树，这里相当于找到了这颗子树的顶点
            // 从这个顶点分别遍历其左右子树去寻找有没有跟待插入的key相同的元素
            if (!searched) {
                TreeNode<K, V> q, ch;
                searched = true;
                // 遍历左右子树找到了直接返回
                if (((ch = p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                                (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            // 如果两者类型相同，再根据它们的内存地址计算hash值进行比较
            dir = tieBreakOrder(k, pk);
        }

        TreeNode<K, V> xp = p;
        // 如果最后没找到对应key的结点，则新建一个节点
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K, V> xpn = xp.next;
            TreeNode<K, V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K, V>) xpn).prev = x;
            // 对树进行平衡处理，并把 root 移动到 table 中
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

### 5.4 resize() 扩容

用于**初始化**或**扩容**桶（数组）的数量
- 初始化：oldTab == null，oldCap == 0
    - 默认构造器建的map初始化
    - 非默认构造器建的map初始化
- 扩容：oldTab != null

**扩容的过程**：
新建容量更大（**2倍**）的数组，然后遍历旧数组中的各个桶，把桶中的树/链表进行**分裂**，然后放到新数组的桶中：
- 树：调用树的API对原树进行分裂，然后加入到新桶中
    - 分裂后的新桶和旧桶要判断是否元素个数小于等于6，然后进行反树化
- 链表：每个桶中的结点分裂成**低位链表**和**高位链表**
    - 低位链表在新桶中的位置与旧桶的位置相同
    - 高位链表在新桶中的位置在原来位置的基础上偏移一个旧桶的容量

```java
final Node<K, V>[] resize() {
    // 保存旧数组、旧容量、旧扩容门槛
    Node<K, V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 如果旧容量达到了最大容量，则不再进行扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 如果旧容量翻倍后小于最大容量并且旧容量大于等于默认初始容量
            // 则容量翻倍，扩容门槛也翻倍
            newThr = oldThr << 1; // double threshold
        // 如果旧容量翻倍后达到了最大容量，则同样会进行扩容，但newThr要重新计算
    } else if (oldThr > 0) // initial capacity was placed in threshold
        // 调用非默认构造器创建的map，桶容量的初始化
        // 如果旧容量为0且旧扩容门槛大于0，则把新容量赋值为旧门槛
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 调用默认构造方法创建的map，其旧容量旧扩容门槛都是0
        // 初始化容量为默认容量，扩容门槛 = 默认容量 * 默认装载因子
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // 如果新扩容门槛为0，则计算为容量*装载因子，但不能超过最大容量
    if (newThr == 0) {
        float ft = (float) newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
                (int) ft : Integer.MAX_VALUE);
    }
    
    // 更新扩容门槛
    threshold = newThr;
    
    // 新建一个新容量的数组，并更新table数组
    @SuppressWarnings({"rawtypes", "unchecked"})
    Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
    table = newTab;
    
    // 遍历旧数组，搬运元素到table数组中
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K, V> e;
            // 如果桶中第一个元素不为空，赋值给e
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 清空旧桶，便于GC回收
                // 如果桶中只有一个元素
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    //如果桶中结点数大于1，并且是树结点，则把这颗树打散成两颗树插入到新桶中去
                    ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 桶中结点数大于1，并且是链表结点，则分成两个链表插入新桶
                    Node<K, V> loHead = null, loTail = null;
                    Node<K, V> hiHead = null, hiTail = null;
                    Node<K, V> next;
                    do {
                        next = e.next;
                        // 按照(e.hash & oldCap) == 0条件是否满足来划分为2个链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {

                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null); // 到这完成了原链表的分裂

                    // 低位链表在新桶中的位置与旧桶一样
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 高位链表在新桶中的位置正好是原来的位置加上旧容量（即7和15搬移到七号桶了）
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

> HashMap数组的大小始终是2的整数次幂，扩容的最大容量是2的30次方，每次扩容会使容量翻倍，所以最终的扩容正好等于最大容量，不会超过。

### 5.5 树化

#### 5.5.1 treeifyBin()

当插入元素后，如果桶中链表长度大于等于8，则判断是否要树化。
- 如果桶数量小于64，直接**扩容**而不用树化，思想是：扩容后会进行链表分化，从而减少链表长度
    - 如果链表中元素hash值相同，则链表不一定进行分化
- 否则把桶中所有的链表结点替换为树结点，然后调用 `treeify()` 方法进行树化

```java
final void treeifyBin(Node<K, V>[] tab, int hash) {
    int n, index;
    Node<K, V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) // 桶少于64，以扩容代替树化
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) { // 否则进行树化
        TreeNode<K, V> hd = null, tl = null;
        // 把所有节点换成树节点
        do {
            TreeNode<K, V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        // 如果进入过上面的循环，则从头节点开始树化
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

#### 5.5.2 treeify()

该方法是真正的树化方法。
1. 把链表头当做树的根
2. 依次插入其余的结点，默认插入红结点
3. 插入结点后进行红黑树的平衡
4. 插入所有节点后把树根移动到桶中，因为红黑树平衡会改变根节点

```java
final void treeify(Node<K, V>[] tab) {
    TreeNode<K, V> root = null;
    for (TreeNode<K, V> x = this, next; x != null; x = next) {
        next = (TreeNode<K, V>) x.next;
        x.left = x.right = null;
        // 第一个元素作为根节点且为黑节点，其它元素依次插入到树中,再做平衡
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        } else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            // 从根节点开始，查找结点的插入位置
            for (TreeNode<K, V> p = root; ; ) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                        (kc = comparableClassFor(k)) == null) ||
                        (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                // 一直找到叶子结点，然后插入，默认插入的是红结点
                TreeNode<K, V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 平衡红黑树
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    // 把根节点移动到链表的头节点，因为经过平衡之后原来的第一个元素不一定是根节点了
    moveRootToFront(tab, root);
}
```

### 5.6 remove()

1. 对 key 进行 hash
2. 找对应的桶
    - 如果桶中结点就是目标结点，则赋值给node
    - 否则判断结点类型
        - 如果是树结点，则getTreeNode()查找树结点
        - 如果是链表结点，则遍历链表查找对应结点
3. 删除结点
    - 如果是树，按树的方式删除
    - 如果是链表，直接删
        - 如果是链表的第一个结点，则第二个顶上来

```java
public V remove(Object key) {
    Node<K, V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
```

#### 5.6.1 removeNode() 删除链表结点

```java
final Node<K, V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K, V>[] tab;
    Node<K, V> p; // 头结点
    int n, index;
    // 如果桶存在
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
        Node<K, V> node = null, e;
        K k;
        V v;
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            // 如果第一个元素正好就是要找的元素，赋值给node变量后续删除使用
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                // 如果第一个元素是树节点，则以树的方式查找节点
                node = ((TreeNode<K, V>) p).getTreeNode(hash, key);
            else {
                // 否则遍历整个链表查找元素
                do {
                    if (e.hash == hash &&
                            ((k = e.key) == key ||
                                    (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 如果找到了元素，则看参数是否需要匹配value值
        // 1. 如果不需要匹配直接删除
        // 2. 如果需要匹配则看value值是否与传入的value相等
        if (node != null && (!matchValue || (v = node.value) == value ||
                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                // 如果是树节点，调用树的删除方法（以node调用的，是删除自己）
                ((TreeNode<K, V>) node).removeTreeNode(this, tab, movable);
            else if (node == p)
                // 如果待删除的元素是第一个元素，则把第二个元素移到第一的位置
                tab[index] = node.next;
            else
                // 否则删除node节点
                p.next = node.next;
            ++modCount;
            --size;
            // 删除节点后置处理
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

#### 5.6.2 TreeNode.removeTreeNode() 删除树结点

`TreeNode` 继承自 `Node`，本身既有链表结点的属性，又有树特有的属性。`removeTreeNode()` 方法首先删除链表属性部分，即改变链表指针，然后删除红黑树属性部分，再做树的平衡。

```java
final void removeTreeNode(HashMap<K, V> map, Node<K, V>[] tab,
                          boolean movable) {
    int n;
    // 如果桶的数量为0直接返回
    if (tab == null || (n = tab.length) == 0)
        return;
    // 找到对应的桶
    int index = (n - 1) & hash;
    // 节点定义
    // 第一个节点，根节点，根左子节点
    TreeNode<K, V> first = (TreeNode<K, V>) tab[index], root = first, rl;
    // 后继节点，前置节点
    TreeNode<K, V> succ = (TreeNode<K, V>) next, pred = prev;

    // 如果前置节点为空，说明当前节点是根节点，则把后继节点赋值到第一个节点的位置，相当于删除了当前节点
    if (pred == null)
        tab[index] = first = succ;
    else // 非根，则把前置节点的下个节点设置为当前节点的后继节点，相当于删除了当前节点
        pred.next = succ;

    // 如果后继节点不为空，则让后继节点的前置节点指向当前节点的前置节点，相当于删除了当前节点
    if (succ != null)
        succ.prev = pred;

    // 如果第一个节点为空，说明没有后继节点了，直接返回
    if (first == null)
        return;

    // 如果根节点的父节点不为空，则重新查找父节点
    if (root.parent != null)
        root = root.root();

    // 如果根节点为空，则需要反树化（将树转化为链表）
    // 如果需要移动节点且树的高度比较小，则需要反树化
    if (root == null
            || (movable
            && (root.right == null
            || (rl = root.left) == null
            || rl.left == null))) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }

    // 分割线，以上都是删除链表中的节点，下面才是直接删除红黑树的节点（因为TreeNode本身即是链表节点又是树节点）

    // 删除红黑树节点的大致过程是寻找右子树中最小的节点放到删除节点的位置，然后做平衡
    TreeNode<K, V> p = this, pl = left, pr = right, replacement;
    if (pl != null && pr != null) {
        TreeNode<K, V> s = pr, sl;
        while ((sl = s.left) != null) // find successor
            s = sl;
        boolean c = s.red;
        s.red = p.red;
        p.red = c; // swap colors
        TreeNode<K, V> sr = s.right;
        TreeNode<K, V> pp = p.parent;
        if (s == pr) { // p was s's direct parent
            p.parent = s;
            s.right = p;
        } else {
            TreeNode<K, V> sp = s.parent;
            if ((p.parent = sp) != null) {
                if (s == sp.left)
                    sp.left = p;
                else
                    sp.right = p;
            }
            if ((s.right = pr) != null)
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)
            sr.parent = p;
        if ((s.left = pl) != null)
            pl.parent = s;
        if ((s.parent = pp) == null)
            root = s;
        else if (p == pp.left)
            pp.left = s;
        else
            pp.right = s;
        if (sr != null)
            replacement = sr;
        else
            replacement = p;
    } else if (pl != null)
        replacement = pl;
    else if (pr != null)
        replacement = pr;
    else
        replacement = p;
    if (replacement != p) {
        TreeNode<K, V> pp = replacement.parent = p.parent;
        if (pp == null)
            root = replacement;
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;
    }

    TreeNode<K, V> r = p.red ? root : balanceDeletion(root, replacement);

    if (replacement == p) {  // detach
        TreeNode<K, V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }
    }
    if (movable)
        moveRootToFront(tab, r);
}
```




