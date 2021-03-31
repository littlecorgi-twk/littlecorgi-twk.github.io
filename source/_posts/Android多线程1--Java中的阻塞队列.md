---
title: Android多线程1--Java中的阻塞队列
tags:
  - Java
  - 多线程
  - Android
categories: Java
comments: true
abbrlink: 63ab42b7
date: 2019-06-09 13:39:10
updated: 2019-06-16 15:47:03
copyright: true
---

# 阻塞队列

## 前言
在谈论阻塞队列之前我们先看下操作系统多线程部分一个经典的例子——生产者和消费者问题：
>现在有两个进程，一个是生产者一个是消费者，还有一个线程缓冲区。生产者主要作用就是向缓冲区中添加数据，消费者就是从缓冲区中取出数据。这个问题的核心就是如何确保生产者不会在缓冲区满了的时候还往其中添加元素，消费者不会在缓冲区空了的时候还要求取出数据。

<!-- More -->
关于这个问题的解决办法我们以后再说，我们现在主要讨论线程缓冲区——阻塞队列。
## 阻塞队列简介
阻塞队列就是队列，只是在一般的队列上添加了两个条件：
1. 当队列满了的时候不允许再添加数据
2. 当队列空了的时候不允许从中取数据

在Java中，阻塞队列是通过`BlockingQueue`来实现的，`BlockingQueue`是`Java.util.concurrent`包下一个重要的数据结构。

## BlockingQueue的操作方法
| 方法 | 抛异常 | 返回特定值 | 阻塞 | 超时 |
| :---: | :---: | :---: | :---: | :---: |
|插入| add(E e) | offer(E e) | put(E e) | offer(E e, long timeout, TimeUnit unit) |
|移除| remove() | poll() | take() | poll(time, unit) |
|检查| element() | peek() | 不可用 | 不可用 |

> 解释：
> 1. 抛异常：如果操作无法执行，则抛出一个异常
> 2. 特定值：如果操作无法执行，则返回一个特定的值
> 3. 阻塞： 如果操作无法执行，则方法调用被阻塞，直到可以执行
> 4. 超时：如果操作无法执行，则方法调用被阻塞，直到可以执行或者超过限定的时间。返回一个特定值以告知该操作是否成功(典型的是true / false)。

## Java中的各种阻塞队列
`Java`基于`BlockingQueue`给开发者提供了7个阻塞队列：
1. `ArrayBlockingQueue`：基于数组的有界阻塞队列。有界就意味着他有一个最大限度，所存储的线程的数量不能超过这个限定值。你也可以在对其初始化的时候给定这个限定值。但是由于它是基于数组所以他和数组一样，在初始化的时候限定了这个大小以后就不能改变。
2. `LinkedBlockingQueue`：基于链表的阻塞队列。它内部以一个链式结构(链接节点)对其元素进行存储。如果需要的话，这一链式结构可以选择一个上限。如果没有定义上限，将使用`Integer.MAX_VALUE`作为上限。`LinkedBlockingQueue`内部以FIFO(先进先出)的顺序对元素进行存储。队列中的头元素在所有元素之中是放入时间最久的那个，而尾元素则是最短的那个。由于默认是无上限的，所以在使用他的时候，如果生产者的速度大于消费者的速度，系统内存可能会被耗尽。所以使用他一定要设置初值。
3. `PriorityBlockingQueue`：支持优先级的无界队列。默认情况按照自然顺序生序排列，你可以重写`compateTo()`方法来制定元素按规定排序。
4. `DelayQueue`：支持延时获取元素的无界阻塞队列。队列中的元素必须实现`Delayed`接口。
5. `SynchromousQueue`：是一个特殊的队列。他不能存储任何元素，他的每一次插入操作必须等待另一个线程相应的删除操作，反之亦然。
6. `LinkedTransferQueue`：基于链表的无界阻塞`TransferQueue`队列。相对于其他队列，他多了`transfer(E e)`、`tryTransfer(E e)` 和 `tryTransfer(E e, long timeout, TimeUnit unit)`方法。
7. `LinkedBlockingDeque`：是一个链表结构的双向阻塞队列。可在两端入队出对。所以当多线程入队时，减少了一半的竞争。

## 阻塞队列实现原理
下面我们以`ArrayBlockingQueue`源码为例，来看下阻塞队列实现原理：
### 定义
首先就是一堆变量的定义：
```java
/** The queued items */
final Object[] items;

/** items index for next take, poll, peek or remove */
int takeIndex;

/** items index for next put, offer, or add */
int putIndex;

/** Number of elements in the queue */
int count;

/*
 * Concurrency control uses the classic two-condition algorithm
 * found in any textbook.
 */

/** Main lock guarding all access */
final ReentrantLock lock;

/** Condition for waiting takes */
private
final Condition notEmpty;

/** Condition for waiting puts */
private
final Condition notFull;
```
`items`是存储队列元素的数组，`takeIndex`和`putIndex`分别是取数据和存数据的索引，`count`是队列中元素个数，`lock`为看一个可重入锁，`notEmpty`和`notFull`均为等待条件，由`loc`k创建。

