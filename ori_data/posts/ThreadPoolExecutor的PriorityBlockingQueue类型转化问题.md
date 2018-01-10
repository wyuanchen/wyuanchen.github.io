---
title: ThreadPoolExecutor的PriorityBlockingQueue类型转化问题
date: 2017-01-28 10:53:32
categories: Java
tags: 多线程
---
最近在使用ThreadPoolExecutor的时候碰到点问题,因为项目原因在使用ThreadPoolExecutor准备把BlockingQueue替换为PriorityBlockingQueue,从而实现对优先级任务处理的线程池,贴下代码先   

```java
public abstract class Event<T> implements Callable<T>,Comparable<Event>{
    ...
}
```


```java
ExecutorService executorService=new ThreadPoolExecutor(1,1,1,TimeUnit.SECONDS,new PriorityBlockingQueue());
```
然后当我进行以下操作的时候

```java
executorService.submit(new Event<Object>() {
            @Override
            public Object call() throws Exception {
                ...
                return null;
            }
        });
```

<!-- more -->

出现了如下错误

```java
java.lang.ClassCastException: java.util.concurrent.FutureTask cannot be cast to java.lang.Comparable
    at java.util.concurrent.PriorityBlockingQueue.siftUpComparable(PriorityBlockingQueue.java:357)
    at java.util.concurrent.PriorityBlockingQueue.offer(PriorityBlockingQueue.java:489)
    at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1361)
    at java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:134)
```
找到java.util.concurrent.PriorityBlockingQueue.siftUpComparable方法： 

```java
private static <T> void siftUpComparable(int k, T x, Object[] array) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = array[parent];
            if (key.compareTo((T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = key;
    }
```
是在`Comparable<? super T> key = (Comparable<? super T>) x;`上出现问题，根据`java.util.concurrent.FutureTask cannot be cast to java.lang.Comparable`知道x的类型是`java.util.concurrent.FutureTask`。现在看看`FutureTask`：

```
public class FutureTask<V> implements RunnableFuture<V>
public interface RunnableFuture<V> extends Runnable, Future<V>
```
可见FutureTask的确没有实现Comparable接口,但是我提交的`Event`是实现了`Comparable`接口的,究竟是因为什么原因导致其成为了FutureTask呢，结果在`ThreadPoolExecutor的submit(Callable<T> task)`找到原因，它是在`AbstractExecutorService`中实现的。

```java
public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```
重点在`newTaskFor`方法

```java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
```
**所以我提交的`Event`被转化为了`FutureTask`了，而`FutureTask`没有实现`Comparable`，所以才会报错**
现在解决的方法有:
- 用一个`ComparableFutureTask`继承`FutureTask`并实现`Comparable`接口，但也必须要override `ThreadPoolExecutor`的`newTaskFor`方法

**另外需要注意的是`PriorityBlockingQueue`的实现是一个最小堆.**

