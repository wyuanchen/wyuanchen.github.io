---
title: HashMap工作原理学习
date: 2017-2-07 22:21:18
categories: Java
tags: 容器
---

## 存储结构

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;    //用来定位数组索引位置
        final K key;
        V value;
        Node<K,V> next;   //链表的下一个node

        Node(int hash, K key, V value, Node<K,V> next) { ... }
        public final K getKey(){ ... }
        public final V getValue() { ... }
        public final String toString() { ... }
        public final int hashCode() { ... }
        public final V setValue(V newValue) { ... }
        public final boolean equals(Object o) { ... }
}
```

<!-- more -->

![](http://tech.meituan.com/img/java-hashmap/hashMap%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE.png)

## 两个重要的参数  

在HashMap中有两个很重要的参数：

- 容量(Capacity)   
- 负载因子(Load factor)

`Capacity`就是bucket的大小，`Load factor`就是bucket填满程度的最大比例，默认是`0.75`。 

当bucket中的entries的数目大于`capacity*load factor`时就需要调整bucket的大小为当前的2倍。


## hash函数的实现

以`Entry[]`数组实现的哈希桶数组(bucket)，用`Key`的哈希值取模桶数组的大小可得到数组下标。

`HashMap`跟`HashTable`的区别之一就是：

- `HashTable`直接用`Key`对象的`hashCode()`作为hash值

- `HashMap`会先调用多一个`hash()`对`Key`对象的`hashCode`进行计算从而得出最终的hash值

在`get`和`put`的过程中，先对hashCode进行hash操作，然后再通过hash值进一步计算下标

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`HashMap`允许`key`为`null`，并且当`key`为`null`的时候，`hash`值都为0


![](https://cloud.githubusercontent.com/assets/1736354/6957712/293b52fc-d932-11e4-854d-cb47be67949a.png)

### 计算下标

因为目前的table长度n为2的幂，而计算下标的时候，是这样实现的(使用`&`位操作，而非`%`求余)： `（n - 1）& hash`
 
因此数组的大小永远是2的N次方， 你随便给一个初始值比如17会转为32。
默认初始容量是16。


## put

`put`方法大致的流程为

1. 用`Key`对象的`hashCode()`去计算`hash`值，进而求下标

2. 如果没碰撞直接放到bucket里

3. 如果碰撞了，以链表的形式存在buckets后面

4. 如果碰撞导致链表过长（大于等于`TREEIFY_THRESHOLD`，默认是8），就把链表转换成红黑树

5. 如果节点已经存在就替换`old value`

6. 如果`bucket`满了(超过`load factor*current capacity`)，就要`resize()`

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

## get

大致流程为

1. 如果是bucket里的第一个节点，则直接命中

2. 如果有冲突：
    
    1. 若为链表，则遍历链表查找，`O(n)`

    2. 若为树，则在树中通过查找，`O(logn)`

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

/**
 * Implements Map.get and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

## Java8 对`HashMap`的改进

> Improve the performance of java.util.HashMap under high hash-collision conditions by using balanced trees rather than linked lists to store map entries. Implement the same improvement in the LinkedHashMap class.

在Java 8之前的实现中是用链表解决冲突的，在产生碰撞的情况下，进行get时，两步的时间复杂度是`O(1)+O(n)`。因此，当碰撞很厉害的时候`n`很大，`O(n)`的速度显然降低效率。

因此在Java 8中，利用**红黑树**替换链表，这样复杂度就变成了`O(1)+O(logn)`了，这样在`n`很大的时候，能够比较理想的解决这个问题。


## resize

当`put`时，如果发现目前的`bucket`占用程度已经超过了`Load Factor`所希望的比例，那么就会发生`resize`。

在resize的过程，简单的说就是把bucket扩充为2倍，之后重新计算index，把节点再放到新的bucket中。  

因为`bucket`的长度一直是2次幂，所以在长度变为两倍后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。

例如我们从16扩展为32时，具体的变化如下所示

![](https://cloud.githubusercontent.com/assets/1736354/6958256/ceb6e6ac-d93b-11e4-98e7-c5a5a07da8c4.png)


因此元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![](https://cloud.githubusercontent.com/assets/1736354/6958301/519be432-d93c-11e4-85bb-dff0a03af9d3.png)


因此，在扩充HashMap的时候，不需要重新计算hash，只需要看看原来的hash值新增的那个bit是`1`还是`0`就好了，

- 是`0`的话索引没变  
- 是`1`的话索引变成`原索引+oldCap`

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。

16扩充为32的resize示意图：

![](https://cloud.githubusercontent.com/assets/1736354/6958677/d7acbad8-d941-11e4-9493-2c5e69d084c0.png)


```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
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

## 参考资料

- [Java HashMap工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/#5-_hash函数的实现)
- [Java 8系列之重新认识HashMap](http://tech.meituan.com/java-hashmap.html)
