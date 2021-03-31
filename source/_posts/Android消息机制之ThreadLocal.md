---
title: Android消息机制之ThreadLocal
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
abbrlink: 64f0a9f6
date: 2019-11-08 23:00:55
updated: 2019-11-08 23:00:55
cover: "https://cdn.littlecorgi.top/blog/Keynote_Blog_FINAL.png"
keywords:
description:
---
# 简介
ThreadLocal是一个线程内部的数据储存类，通过他可以在指定的线程中存储数据。数据存储以后，只有在指定的线程中可以获取到存储的数据，对于其他线程来说则无法获取。

在源码中是这样写的：
>This class provides thread-local variables.  These variables differ from
>their normal counterparts in that each thread that accesses one (via its
>{@code get} or {@code set} method) has its own, independently initialized
>copy of the variable.  {@code ThreadLocal} instances are typically private
>static fields in classes that wish to associate state with a thread (e.g.,
>a user ID or Transaction ID).
 
翻译过来，就是说这个类提供线程局部变量，但是他和普通变量不同，他是每个线程都有自己的一个独立初始化的变量副本。而通过`get()`和`set()`方法就能得到和设置当前线程对应的该变量的值。

# 示例
接下来我们看一下示例代码，在一个Activity的onCreate方法中写入如下代码：
```java
public class MainActivity extends AppCompatActivity {

    // 需要添加的代码
    private static final String TAG = "MainActivity";
    // 需要添加的代码
    private ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 需要添加的代码
        mBooleanThreadLocal.set(true);
        Log.d(TAG, "[Thread#main]mBooleanThreadLocal=" + mBooleanThreadLocal.get());

        // 需要添加的代码
        new Thread("Thread#1") {
            @Override
            public void run() {
                mBooleanThreadLocal.set(false);
                Log.d(TAG, "[Thread#1]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
            }
        }.start();
        
        // 需要添加的代码
        new Thread("Thread#2") {
            @Override
            public void run() {
                Log.d(TAG, "[Thread#2]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
            }
        }.start();
    }
}
```

根据我们上面说的ThreadLocal的特性，就可以猜出应该会输出：
```java
[Thread#main]mBooleanThreadLocal=true
[Thread#1]mBooleanThreadLocal=false
[Thread#2]mBooleanThreadLocal=null
```

然后运行输出的结果，不出所料，果然和上面一样。

# 使用场景
一般来说，当某些数据是以线程作为作用域并且不同线程具有不同的数据副本的时候，就可以考虑使用ThreadLocal。比如对于Handler来说，它需要获取当前线程的Looper，很显然Looper的作用域就是线程并且不同线程具有不同的Looper，这个时候通过ThreadLocal就可以轻松实现Looper在线程中的存取，如果不采用ThreadLocal，那么系统就必须提供一个全局的哈希表供Handler查找指定线程的Looper，这样一来就必须提供一个类似于LooperManager的类了，但是系统并没有这么做而是选择了ThreadLocal，这就是ThreadLocal的好处。

另一个使用场景就是复杂逻辑下的对象传递。比如监听器的传递，有些时候一个线程中的任务过于复杂，这可能表现为函数调用栈比较深以及代码入口的多样性，在这种情况下，我们又需要监听器能够贯穿整个线程的执行过程，这个时候可以怎么做呢？其实就可以采用ThreadLocal，采用ThreadLocal可以让监听器作为线程内的全局对象而存在，在线程内部只要通过get方法就可以获取到监听器。而如果不采用ThreadLocal，那么我们能想到的可能是如下两种方法：第一种方法是将监听器通过参数的形式在函数调用栈中进行传递，第二种方法就是将监听器作为静态变量供线程访问。上述这两种方法都是有局限性的。第一种方法的问题时当函数调用栈很深的时候，通过函数参数来传递监听器对象这几乎是不可接受的，这会让程序的设计看起来很糟糕。第二种方法是可以接受的，但是这种状态是不具有可扩充性的，比如如果同时有两个线程在执行，那么就需要提供两个静态的监听器对象，如果有10个线程在并发执行呢？提供10个静态的监听器对象？这显然是不可思议的，而采用ThreadLocal每个监听器对象都在自己的线程内部存储，根据就不会有方法2的这种问题。

# 源码
作为一个存储数据的类，关键点就在get和set方法。
## ThreadLocal # set

