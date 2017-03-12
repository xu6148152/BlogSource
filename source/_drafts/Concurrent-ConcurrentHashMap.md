---
title: Concurrent-ConcurrentHashMap
tags: Java-Concurrent
---

### 概要
``jdk1.5``之后引入的线程安全容器，该类与``HashTable``的规范相同，有与``HashTable``对应的方法.其1.8之前使用分段锁，之后使用``Unsafe``和``syncronized``来确保线程安全，而``HashTable``则是对整个操作直接使用``synchronized``来确保线程安全，这也造成了``ConcurrentHashMap``的性能会比``HashTable``有比较大的提升

#### 源码分析

```java
//最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;
//默认容量,一般来说为了内存对齐，因此大小一般都为2^*
private static final int DEFAULT_CAPACITY = 16;

//默认并发等级16
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
```

使用``Node<K, V>``作为基本数据结构

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
散列函数用于解决冲突

##### 构造方法

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}

private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

* 能够指定容量,容量的大小为2的次幂

##### get

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
* 从``Node``数组里面找键对应的值

这里需要提到

```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

* ``tabAt``和``casTabAt``,``casTabAt``成功设置后，对象已经发生改变，那么``tabAt``会返回``null``。当有多个线程对``ConcurrentHashMap``进行操作时，包括插入和读取，那么这个时候采用这种方式就能保证数据的正确性

##### containsValue

```java
public boolean containsValue(Object value) {
    if (value == null)
        throw new NullPointerException();
    Node<K,V>[] t;
    if ((t = table) != null) {
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V v;
            if ((v = p.val) == value || (v != null && value.equals(v)))
                return true;
        }
    }
    return false;
}
```
* 使用``Traverser``遍历寻找对象

```java
final Node<K,V> advance() {
    Node<K,V> e;
    if ((e = next) != null)
        e = e.next;
    for (;;) {
        Node<K,V>[] t; int i, n;  // must use locals in checks
        if (e != null)
            return next = e;
        if (baseIndex >= baseLimit || (t = tab) == null ||
            (n = t.length) <= (i = index) || i < 0)
            return next = null;
        if ((e = tabAt(t, i)) != null && e.hash < 0) {
            if (e instanceof ForwardingNode) {
                tab = ((ForwardingNode<K,V>)e).nextTable;
                e = null;
                pushState(t, i, n);
                continue;
            }
            else if (e instanceof TreeBin)
                e = ((TreeBin<K,V>)e).first;
            else
                e = null;
        }
        if (stack != null)
            recoverState(n);
        else if ((index = i + baseSize) >= n)
            index = ++baseIndex; // visit upper slots if present
    }
}
```

* 遍历``Node``表，区分当前节点是``TreeBin``还是``ForwardingNode``，如果是后者，保存状态。直到查找结束

##### put

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //表为空或者已满
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
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
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

* 表为空或者已满，初始化表(使用``cas``)
* 使用``synchronized``来同步真正插入过程

