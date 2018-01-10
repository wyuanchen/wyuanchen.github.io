---
title: WeakHashMap原理学习
date: 2017-2-13 20:31:22
categories: Java
tags: 容器
---

- WeakHashMap，此种Map的特点是，当除了自身有对key的引用外，此key没有其他引用那么此map会自动丢弃此值   


## 内部Entry构造  

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
        
        ...

}
```

请注意它构造父类的语句：`super(key, queue);`，传入的是`key`，因此`key`才是进行弱引用的，`value`是直接强引用关联在`this.value`之中  

<!-- more -->


## 引用队列（Reference Quene）

一旦弱引用返回null值，那么其指向的对象（即Widget）就变成了垃圾，这个弱引用对象(即weakWidget)也就没有用了。这通常意味着要进行一定方式的清理（cleanup）。

WeakHashmap将会移除一些死亡的的entry，避免持有过多死的弱引用    

`ReferenceQuene`能够轻易的追踪这些死掉的弱引用。可以讲ReferenceQuene传入WeakHashmap的构造方法（constructor）中，这样，一旦这个弱引用指向的对象成为垃圾，这个弱引用将加入ReferenceQuene中。

## 清理无效的项目   

- WeakHashMap是主要通过`expungeStaleEntries`这个函数的来实现移除其内部不用的条目从而达到的自动释放内存的目的的.基本上只要对WeakHashMap的内容进行访问就会调用这个函数，从而达到清除其内部不在为外部引用的条目。

- `expungeStaleEntries`方法只有在以下方法中才被调用：

    - `size()`

    - `resize(int newCapacity)`

    - `private Entry<K,V>[] getTable()`

    而`getTable`方法在很多方法中被调用，基本上对`WeakHashMap`的每一次访问都会调用到，例如：  

    - `public V get(Object key) `

    - `Entry<K,V> getEntry(Object key)`  

    - `public V put(K key, V value) `

    - `public boolean containsValue(Object value)`

    - `private boolean containsNullValue() `


```java
/**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```


