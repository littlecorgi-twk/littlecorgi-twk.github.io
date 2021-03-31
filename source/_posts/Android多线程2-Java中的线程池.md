---
title: Android多线程2--Java中的线程池
tags:
  - Java
  - 多线程
  - Android
categories: Java
comments: true
abbrlink: b39fd0ab
date: 2019-06-16 15:49:47
updated: 2019-06-16 15:49:47
copyright: true
---
# 简介
我们在写项目经常要用到多线程。但是线程的创建和摧毁都是较消耗资源和性能的，如果你每需要一个任务就新建一个线程，那可能会在线程的创建和摧毁上浪费掉很多资源。那如果我们让线程执行任务后不摧毁，接着执行下一个任务，这样是不是就能避免这种情况了。Java1.5中提供了`Executor`框架用于把任务的提交和执行解耦，任务的执行就交给`Runnable`或者`Callable`，而`Executor`框架用于处理任务。`Executor`中最核心的成员就是`ThreadPoolExecutor`，他就是线程池核心实现类。

<!-- More -->

# ThreadPoolExecutor
我们现在先来看下这个方法。
构造器：
```java
public ThreadPoolExecutor(int corePoolSize,
                        int maximumPoolSize,markdownlint
                        long keepAliveTime,
                        TimeUnit unit,
                        BlockingQueue<Runnable> workQueue,
                        ThreadFactory threadFactory,
                        RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
我们来看下这些参数：
* `corePoolSize`：核心线程数。线程池刚创建的时候，线程数量为0，只有任务提交的时候才会创建线程。如果当前线程数量小于`corePoolSize`，则创建新线程；如果等于或者大于，则不再创建。
* `maximumPoolSize`：线程池允许创建的最大线程数。当`workQueue`满了而且线程数小于`maximumPoolSize`时，线程池仍会创建新的线程。但是如果超过了`maximumPoolSize`时，则会抛出异常。
* `keepAliveTime`：非核心线程闲置的超过时间。当线程池中线程数量大于workQueue，如果一个线程的空闲时间大于keepAliveTime，则该线程会被回收。如果任务很多，并且每个任务 的执行事件很短，则可以调大keepAliveTime来提高线程的利用率。另外，如果设置 allowCoreThreadTimeOut属性为true时，keepAliveTime也会应用到核心线程上。
* `timeUnit`：keepAliveTime参数的时间单位。可选的单位有天(DAYS)、小时(HOURS)、分钟(MINUTES)、秒(SECONDS)、毫秒(MILLISECONDS)等。
* `workQueue`：任务队列。如果当前线程数大于`corePoolSize`，则将任务添加到此任务队列中。该任务 队列是`BlockingQueue`类型的，也就是阻塞队列。
* `threadFactory`：线程工厂。可以用线程工厂给每个创建出来的线程设置名字。一般情况下无须设置该参数。
* `RejectedExecutionHandler`：饱和策略。这是当任务队列和线程池都满了时所采取的应对策略，默认是AbordPolicy，表示无法处理新任务，并抛出`RejectedExecutionException`异常。此外还有3种策略，它们分别如下:
    1. `CallerRunsPolicy`：用调用者所在的线程来处理任务。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度。
    2. `DiscardPolicy`：不能执行的任务，并将该任务删除。
    3. `DiscardOldestPolicy`：丢弃队列最近的任务，并执行当前的任务。

# 线程池的处理流程
线程池的任务处理主要分为3个步骤
1. 提交任务后，线程池先判断线程数是否达到了核心线程数`corePoolSize`。如果未达到核心线程数，则创建核心线程处理任务；否则，就执行下一步操作。
2. 接着线程池判断任务队列是否满了。如果没满，则将任务添加到任务队列中；否则，就执行下一步操作。
3. 接着因为任务队列满了，线程池就判断线程数是否达到了最大线程数。如果未达到，则创建非核心线程处理任务；否则，就执行饱和策略，默认会抛出`RejectedExecutionException`异常。

![线程池执行流程](https://psb1j8lvg.bkt.clouddn.com/mweb/屏幕快照2019-06-16下午3.03.51.png)
1. 如果线程池中的线程数未达到核心线程数，则创建核心线程处理任务。
2. 如果线程数大于或者等于核心线程数，则将任务加入任务队列，线程池中的空闲线程会不断地从 任务队列中取出任务进行处理。
3. 如果任务队列满了，并且线程数没有达到最大线程数，则创建非核心线程去处理任务。
4. 如果线程数超过了最大线程数，则执行饱和策略。

# 线程池的种类
## CachedThreadPool：可缓存线程池
1. 线程数无限制
2. 有空闲线程则复用空闲线程，若无空闲线程则新建线程
3. 一定程序减少频繁创建/销毁线程，减少系统开销

创建源码：
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```

`CachedThreadPool`的`corePoolSize`为0，`maximumPoolSize`设置为`Integer.MAX_VALUE`，这意味着`CachedThreadPool`没有核心线程，非核心线程是无界的。`keepAliveTime`设置为60L，则空闲线程等待新任务 的最长时间为60s。在此用了阻塞队列`SynchronousQueue`，它是一个不存储元素的阻塞队列，每个插入操作必须等待另一个线程的移除操作，同样任何一个移除操作都等待另一个线程的插入操作。

## FixedThreadPool：定长线程池
1. 可控制线程最大并发数（同时执行的线程数）
2. 超出的线程会在队列中等待

创建源码：
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}
```

`FixedThreadPool`的`corePoolSize`和`maximumPoolSize`都设置为创建`FixedThreadPool`指定的参数`nThreads`，也就意味着`FixedThreadPool`只有核心线程，并且数量是固定的，没有非核心线程。`keepAliveTime`设置为0L意味着多余的线程会被立即终止。因为不会产生多余的线程，所以`keepAliveTime`是无效的参数。另外，任务队列采用了无界的阻塞队列`LinkedBlockingQueue`。

## ScheduledThreadPool：定长线程池
支持定时及周期性任务执行。

创建源码：
```java
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```
我们可以看到在创建源码中他跳转到了`ScheduledThreadPoolExecutor`的构造方法，我们继续看进去：
```java
public ScheduledThreadPoolExecutor(int corePoolSize,
                                    ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE,
            DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
            new DelayedWorkQueue(), threadFactory);
}
```

从上面的代码可以看出，`ScheduledThreadPoolExecutor`的构造方法最终调用的是`ThreadPoolExecutor`的构造方法。`corePoolSize`是传进来的固定数值，`maximumPoolSize`的值是`Integer.MAX_VALUE`。因为采用的`DelayedWorkQueue`是无界的，所以`maximumPoolSize`这个参数是无效的。

## SingleThreadExecutor：单线程化的线程池
1. 有且仅有一个工作线程执行任务
2. 所有任务按照指定顺序执行，即遵循队列的入队出队规则

创建源码：
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

`corePoolSize`和`maximumPoolSize`都为1，意味着`SingleThreadExecutor`只有一个核心线程，其他的参数都和`FixedThreadPool`一样，这里就不赘述了。