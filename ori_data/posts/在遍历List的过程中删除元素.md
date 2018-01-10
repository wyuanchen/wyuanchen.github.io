---
title: 在遍历List的过程中删除元素
date: 2017-01-28 10:51:33
categories: Java
tags: Java
---
### 首先遍历List的方式有以下几种:
- 普通for循环
- foreach循环
- 使用迭代器(Iterator)

```java
/**
     * 使用foeach循环 
     * 在循环过程中从List中删除元素以后，继续循环List时会抛出
     * ConcurrentModificationException 
     */
    public void listRemove() {
        List<Handler> handlers = this.handlers();
        for (Handler handler : handlers) {
            if (handler.getId() == 2)
                handlers.remove(handler);
        }
    }
```
  
```java
    /** 
     * 像这种foreach循环+break操作对List进行遍历删除，但删除之后马上就跳出的也
     * 不会出现异常 
     */  
    public void listRemoveBreak() {  
        List<Handler> handlers = this.Handlers();  
        for (Handler handler : handlers) {  
            if (handler.getId() == 2) {  
                handlers.remove(handler);  
                break;  
            }  
        }  
    }  
```

<!-- more -->

```java
    /** 
     * 这种遍历有可能会遗漏某个元素,因为删除元素后List的size在 
     * 变化，元素的索引也在变化，比如你循环到第2个元素的时候你把它删了， 
     * 接下来你去访问第3个元素，实际上访问到的是原先的第4个元素。当访问的元
     * 素 
     * 索引超过了当前的List的size后还会出现数组越界的异常，当然这里不会出现
     * 这种异常， 
     * 因为这里每遍历一次都重新拿了一次当前List的size。 
     */  
    public void listRemove2() {  
        List<handlers> handlers = this.getHandlers();  
        for (int i=0; i<handlers.size(); i++) {  
            if (handlers.get(i).getId() == 1) {  
                Handler handler = handlers.get(i);  
                handlers.remove(handler);  
            }  
        }  
    }  
```

```java
    /** 
     * 使用Iterator的方式也可以顺利删除和遍历 
     */  
    public void iteratorRemove() {  
        List<Handlers> handlers = this.getHandlers();  
        System.out.println(handlers);  
        Iterator<Handler> handlIter = handlers.iterator();  
        while (handlIter.hasNext()) {  
            Handler handler = handlIter.next();  
            if (handler.getId()  == 1)  
                handlIter.remove();//这里要使用Iterator的remove方法移除当前对象，如果使用List的remove方法，则同样会出现ConcurrentModificationException  
        }  
        System.out.println(handlers);  
    }  
```

### 如果是在遍历操作远多于可变操作的时候，还可以可以考虑CopyOnWriteArrayList

        这一般需要很大的开销，但是当遍历操作的数量大大超过可变操作的数量时，这种方法可能比其他替代方法更 有效。在不能或不想进行同步遍历，但又需要从并发线程中排除冲突时，它也很有用。“快照”风格的迭代器方法在创建迭代器时使用了对数组状态的引用。此数组在迭代器的生存期内不会更改，因此不可能发生冲突，并且迭代器保证不会抛出ConcurrentModificationException。创建迭代器以后，迭代器就不会反映列表的添加、移除或者更改。在迭代器上进行的元素更改操作（remove、set和add）不受支持。这些方法将抛出UnsupportedOperationException。允许使用所有元素，包括null。
