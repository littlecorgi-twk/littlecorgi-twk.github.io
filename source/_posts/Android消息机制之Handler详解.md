---
title: Android消息机制之Handler
categories:
  - Android
tags:
  - Android
  - Handler
  - ThreadLocal
  - 源码
comments: true
copyright: true
feature_img: false
abbrlink: 21eb5c30
date: 2019-11-17 14:43:10
updated: 2019-11-17 14:43:10
cover: "https://cdn.littlecorgi.top/blog/Keynote_Blog_FINAL.png"
keywords:
description:
---
# 简介
作为一个Android开发者，Handler的大名你一定听过。做Android开发肯定离不开Handler，它通常被我们用来连通主线程和子线程。

可以说只要有异步的地方一定有Handler。

那么，你了解过为什么Handler能连通主线程和子线程吗，也就是说，你了解过Handler背后的原理吗？

就让本文带你了解。

# Handler的基本用法
按照惯例，我们首先看下Handler的一般用法：
```java
Handler mHandler = new Handler() {
    @Override
    public void handleMessage(final Message msg) {
        // 在这里接受并处理消息
    }
};

//发送消息
mHandler.sendMessage(message);
mHandler.post(runnable);
```

也就是创建一个Handler对象，并重写handlerMessage方法，然后在需要的时候调用sendMessage方法传入一个message对象，或者调用post方法传入一个runnable对象。

那么我们从他的构造方法开始看起吧。

# 从Handler构造方法入手
```java
public Handler() {
    // ->> 分析1.1
    this(null, false);
}
```

## 分析1.1 Handler # 构造方法
```java
/**
 * 分析1.1：实际上调用的又是另一个构造方法
 */
public Handler(@Nullable Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        // 这个变量字面意思是找到内存泄漏，
        // 但是整个Handler中除了这块以及定义这个变量为false之外，
        // 就没有其他地方使用到过这个变量了，所以这块我也不太懂这个变量是怎么变为true的
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    // ->> 分析1.2
    mLooper = Looper.myLooper();
    // ->> 分析1.3
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    // 获取对象的mQueue属性，mQueue就是MessageQueue
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

## 分析1.2 Looper # myLooper()
```java
/**
 * 分析1.2：Looper.myLopper()
 * 实际上调用的是ThreadLocal的get方法，然后返回该线程的Looper对象
 */
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
ThreadLocal是一个数据存储类，他的特别之处在于他是线程间独立的，也就是说这个线程放入到ThreadLocal的数据，其它线程是获取不到的。sThreadLocal就是获取当前线程的Looper对象。详细可见我之前的博客。

## 分析1.3 Looper # prepare()
为什么说如果mLooper为空就抛异常呢，这是因为一个Handler必须和一个Looper绑定，并且要先初始化Looper才能去初始化Handler。初始化Looper就通过Looper的prepare方法。
```java
/**
 * 分析1.3：Looper.prepare()
 */
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // ->> 分析1.4
    sThreadLocal.set(new Looper(quitAllowed));
}
```
这个方法和myLooper方法类似，就是通过ThreadLocal设置Looper。

## 分析1.4 Looper # 构造方法
```java
/**
 * 分析1.4：Looper的构造方法
 */
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
Looper的构造方法是私有的，也就是说外界并不能直接创建Looper方法，必须通过Looper.prepare方法来创建Looper对象，这样也就方便了保证一个线程只能有一个Looper对象。

在构造方法中，我们创建了一个MessageQueue对象。这个对象就是用来存放消息的消息队列。关于MessageQueue我们后面再来讲。

# 消息是如何存放的
上面就是Handler和Looper的一个构造方式，构造完成后，我们就要关注Handler是如何把消息放入MessageQueue中的，以及是如何从MessageQueue中取出的。

上面我们在讲Handler的基本用法的时候，讲到过sendMessage方法和post方法。我们先来看下这两个方法：

## Handler # sendMessage() & post()
其实这块先不上具体的方法实现，先说一下他们的方法调用流程：
```java
sendMessage()
    -> sendMessageDelayed()
        -> sendMessageAtTime()
            -> enqueueMessage()