```java
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 实际上存储数据的结构类型
    // ->> 分析1
    ThreadLocalMap map = getMap(t);
    // 如果存在Map就直接set，没有就创建map并set
    if (map != null)
        map.set(this, value);
    else
        // ->> 分析2
        createMap(t, value);
}


/**
 * 分析1：getMap()方法
 */
ThreadLocalMap getMap(Thread t) {
    // 返回传入线程的ThreadLocalMap
    return t.threadLocals;
}

/**
 * 分析2：createMap()方法
 */
void createMap(Thread t, T firstValue) {
    // 创建一个新的ThreadLocalMap并将值传入
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
当你调用set方法时，就会拿到当前线程并得到当前线程的ThreadLocalMap，如果map不为空，那就直接把值传入map；如果map为空那就新建一个再传值。

这块我们就能懂为啥ThreadLocal能只操作自己线程里面的东西了，因为所有ThreadLocal都与他线程中的ThreadLocalMap有关。

那我们再来看看ThreadLocalMap。
## ThreadLocalMap
### 属性变量
先来看下ThreadLocalMap的一些变量。
```java
// 存储数据的结构为Entry，而且key是弱引用
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

// table的初始容量
private static final int INITIAL_CAPACITY = 16;

// table用于存储数据
private Entry[] table;

// 
private int size = 0;

// 负载因子，用于扩容
private int threshold; // Default to 0

// 设置负载因子，默认为当前大小的2/3
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}

// 下一个索引
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

// 上一个索引
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}

// 构造函数
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

### ThreadLocalMap # set()
```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 根据哈希算法找到对应节点
    int i = key.threadLocalHashCode & (len-1);

    //判断当前位置是否有数据，如果key值相同，就替换，如果不同则找空位放数据。
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        //判断key值相同否，如果是直接覆盖 （第一种情况）
        if (k == key) {
            e.value = value;
            return;
        }
        //如果当前Entry对象对应Key值为null,则清空所有Key为null的数据（第二种情况）
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    //以上情况都不满足，直接添加（第三种情况）
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
先根据key通过哈希算法得到对应的i，然后再开始从这个i开始遍历，进行判断，由于下面三个判断比较复杂，所以我们分开来讲。

#### 第一种情况
这种情况下，key值相同，就需要将value替换掉。
![第一种情况](https://cdn.littlecorgi.top/mweb/2019-11-08/%E7%AC%AC%E4%B8%80%E7%A7%8D%E6%83%85%E5%86%B5.jpg)

#### 第二种情况
这种情况比较复杂，先看下代码：
```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    
    int slotToExpunge = staleSlot;
    // 先往前进行判断，看是否能找到空的节点，找到了就更新slotToExpunge
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // 再往后遍历
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 如果当前i节点的key和传入的key相同，那就进行替换
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            /*
             * 如果在整个扫描过程中（包括函数一开始的向前扫描与i之前的向后扫描）
             * 找到了之前的无效slot则以那个位置作为清理的起点，
             * 否则则以当前的i作为清理起点
             */
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            // 进行一次连续段的清理，再做一次启发式清理
            // ->> 分析1
            // ->> 分析2
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果当前的slot已经无效，并且向前扫描过程中没有无效slot，则更新slotToExpunge为当前位置
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果key在table中不存在，则在原地放一个即可
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 在探测过程中如果发现任何无效slot，则做一次清理（连续段清理+启发式清理）
    // ->> 分析1
    // ->> 分析2
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}


/**
 * 分析1：expungeStaleEntry()
 * 作用：把连续段内所有无效的slot都清理一遍
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}

/**
 * 分析2：cleanSomeSlots()
 * 作用：遍历删除所有位置下key==null的数据
 */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            // ->> 分析3
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

示意图如下：
![第二种情况](https://cdn.littlecorgi.top/mweb/2019-11-08/%E7%AC%AC%E4%BA%8C%E7%A7%8D%E6%83%85%E5%86%B5.jpg)


#### 第三种情况
第三种情况就是上面两种情况都不满足的情况，也就是需要插入的位置为null的时候，就直接扩大ThreadLocalMap，然后再插入：
![第三种情况](https://cdn.littlecorgi.top/mweb/2019-11-08/%E7%AC%AC%E4%B8%89%E7%A7%8D%E6%83%85%E5%86%B5.jpg)


```java
tab[i] = new Entry(key, value);
int sz = ++size;
// ->> cleanSomeSlots见上面第二种情况的分析2
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    // ->> 分析1
    rehash();


/**
 * 分析1：rehash()
 */
private void rehash() {
    // ->> expungeStaleEntries见上面第二种情况的分析1
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        // ->> 分析2
        resize();
}

/**
 * 分析2：resize()
 */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### ThreadLocalMap # getEntry()
```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        // ->> 分析1
        return getEntryAfterMiss(key, i, e);
}


/**
 * 分析1：getEntryAfterMiss()
 */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    // 从当前往后一个个进行遍历，直至找到和key相等的然后返回
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

### ThreadLocalMap # remove()
```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            // 显式断开弱引用
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```