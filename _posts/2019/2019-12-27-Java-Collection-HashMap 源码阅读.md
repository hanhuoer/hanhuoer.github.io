---
layout: post
title: Java - Collection - HashMap 源码阅读
categories: [Java]
category: Java
tags: [Java]
keywords: Java,Collection,Map,HashMap,集合
---

> 以下代码均基于 Java 8


## HashMap 是什么？

HashMap 是基于哈希表的实现，是一个用于存储键值对（key，value）的集合，它的 key/value 均可为 null；由于它是基于哈希表的，这意味着 HashMap 不能保证存入键值的有序性；它的实现并非是同步的，因此它并不是线程安全的。


## HashMap 的原理是什么？

HashMap 其实就是哈希表，键值对的存储位置是根据 key 的哈希码与当前最大容量减一做与运算得到的，为了避免哈希冲突，HashMap 的设计者采用了数组 + 链表的存储结构，当发生冲突的时候，HashMap 将使用链表存储发生哈希冲突的元素。如果任由链表无限增大，HashMap 的查找速度必定下降，我们知道，链表的查询速度较慢，为了解决查询速度慢，HashMap 会在合适的时候将链表的结构转换为红黑树。

如果当前容器的容量大于 TREEIFY_THRESHOLD（默认为 8），那么将会考虑对该链表树形化。树形化的时候有一个重要的常量 MIN_TREEIFY_CAPACITY，当满足 **[key-value 数量 > TREEIFY_THRESHOLD]** 并且 **[table的长度 >= MIN_TREEIFY_CAPACITY]** 的时候才会将当前存储结构转换为红黑树；如果不满足条件时，HashMap 会对当前 table.resize() 扩容一次。


### 常量

```java
/**
 * 默认初始化容量，该值必须为二次幂
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * 最大容量，可以在构造函数中指定最大容量，如果超出最大容量，则默认使用最大容量
 * 容量 = 二次幂 <= 1<<30
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 负载因子，可以在构造函数中设置改值
 * HashMap 会通过当前容量和负载因子计算出什么时候进行扩容，HashMap 将会把计算结果赋值给 TREEIFY_THRESHOLD
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * HashMap 将存储结构从链表转为红黑树的阈值
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * HashMap 将存储结构从红黑树转为链表的阈值，这个值要小于 TREEIFY_THRESHOLD
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * 树形化的最小阈值，这个值至少满足 4 * TREEIFY_THRESHOLD
 * 如果数组中桶的个数大于 TREEIFY_THRESHOLD，但 capacity 小于 MIN_TREEIFY_CAPACITY，那么依然使用链表结构，此时会对 HashMap 进行扩容；
 * 如果数组中桶的个数大于 TREEIFY_THRESHOLD，并且 capacity 大于 MIN_TREEIFY_CAPACITY，那么 HashMap 才会将存储结构树形化
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```


### resize

HashMap 的扩容方法，会在原容量的基础上扩大两倍。

```java
/**
 * 初始化或者扩容的时候使用
 * 初始化的时候按照当时的阈值以及初始容量进行分配
 * 此外，由于是使用2次幂扩容，所以每个链表中的元素必须保持相同的索引，或者在新表中以2次幂进行偏移
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 当前 table 的容量是否大于 0
    if (oldCap > 0) {
        // table 的容量已经超过可以使用的最大容量，直接置 threshold 为 0x7fffffff，并且返回原 table
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // tbale 的容量没有超过最大值，将原容量以及阈值扩大两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 初始化
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算新的 threshold
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    // 创建一个新容量大小的空的 table，准备向里面复制内容
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 遍历旧 table
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // table[j] 不为空
            if ((e = oldTab[j]) != null) {
                // 释放旧 table[j] 的引用
                oldTab[j] = null;
                // table[j] 的下一个节点为空，说明 table[j] 只有一个节点（只有一个元素）
                if (e.next == null)
                    // 把 table[j] 的 key-value 放到扩容后新的位置
                    newTab[e.hash & (newCap - 1)] = e;
                // table[j] 是个红黑树
                else if (e instanceof TreeNode)
                    // 在树中操作
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // table[j] 是个链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                // 链表为空时，将当前节点设置为头节点
                                loHead = e;
                            else
                                // 否则设置尾的下一个节点为当前节点
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
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


### hash

```java
/**
 * 计算 key.hashCode() 的高位与低位的异或值
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```


### put

可以看到，下面第二段代码第 19 行 `if ((p = tab[i = (n - 1) & hash]) == null)` 这是在计算键值对存储位置。这里比较巧妙 `table[(n - 1) & hash]`，在与运算之前做了减一操作，使其为奇数，这样就保证与运算出来的结果可能为奇数也可能为偶数。

```java
/**
 * 在 HashMap 中关联指定的 key 和 value
 * 如果 put 操作之前 HashMap 中存在该 key，那么旧的 value 将会被替换
 *
 * @param key 与 value 关联的指定 key
 * @param value 与 key 关联的指定 value
 */
public V put(K key, V value) {
    // 计算 key 的 hash 码，然后存储，如果存在 key 则替换原有的 value
    return putVal(hash(key), key, value, false, true);
}
```

```java
/**
 * Map.put 的实现方法
 *
 * @param hash key 的 hash 码
 * @param key key
 * @param value value
 * @param onlyIfAbsent 如果为 true，则不改变已存在的值
 * @param evict 如果为 false，则为创建模式
 * @return 之前的值，如果没有则返回 null
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1. 如果 table 为空或者 table.length = 0，那么将会调用 resize() 得到一个经过初始化的 Node 对象
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2. 如果 tab[(n - 1) & hash] 为空，那么将会在此下标处直接创建一个节点并添加 key-value
    // 下一步跳转到 6
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 3. 如果 table[i] 的首个元素的 K 是否等于 key，如果相等，则直接覆盖 value
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 4. 如果 p 是红黑树，则在树中插入 key-value
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 这个逻辑块就属于链表了，接下来将遍历 table[i]
            for (int binCount = 0; ; ++binCount) {
                // 如果下一个节点为空
                if ((e = p.next) == null) {
                    // 直接在链表的下一个节点进行新建节点
                    p.next = newNode(hash, key, value, null);
                    // 如果链表长度已经满足 TREEIFY_THRESHOLD，则做树形操作
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果发现 key 已经存在，则直接覆盖 value
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 5. 如果 key 已存在，新的 value 则会覆盖旧的 value，并且返回旧的 value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 6. 判断当前存在的 key-value 数量是否超出 threshold，如果超过则扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

放上一张 put 执行的流程图

![1577359222922](../_picture/2019/1577359222922.png)


### get

```java
/**
 * 返回指定 key 的 value，如果没有此 key 的映射，那么将会返回 null
 */
public V get(Object key) {
    Node<K,V> e;
    // 计算出 key 的哈希值，获取 node
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```java
/**
 * @param hash key 的哈希值
 * @param key key
 * @return 节点，如果不存在返回 null
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 1. table 非空并且 table.length 大于 0 并且 (n-1)&hash 这个位置非空，否则直接 return null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 2. 检查 (n-1)&hash 位置的第一个节点的 key 是否与操作 key 一致，如果一致直接返回第一个节点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 3. 如果是树节点，那么将在树中取
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 4. 这里就是链表结构，将遍历链表进行查找
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