post()
    -> sendMessageDelayed()
        -> sendMessageAtTime()
            -> enqueueMessage()
```

可以看到，这两个方法都会调用立马调用sendMessageDelayed():
```java
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    // 默认都传入0
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    // ->> 分析2.1
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

## 分析2.1 Handler # sendMessageAtTime()
```java
/**
 * 分析2.1：sendMessageAtTime()
 * 传入参数：
 *     msg：需要传递的消息
 *     uptimeMillis：更新时间，也就是系统从开机到目前经过的时候 + delayMillis(也就是0)
 */
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    // ->> 分析2.2
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

## 分析2.2 Handler # enqueueMessage()
```java
/**
 * 分析2.2：enqueueMessage()
 */
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    // ->> 这块就将消息存放到MessageQueue队列中去
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

那么消息就成功的从Handler中被存入了MessageQueue，那么消息什么时候从MessageQueue中被调出来呢?

# 消息是如何取出的
## Looper # loop()
Looper里面有一个loop方法，我们可以看下这个方法:
```java
public static void loop() {
    // 获取当前线程的Looper对象
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    // 获取当前线程的MessageQueue
    final MessageQueue queue = me.mQueue;

    // ...省略中间代码

    // 死循环
    for (;;) {
        // 从消息队列中取出一个消息
        // ->> 分析3.1
        Message msg = queue.next(); // might block
        // 如果取出的消息为空，就退出。
        // 但是我之前说这个循环是个死循环？
        // 这是因为虽然会有这个if可能会导致循环退出，但是通过next方法取出消息的时候，如果没有消息了，next方法会阻塞线程，直到有新的消息进来
        // 这也就意味着，一般正常的遍历MessageQueue情况下，是不会有msg==null的。
        // 但是如果你调用了Looper的quit或者quitSafely方法，这个时候next会返回null，就会退出循环。
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // ...省略中间代码
        
        try {
            // 传给Handler去处理消息
            // ->> 分析3.2
            msg.target.dispatchMessage(msg);
            if (observer != null) {
                observer.messageDispatched(token, msg);
            }
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } catch (Exception exception) {
            // ...省略中间代码
        } finally {
            // ...省略中间代码
        }
        
        // ...省略中间代码
    }
}
```

## 分析3.1 MessageQueue # next()
```java
/**
 * 分析3.1 MessageQueue.next()
 * 作用：
 *     用于取出消息队列中的下一个消息
 * PS：
 *     看到这个next你或许就应该懂了，其实这个MessageQueue并不是咱们顾名思义以为的那个Queue，其实他的实现是一个链表 
 */
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    // 如果Looper已经退出（调用了dispose方法后mPtr=0）
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    // 记录空闲时需要处理的IdleHandler数量
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    // ->> 分析3.2.1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            // 刷新Binder命令
            Binder.flushPendingCommands();
        }

        // 调用native层，如果返回了就说明可以从队列中取出一条消息
        // 如果消息队列中没有消息就阻塞等待，靠enqueueMessage()中最后一步调用nativeWake()来唤醒该方法
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            // 得到时间-从手机开机到现在调用经过的时间
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            // 得到队头
            Message msg = mMessages;
            // 判断这个message是不是barrier
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                // 循环遍历出第一个异步消息，如果设置了barrier，就不能再执行同步消息了，除非将barrier移除。
                // 但是异步消息不受影响照样执行，所以在这里要找到异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                // 如果分发时间还没到
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    // 更新执行时间点
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 时间到了
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 如果没有其他消息了
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            // 正在退出了，返回null。
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            // 判断如果这是第一次循环（只有第一次循环时会小于0）并且队列为空或还没到处理第一个的时间
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                // 置为阻塞状态
                mBlocked = true;
                continue;
            }

            // 初始化最少四个要被执行的IdleHandler
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        // 开始循环执行所有的IdleHandler并根据返回值判断是否保留
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // IdleHandler只会在消息队列阻塞之前执行一次，之后再不会执行，知道下一次被调用next()。
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        // 当执行了IdleHandler后，会消耗一段时间，刺死可能已经到达执行消息的时间了，所以重置该变量再重新检查时间。
        nextPollTimeoutMillis = 0;
    }
}
```

