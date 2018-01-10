---
title: ScheduledThreadPoolExecutor内部任务获取机制
date: 2017-02-18 22:17:50
categories: Java
tags: 多线程
---

去大概看了ScheduledThreadPoolExecutor的源码，发现内部是用一个队列来存储提交的任务，而使用的队列实质上是一个最小堆，具体的比较是任务执行时间与当前时间的延迟差，也就是说任务应该执行的时间点越近，就处于堆的越上端，也就是越靠近队列的头部。    

<!-- more -->

下面是向队列加入新任务的方法。     

```java
public boolean offer(Runnable x) {
   if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length)
            grow();
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
            siftUp(i, e);
        }
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
```

过程大概就是   

1. 查看容量是否够，不够则扩充   
2. 往堆插入新任务，然后根据任务的执行时间点到当前时间的差距来调整任务的位置     
3. 调整完位置后检测一下新任务是不是位于堆的顶点，也就是队列头部，如果是就发出信号唤醒其他等待获取任务的线程。       



下面是获取任务的方法   

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

过程大概为   

1. 查看队列是否有任务，如果没有就进入等待  
2. 如果有任务，再判断下任务是否能够马上执行，如果行就直接执行    
3. 如果任务还没到执行的时间，则查看当前线程是不是第一个等待线程，如果不是则直接进入等待   
4. 如果当前线程是第一个等待线程则更新leader，并且进入等待，等待的最长时间设置为距离任务能够执行的时间差        

如果在等待的过程中又有一个任务提交了，并且这个任务是可以马上执行的，则被唤醒的线程会重新获取队列首部的任务，也就是新提交的任务了。 