---
title: TreeMap原理学习
date: 2017-3-31 23:31:22
categories: Java
tags: 容器
---

## 性质

`TreeMap`底层是由红黑树实现的，所以满足红黑树的性质：

- 每个结点要么是红的，要么是黑的。  

- 根结点是黑的。  

- 每个叶结点（叶结点即指树尾端`NIL`指针或`NULL`结点）是黑的。  

- 如果一个结点是红的，那么它的俩个儿子都是黑的。  

- 对于任一结点而言，其到叶结点树尾端NIL指针的每一条路径都包含相同数目的黑结点。

红黑树，本质上来说就是一棵二叉查找树，但它在二叉查找树的基础上增加了着色和相关的性质使得红黑树相对平衡，从而保证了红黑树的查找、插入、删除的时间复杂度最坏为`O(log n)`。

<!-- more -->

下面是一棵红黑树的示意图：

![](http://upload-images.jianshu.io/upload_images/1637925-4973e62518192bcb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 内部结点

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;

    /**
     * Make a new cell with given key, value, and parent, and with
     * {@code null} child links, and BLACK color.
     */
    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
    
    public K getKey() {
        return key;
    }

    public V getValue() {
        return value;
    }

    public V setValue(V value) {
        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return valEquals(key,e.getKey()) && valEquals(value,e.getValue());
    }

    public int hashCode() {
        int keyHash = (key==null ? 0 : key.hashCode());
        int valueHash = (value==null ? 0 : value.hashCode());
        return keyHash ^ valueHash;
    }

    public String toString() {
        return key + "=" + value;
    }
}
```

## 旋转操作

### 左旋

![](http://upload-images.jianshu.io/upload_images/1637925-fbb109dbaa9b2c48.gif?imageMogr2/auto-orient/strip)

```java
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> r = p.right;
        p.right = r.left;
        if (r.left != null)
            r.left.parent = p;
        r.parent = p.parent;
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}
```

### 右旋

![](http://upload-images.jianshu.io/upload_images/1637925-185dea52a871f85e.gif?imageMogr2/auto-orient/strip)

```java
private void rotateRight(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```

## 插入

- 红黑树每次插入的新结点都是红色。

- 红黑树的插入相当于在二叉查找树插入的基础上，为了重新恢复平衡，继续做了插入修复操作。

```java
public V put(K key, V value) { 
    // 先以 t 保存链表的 root 结点
    Entry<K,V> t = root; 
    // 如果 t==null，表明是一个空链表，即该 TreeMap 里没有任何 Entry 
    if (t == null) 
    { 
        // 将新的 key-value 创建一个 Entry，并将该 Entry 作为 root 
        root = new Entry<K,V>(key, value, null); 
        // 设置该 Map 集合的 size 为 1，代表包含一个 Entry 
        size = 1; 
        // 记录修改次数为 1 
        modCount++; 
        return null; 
    } 
    int cmp; 
    Entry<K,V> parent; 
    Comparator<? super K> cpr = comparator; 
    // 如果比较器 cpr 不为 null，即表明采用定制排序
    if (cpr != null) 
    { 
        do { 
            // 使用 parent 上次循环后的 t 所引用的 Entry 
            parent = t; 
            // 拿新插入 key 和 t 的 key 进行比较
            cmp = cpr.compare(key, t.key); 
            // 如果新插入的 key 小于 t 的 key，t 等于 t 的左边结点
            if (cmp < 0) 
                t = t.left; 
            // 如果新插入的 key 大于 t 的 key，t 等于 t 的右边结点
            else if (cmp > 0) 
                t = t.right; 
            // 如果两个 key 相等，新的 value 覆盖原有的 value，
            // 并返回原有的 value 
            else 
                return t.setValue(value); 
        } while (t != null); 
    } 
    else 
    { 
        if (key == null) 
            throw new NullPointerException(); 
        Comparable<? super K> k = (Comparable<? super K>) key; 
        do { 
            // 使用 parent 上次循环后的 t 所引用的 Entry 
            parent = t; 
            // 拿新插入 key 和 t 的 key 进行比较
            cmp = k.compareTo(t.key); 
            // 如果新插入的 key 小于 t 的 key，t 等于 t 的左边结点
            if (cmp < 0) 
                t = t.left; 
            // 如果新插入的 key 大于 t 的 key，t 等于 t 的右边结点
            else if (cmp > 0) 
                t = t.right; 
            // 如果两个 key 相等，新的 value 覆盖原有的 value，
            // 并返回原有的 value 
            else 
                return t.setValue(value); 
        } while (t != null); 
    } 
    // 将新插入的结点作为 parent 结点的子结点
    Entry<K,V> e = new Entry<K,V>(key, value, parent); 
    // 如果新插入 key 小于 parent 的 key，则 e 作为 parent 的左子结点
    if (cmp < 0) 
        parent.left = e; 
    // 如果新插入 key 小于 parent 的 key，则 e 作为 parent 的右子结点
    else 
        parent.right = e; 
    // 修复红黑树
    fixAfterInsertion(e);                               
    size++; 
    modCount++; 
    return null; 
 }
```

### 插入后的平衡修复

新插入的结点都先被设置为红色。

无需修复的情况：

1. 如果插入的是根结点，因为原树是空树，所以直接把此结点涂为黑色。

2. 如果插入的结点的父结点是黑色，红黑树没有被破坏，所以此时也是什么也不做。


需要修复的情况：

1. 如果当前结点的父结点是红色且祖父结点的另一个子结点（叔叔结点）是红色

2. 当前结点的父结点是红色，叔叔结点是黑色，当前结点是其父结点的右子

3. 当前结点的父结点是红色，叔叔结点是黑色，当前结点是其父结点的左子


#### 修复情况1：当前结点的父结点是红色且祖父结点的另一个子结点（叔叔结点）是红色

此时父结点的父结点一定存在，否则插入前就已不是红黑树。   
与此同时，又分为父结点是祖父结点的左子还是右子，对于对称性，我们只要解开一个方向就可以了。

在此，我们只考虑父结点为祖父左子的情况。   
同时，还可以分为当前结点是其父结点的左子还是右子，但是处理方式是一样的。我们将此归为同一类。   

##### 解决方案

将当前结点的父结点和叔叔结点涂黑，祖父结点涂红，把当前结点指向祖父结点，从新的当前结点重新开始算法。

例子：

![](http://upload-images.jianshu.io/upload_images/1637925-1008c055cb42f52e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

新插入结点N的父结点P和叔叔结点U都是红色。  

方法是：将祖父结点G设置为红色，父结点P和叔叔结点U设置为黑色，这时候就看似平衡了。   

但是，如果祖父结点G的父结点也是红色，这时候又违背规则（两个红色结点不能相邻），    
方法是：将GPUN这一组看成一个新的结点，按照前面的方案递归；  

如果一直递归到根结点为红就违反规则（根结点必须为黑色）了，   
方法是直接将根结点设置为黑色（两个连续黑色是没问题的）


#### 修复情况2：当前结点的父结点是红色,叔叔结点是黑色或者缺少，当前结点是其父结点的右子   

#### 解决方案   

当前结点的父结点做为新的当前结点，以新当前结点为支点左旋。

例子：

![](http://upload-images.jianshu.io/upload_images/1637925-a1989d9144ce89b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

左旋父结点P。左旋后N和P角色互换。

但是P和N还是连续的两个红色结点，还没有平衡，所以进入第三种修复情况。


#### 修复情况3：当前结点的父结点是红色,叔叔结点是黑色或者缺少，当前结点是其父结点的左子

##### 解决方案

父结点变为黑色，祖父结点变为红色，在**祖父结点为支点**右旋。   
最后，把根结点涂为黑色，整棵红黑树便重新恢复了平衡

例子：

![](http://upload-images.jianshu.io/upload_images/1637925-877db12aad3f2817.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

右旋祖父结点G，然后将P设置为黑色，G设置为红色，达到平衡。   
此时父结点P是黑色，所有不用担心P的父结点是红色。

**上面讨论的3种情况都是当前结点N的父结点P是祖父结点G的左孩子，所以如果父结点P是祖父结点G的右孩子的情形是相应的一种对称，只需要把相应的左旋和右旋对调即可**

### `TreeMap`的插入修复代码

```java
// 插入结点后修复红黑树
 private void fixAfterInsertion(Entry<K,V> x) 
 { 
    x.color = RED; 
    // 直到 x 结点的父结点不是根，且 x 的父结点不是红色
    while (x != null && x != root 
        && x.parent.color == RED) 
    { 
        // 如果 x 的父结点是其父结点的左子结点
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) 
        { 
            // 获取 x 的父结点的兄弟结点
            Entry<K,V> y = rightOf(parentOf(parentOf(x))); 
            // 如果 x 的父结点的兄弟结点是红色
            if (colorOf(y) == RED) 
            { 
                // 将 x 的父结点设为黑色
                setColor(parentOf(x), BLACK); 
                // 将 x 的父结点的兄弟结点设为黑色
                setColor(y, BLACK); 
                // 将 x 的父结点的父结点设为红色
                setColor(parentOf(parentOf(x)), RED); 
                x = parentOf(parentOf(x)); 
            } 
            // 如果 x 的父结点的兄弟结点是黑色
            else 
            { 
                // 如果 x 是其父结点的右子结点
                if (x == rightOf(parentOf(x))) 
                { 
                    // 将 x 的父结点设为 x 
                    x = parentOf(x); 
                    rotateLeft(x); 
                } 
                // 把 x 的父结点设为黑色
                setColor(parentOf(x), BLACK); 
                // 把 x 的父结点的父结点设为红色
                setColor(parentOf(parentOf(x)), RED); 
                rotateRight(parentOf(parentOf(x))); 
            } 
        } 
        // 如果 x 的父结点是其父结点的右子结点
        else 
        { 
            // 获取 x 的父结点的兄弟结点
            Entry<K,V> y = leftOf(parentOf(parentOf(x))); 
            // 如果 x 的父结点的兄弟结点是红色
            if (colorOf(y) == RED) 
            { 
                // 将 x 的父结点设为黑色。
                setColor(parentOf(x), BLACK); 
                // 将 x 的父结点的兄弟结点设为黑色
                setColor(y, BLACK); 
                // 将 x 的父结点的父结点设为红色
                setColor(parentOf(parentOf(x)), RED); 
                // 将 x 设为 x 的父结点的结点
                x = parentOf(parentOf(x)); 
            } 
            // 如果 x 的父结点的兄弟结点是黑色
            else 
            { 
                // 如果 x 是其父结点的左子结点
                if (x == leftOf(parentOf(x))) 
                { 
                    // 将 x 的父结点设为 x 
                    x = parentOf(x); 
                    rotateRight(x); 
                } 
                // 把 x 的父结点设为黑色
                setColor(parentOf(x), BLACK); 
                // 把 x 的父结点的父结点设为红色
                setColor(parentOf(parentOf(x)), RED); 
                rotateLeft(parentOf(parentOf(x))); 
            } 
        } 
    } 
    // 将根结点设为黑色
    root.color = BLACK; 
 }