### 分析3.2.1 
nextPollTimeoutMillis是一个变量，用于表示时间。
* 如果nextPollTimeoutMillis = -1，则一直会阻塞不会超时
* 如果nextPollTimeoutMillis = 0，不会阻塞，立即返回
* 如果nextPollTimeoutMillis > 0，最长阻塞nextPollTimeoutMillis毫秒，如果期间有程序唤醒会立即返回


## 分析3.2 Handler # dispatchMessage()
```java
/**
 * 分析3.2 Handler.dispatchMessage()
 */
public void dispatchMessage(@NonNull Message msg) {
    // 先查看message自己callback有没有被设置，如果有那就交给自己的callback去处理
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        // 如果没有message自己的callback，那就看看Handler有没有callback，
        // 如果有，那就交给Handler的Callback去处理，
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 如果Handler自己还没有Callback，那就Handler自己处理
        handleMessage(msg);
    }
}
```
从上面可以得到以下信息：
处理消息的方式有3个，分别为Message自己的Callback、Handler的Callback以及Handler自己。
1. 优先级最高的是Message自己的callback，这是一个Runnable对象，我们用Handler post一个Runnable的时候，其实就是把这个Runnable赋值个一个Message对象的callback，然后把Message对象发送给MessageQueue。
2. 优先级第二的是Handler自己的mCallback，在我们创建一个Handler对象的使用可以传一个Handler.Callback对象给它并实现Callback里的handleMessage(msg)方法，作为我们的消息处理方法。
3. 优先级最低的是Handler自己的handleMessage(msg)方法，这也是我们最常用的消息处理方法。

# 流程图
（图片来自于[此处](https://juejin.im/post/5c74b64a6fb9a049be5e22fc#heading-7)）
![](https://cdn.littlecorgi.top/mweb/2019-11-17/15739707276639.jpg)
![](https://cdn.littlecorgi.top/mweb/2019-11-17/15739710069538.jpg)


# 延伸
## 1. 为什么说Handler可能会导致内存泄漏
只要你用Android Studio，并在Activity里面用过Handler，都会注意到一个地方，就是如果你直接创建Handler对象并重写handleMessage方法的话，AS一把都会报一个warning：
![](https://cdn.littlecorgi.top/mweb/2019-11-17/15739712887055.jpg)
```
This Handler class should be static or leaks might occur.

该处理程序类应为静态，否则可能发生泄漏
```

就是说如果直接这样写就可能会导致内存泄漏，但是如果你在不是Activity的类里面这样写又不会报warning，这是为什么呢？

因为Handler允许我们发送延时消息，但是如果在延时期间，用户关闭了Activity。这时Message持有Handler，而又因为Java的特性，内部类会持有外部类，也就是说Handler会持有Activity，这样就导致Activity泄漏了。

解决办法就是把Handler定义为静态内部类，并在内部持有Activity的弱引用，并及时移除所有消息：
```java
private static class SafeHandler extends Handler {
    private WeakReference<MainActivity> ref;

    public SafeHandler(MainActivity activity) {
        this.ref = new WeakReference<>(activity);
    }

    @Override
    public void handleMessage(@NonNull Message msg) {

    }
}
```


## 2. 为什么主线程不需要创建Looper，而且主线程的Looper不许退出
因为在主线程创建时会自动调用Looper的prepare方法，并调用loop方法。我们就可以直接在主线程使用Handler。

而为什么不能退出，是因为如果Looper退出了，那么主线程就会挂掉。Android也不许你手动退出主线程的Looper。

