---
title: LinkedHashMap原理学习
date: 2016-09-13 22:36:22
categories: Java
tags: 容器
---

- LinkedHashMap的迭代输出的结果保持了插入顺序。    
- LinkedHashMap是Hash表和链表的实现，并且依靠着双向链表保证了迭代顺序是插入的顺序。      
- 根据链表中元素的顺序可以分为：    

    - 按插入顺序的链表   
    - 按访问顺序(调用 get 方法)的链表。   

    默认是按插入顺序排序，如果指定按访问顺序排序，那么调用get方法后，会将这次访问的元素移至链表尾部，不断访问可以形成按访问顺序排序的链表。      


<!-- more -->

### 构造   

```java    
LinkedHashMap(int initialCapacity, float loadFactor)   
LinkedHashMap(int initialCapacity)   
LinkedHashMap()   
LinkedHashMap(Map<? extends K, ? extends V> m)   
LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)
```

`accessOrder` 这个参数，如果不设置，**默认为 `false`**，代表按照插入顺序进行迭代；当然可以显式设置为` true`，代表以访问顺序进行迭代。      

### 例子   

```java   
LinkedHashMap<String, Integer> lmap = new LinkedHashMap<String, Integer>();
lmap.put("语文", 1);
lmap.put("数学", 2);
lmap.put("英语", 3);
lmap.put("历史", 4);
lmap.put("政治", 5);
lmap.put("地理", 6);
lmap.put("生物", 7);
lmap.put("化学", 8);
for(Entry<String, Integer> entry : lmap.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

运行结果    

```java  
语文: 1
数学: 2
英语: 3
历史: 4
政治: 5
地理: 6
生物: 7
化学: 8
```

#### LinkedHashMap的内部结构   

![](https://cloud.githubusercontent.com/assets/1736354/6981649/03eb9014-da38-11e4-9cbf-03d9c21f05f2.png)        


### 三个重点实现的函数   

在HashMap中提到了下面的定义：   

```java   
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

LinkedHashMap继承于HashMap，因此也重新实现了这3个函数，顾名思义这三个函数的作用分别是：节点访问后、节点插入后、节点移除后做一些事情。     

在LinkedHashMap中，上面3个函数都是为了保证双向链表中的节点次序或者双向链表容量所做的一些额外的事情，**目的就是保持双向链表中节点的顺序（head -> tail）是 eldest 到 youngest。**       

#### afterNodeAccess函数   

在进行put之后就算是对节点的访问了，那么这个时候就会更新链表，把最近访问的放到最后，保证链表。     

```java  
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 如果定义了accessOrder，那么就保证最近访问节点放到最后
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

#### afterNodeInsertion函数   

```java  
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    // 如果定义了移除规则，则执行相应的溢出
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
```

如果用户定义了`removeEldestEntry`的规则，那么便可以执行相应的移除操作，**而原始方法默认不需要移除**，如果要实现一个LRU的缓存可以这么定义`removeEldestEntry`          

```java   
@Override
protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
     return size() > size;
}
```

#### afterNodeRemoval函数     

这个函数是在移除节点后调用的，就是将节点从双向链表中删除。   

```java   
void afterNodeRemoval(Node<K,V> e) { // unlink
    // 从链表中移除节点
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

### put和get函数      

`put`函数在LinkedHashMap中未重新实现，只是实现了`afterNodeAccess`和`afterNodeInsertion`两个回调函数。get函数则重新实现并加入了`afterNodeAccess`来保证访问顺序，下面是get函数的具体实现：       

```java   
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

值得注意的是，在accessOrder模式下，只要执行get或者put等操作的时候，就会产生structural modification。     












  


   