```


## 删除

当程序从排序二叉树中删除一个结点之后，为了让它依然保持为排序二叉树，程序必须对该排序二叉树进行维护。维护可分为如下几种情况：

- 被删除的结点是叶子结点，则只需将它从其父结点中删除即可。

- 被删除结点 p 只有左子树，将 p 的左子树 pL 添加成 p 的父结点的左子树即可；被删除结点 p 只有右子树，将 p 的右子树 pR 添加成 p 的父结点的右子树即可。

- 若被删除结点 p 的左、右子树均非空，以 p 结点的中序后继替代 p 所指结点，然后再从原排序二叉树中删去中序后继结点即可


```java
private void deleteEntry(Entry<K,V> p) 
 { 
    modCount++; 
    size--; 
    // 如果被删除结点的左子树、右子树都不为空
    if (p.left != null && p.right != null) 
    { 
        // 用 p 结点的中序后继结点代替 p 结点
        Entry<K,V> s = successor (p); 
        p.key = s.key; 
        p.value = s.value; 
        p = s; 
    } 
    // 如果 p 结点的左结点存在，replacement 代表左结点；否则代表右结点。
    Entry<K,V> replacement = (p.left != null ? p.left : p.right); 
    if (replacement != null) 
    { 
        replacement.parent = p.parent; 
        // 如果 p 没有父结点，则 replacemment 变成父结点
        if (p.parent == null) 
            root = replacement; 
        // 如果 p 结点是其父结点的左子结点
        else if (p == p.parent.left) 
            p.parent.left  = replacement; 
        // 如果 p 结点是其父结点的右子结点
        else 
            p.parent.right = replacement; 
        p.left = p.right = p.parent = null; 
        // 修复红黑树
        if (p.color == BLACK) 
            fixAfterDeletion(replacement);       // ①
    } 
    // 如果 p 结点没有父结点
    else if (p.parent == null) 
    { 
        root = null; 
    } 
    else 
    { 
        if (p.color == BLACK) 
            // 修复红黑树
            fixAfterDeletion(p);                 // ②
        if (p.parent != null) 
        { 
            // 如果 p 是其父结点的左子结点
            if (p == p.parent.left) 
                p.parent.left = null; 
            // 如果 p 是其父结点的右子结点
            else if (p == p.parent.right) 
                p.parent.right = null; 
            p.parent = null; 
        } 
    } 
 }