### 构造器
接下来看下它的构造器
```java
public ArrayBlockingQueue(int capacity) {
}

public ArrayBlockingQueue(int capacity, boolean fair) { 
}

public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
}
```
构造器有三个重载的版本，第一个构造器只有一个参数用来指定容量，第二个构造器多了一个参数来指定访问策略，第三个构造器又多了一个参数可以指定用另外一个集合进行初始化。

### 数据的添加
接下来我们看看`BlockingQueue`的三个插入的方法：`put()`、`add()`和`offer()`：
* `put()` 方法：队列满，会阻塞调用存储元素的线程

```java
public void put(E e) throws InterruptedException {
    // 先检查e是不是空，如果空则抛异常
    Objects.requireNonNull(e);
    // 获取一个重入锁lock
    final ReentrantLock lock = this.lock;
    // 加锁，保证调用put方法的时候只有1个线程
    lock.lockInterruptibly();
    try {
    // 如果线程中的元素数量是否等于当前数组的长度，如果相等则调用await方法等待，如果不相等则enqueue方法插入元素
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
    // 解锁
        lock.unlock();
    }
}
```
* `add()`方法：实际上调用了`offer()`方法

```java
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```
* `offer()`方法：成功返回true，失败返回false

```java
public boolean offer(E e) {
    // 检查e是否为空
    Objects.requireNonNull(e);
    // 获取重入锁lock
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 如果如果线程中的元素数量是否等于当前数组的长度，如果相等则调返回false，如果不相等则enqueue方法插入元素并返回true
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        // 解锁
        lock.unlock();
    }
}
```

以上三个方法都调用了`enqueue()`方法。下面我们就来看看这个方法：
```java
/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E e) {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = e;
    if (++putIndex == items.length) 
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```
先获取元素数组`items`，然后添加`putIndex`上，如果`++putIndex`等于`items`的长度，则证明当前这个`items`所有元素都添加进了，就让`putIndex`等于0.然后调用`notEmpty.signal()`方法唤醒正在获取元素的线程，让他们从队列中取数据。

### 数据的取出
ArrayBlockingQueue的取数据方法总共也有三个方法：`poll()`、`take()`和`remove()`
* `poll()`方法：获取元素，存在返回元素e,不存在返回null
 
```java
public E poll() {
    // 获取重入锁lock
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 如果元素数量等于0就返回null，否则调用dequeue()方法
        return (count == 0) ? null : dequeue();
    } finally {
        // 解锁
        lock.unlock();
    }
}
```
* `take()`方法：取元素。如果队列为空,则会阻塞调用获取元素的线程

```java
public E take() throws InterruptedException {
    // 获取重入锁lock
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lockInterruptibly();
    try {
        // 如果线程中的元素数量是否等于0，如果相等则调用await方法等待，如果不相等则dequeue方法删除元素
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        // 解锁
        lock.unlock();
    }
}
```
* `remove()`方法：取元素，它是取特定的那个元素

```java
public boolean remove(Object o) {
    // 判断o是否为空
    if (o == null) return false;
    // 获取重入锁lock
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 如果元素数量大于0，则获取items，然后便利元素，判断o是其中的哪个，然后删除那个
        if (count > 0) {
            final Object[] items = this.items;
            for (int i = takeIndex, end = putIndex, to = (i < end) ? end : items.length; ; i = 0, to = end) {
                for (; i < to; i++)
                    if (o.equals(items[i])) {
                        removeAt(i);
                        return true;
                    }
                if (to == end) break;
            }
        }
        return false;
    } finally {
        // 解锁
        lock.unlock();
    }
}
```
`poll()`和`take()`两个方法都调用了`dequeue()`方法，我们就看下`dequeue()`是如何来实现的：
```java
/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E e = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return e;
}
```
和上面的`enqueue()`方法类似，再次就不再赘述。

## 阻塞队列的应用
前面我说过，阻塞队列主要用在生产者和消费者模式中，那下面我们就来写一个简单的小demo
>>这段代码来自刘望舒所著《Android进阶之光》

如果不用阻塞队列：
```java
import java.util.PriorityQueue;

public class Test {
    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<>(queueSize);

    public static void main(String[] args) {
        Test test = new Test();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
        producer.start();
        consumer.start();
    }

    class Consumer extends Thread {
        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    while (queue.size() == 0) {
                        try {
                            System.out.println("队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    // 每次移走队首元素
                    queue.poll();
                    queue.notify();
                }
            }
        }
    }

    private class Producer extends Thread {
        @Override
        public void run() {
            while (true) {
                synchronized (queue) {
                    try {
                        System.out.println("队列满，等待有空余空间");
                        queue.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                        queue.notify();
                    }
                }
                // 每次插入一个元素
                queue.offer(1);
                queue.notify();
            }
        }
    }
}
```

使用阻塞队列`ArrayBlockingQueue`：
```java
import java.util.concurrent.ArrayBlockingQueue;

public class BlockingQueueTest {
    private int queueSize = 10;
    private ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<>(queueSize);

    public static void main(String[] args) {
        BlockingQueueTest test = new BlockingQueueTest();
        BlockingQueueTest.Producer producer = test.new Producer();
        BlockingQueueTest.Consumer consumer = test.new Consumer();
        producer.start();
        consumer.start();
    }

    class Consumer extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    queue.take();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private class Producer extends Thread {
        @Override
        public void run() {
            while (true) {
                try {
                    queue.put(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```