```


### 删除后的平衡修复
 
无需修复的情况：

- 如果删除的是红色结点，那么原红黑树的性质依旧保持，此时不用做修正操作

 
如果删除的结点是黑色的话则需要进行修复

修复的情况有：

1. 当前结点为黑色，兄弟结点为红色

2. 当前结点为黑色，兄弟结点为黑色，其兄弟结点的两个孩子节点都是黑色，父结点为任意颜色

3. 当前结点为黑色，兄弟结点为黑色，且兄弟结点的左孩子为红色，右孩子为黑色，父结点为任意颜色

4. 当前结点为黑色，兄弟结点为黑色，且兄弟结点的右孩子为红色，左孩子为任意颜色，父结点为任意颜色


#### 情况1：当前结点为黑色，兄弟结点为红色
 
因为兄弟结点为红色，所以父结点肯定为黑色。

##### 解决方案

交换兄弟结点和父结点的颜色，然后左旋父结点。 
调整还未平衡，进入情况2。

例子：

![](http://upload-images.jianshu.io/upload_images/1637925-1a0af4d0f585cd9c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

交换P和B的颜色，左旋父节点P。此时并未完成平衡


#### 情况2：当前结点为黑色，兄弟结点为黑色，其兄弟结点的两个孩子节点都是黑色，父结点为任意颜色

##### 解决方案

把兄弟结点变为红色，然后把当前结点设为父结点，继续查看是否需要调整。

例子1：

![](http://upload-images.jianshu.io/upload_images/1637925-be903bf2939fb2db.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

N的父节点P是黑色，且兄弟节点B和它的两个孩子节点也都是黑色。  

将N的兄弟节点B改为红色，这样从P出发到叶子节点的路径都包含了相同的黑色节点，但是，对于节点P这个子树，P的父节点G到P的叶子节点路径上的黑色节点就少了一个，此时需要将P整体看做一个节点，继续调整


例子2：

![](http://upload-images.jianshu.io/upload_images/1637925-50e11d926d333e30.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

N的父节点P为红色，兄弟节点B和它的两个孩子节点也都是黑色。

此时只需要交换P和B的颜色，将P改为黑色，B改为红色，则可到达平衡。这相当于既然节点N路径少了一个黑色节点，那么B路径也少一个黑色节点，这两个路径达到平衡，为了防止P路径少一个黑色节点，将P节点置黑，则达到最终平衡。


#### 情况3：当前结点为黑色，兄弟结点为黑色，且兄弟结点的左孩子为红色，右孩子为黑色，父结点为任意颜色

##### 解决方案

把兄弟结点颜色变红，兄弟结点的左孩子变黑，然后右旋兄弟结点，进入情况4。

例子：

![](http://upload-images.jianshu.io/upload_images/1637925-a8487108d0214965.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

N的兄弟节点B是黑色，B的左孩子节点BL是红色，B的右孩子节点BR是黑色，P为任意颜色。

交换B和BL的颜色，右旋节点B。此时N子树路径并没有增加黑色节点，也就是没有达到平衡，此时进入情况4。

#### 情况4：当前结点为黑色，兄弟结点为黑色，且兄弟结点的右孩子为红色，左孩子为任意颜色，父结点为任意颜色

##### 解决方案

兄弟结点变为父结点的颜色，兄弟结点的右孩子变为黑色，父结点变为黑色，左旋父结点，这个时候已经达到平衡。

例子：

![](http://upload-images.jianshu.io/upload_images/1637925-54d3ff73d1eba579.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

N的兄弟节点B是黑色，B的右孩子节点BR是红色，B的左孩子节点BL任意颜色，P任意颜色。  

BR变为黑色，P变为黑色，B变为P的颜色；左旋节点P。首先给N路径增加一个黑色节点P，P原位置上的颜色不变；S路径少了一个黑色节点，于是将BR改为黑色，最终达到了平衡。


### `TreeMap`的删除修复代码

```java
// 删除节点后修复红黑树
 private void fixAfterDeletion(Entry<K,V> x) 
 { 
    // 直到 x 不是根节点，且 x 的颜色是黑色
    while (x != root && colorOf(x) == BLACK) 
    { 
        // 如果 x 是其父节点的左子节点
        if (x == leftOf(parentOf(x))) 
        { 
            // 获取 x 节点的兄弟节点
            Entry<K,V> sib = rightOf(parentOf(x)); 
            // 如果 sib 节点是红色
            if (colorOf(sib) == RED) 
            { 
                // 将 sib 节点设为黑色
                setColor(sib, BLACK); 
                // 将 x 的父节点设为红色
                setColor(parentOf(x), RED); 
                rotateLeft(parentOf(x)); 
                // 再次将 sib 设为 x 的父节点的右子节点
                sib = rightOf(parentOf(x)); 
            } 
            // 如果 sib 的两个子节点都是黑色
            if (colorOf(leftOf(sib)) == BLACK 
                && colorOf(rightOf(sib)) == BLACK) 
            { 
                // 将 sib 设为红色
                setColor(sib, RED); 
                // 让 x 等于 x 的父节点
                x = parentOf(x); 
            } 
            else 
            { 
                // 如果 sib 的只有右子节点是黑色
                if (colorOf(rightOf(sib)) == BLACK) 
                { 
                    // 将 sib 的左子节点也设为黑色
                    setColor(leftOf(sib), BLACK); 
                    // 将 sib 设为红色
                    setColor(sib, RED); 
                    rotateRight(sib); 
                    sib = rightOf(parentOf(x)); 
                } 
                // 设置 sib 的颜色与 x 的父节点的颜色相同
                setColor(sib, colorOf(parentOf(x))); 
                // 将 x 的父节点设为黑色
                setColor(parentOf(x), BLACK); 
                // 将 sib 的右子节点设为黑色
                setColor(rightOf(sib), BLACK); 
                rotateLeft(parentOf(x)); 
                x = root; 
            } 
        } 
        // 如果 x 是其父节点的右子节点
        else 
        { 
            // 获取 x 节点的兄弟节点
            Entry<K,V> sib = leftOf(parentOf(x)); 
            // 如果 sib 的颜色是红色
            if (colorOf(sib) == RED) 
            { 
                // 将 sib 的颜色设为黑色
                setColor(sib, BLACK); 
                // 将 sib 的父节点设为红色
                setColor(parentOf(x), RED); 
                rotateRight(parentOf(x)); 
                sib = leftOf(parentOf(x)); 
            } 
            // 如果 sib 的两个子节点都是黑色
            if (colorOf(rightOf(sib)) == BLACK 
                && colorOf(leftOf(sib)) == BLACK) 
            { 
                // 将 sib 设为红色
                setColor(sib, RED); 
                // 让 x 等于 x 的父节点
                x = parentOf(x); 
            } 
            else 
            { 
                // 如果 sib 只有左子节点是黑色
                if (colorOf(leftOf(sib)) == BLACK) 
                { 
                    // 将 sib 的右子节点也设为黑色
                    setColor(rightOf(sib), BLACK); 
                    // 将 sib 设为红色
                    setColor(sib, RED); 
                    rotateLeft(sib); 
                    sib = leftOf(parentOf(x)); 
                } 
                // 将 sib 的颜色设为与 x 的父节点颜色相同
                setColor(sib, colorOf(parentOf(x))); 
                // 将 x 的父节点设为黑色
                setColor(parentOf(x), BLACK); 
                // 将 sib 的左子节点设为黑色
                setColor(leftOf(sib), BLACK); 
                rotateRight(parentOf(x)); 
                x = root; 
            } 
        } 
    } 
    setColor(x, BLACK); 
 }
```


#### 附加资料

- [通过分析 JDK 源代码研究 TreeMap 红黑树算法实现](https://www.ibm.com/developerworks/cn/java/j-lo-tree/)

- [教你透彻了解红黑树](https://github.com/julycoding/The-Art-Of-Programming-By-July/blob/master/ebook/zh/03.01.md)

- [Java集合干货系列-（四）TreeMap源码解析](http://www.jianshu.com/p/210c2f4ca130)

