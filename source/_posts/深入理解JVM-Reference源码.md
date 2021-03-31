---
title: 深入理解JVM-Reference源码
categories:
  - Java
tags:
  - Java
  - JVM
cover: 'https://raw.githubusercontent.com/zxh0/jvmgo-book/master/v1/gophers/cover.png'
toc_number: false
abbrlink: 79199b37
date: 2021-03-21 22:15:24
updated: 2021-03-21 22:15:24
description:
---
# Reference源码分析

>   本文基于JDK1.8.0_271分析，native源码下载自[openJDK官网(build 1.8.0_41-b04)](http://jdk.java.net/java-se-ri/8-MR3)

# 0. 前言

JDK1.2开始，引入了一个新的包，`java.lang.ref`：

>   - java.lang.ref
>       - Finalizer.class
>       - FinalizerHistogram.class
>       - FinalReference.class
>       - PhantomReference.class
>       - Reference.class
>       - ReferenceQueue.class
>       - SoftReference.classs
>       - WeakReference.class
>   - 额外的还有一个在sun.misc包下
>       - Cleaner.class

随之带来了四个新的概念：

- 强引用Storn References：随处可见，我们直接new出来的代码就是强引用。内存不足时，宁愿抛出`OutOfMemoryError`也不愿意回收这些对象。我们可以手动的设置为null让GC回收他。
- 软引用SoftReference：等级比强引用低，只有在内存不足的时候才回去回收。我们可以实现内存敏感的高速缓存。
- 弱引用WeakReference：等级比软引用低，不管内存足不足，发生GC时，就有可能被回收。如果这个对象是一个偶尔使用的对象，并且需要在使用的时候就能获取到，但又不想影响生命周期，就可以使用WeakReference，比如处理内部类内存泄漏的问题时。
- 虚引用PhantomReference：等级比弱引用低，形同虚设，任何时候都可能被回收。它可以和引用队列配合用于监控对象是否被回收

<!-- More -->

并且，除了强引用之外，其他三个都可以和引用队列配合使用，就是在调用他们的构造方法时，除了传入Object外，还可以传入一个ReferenceQueue，他的作用就是当Object被回收时，JVM会自动帮我们把这个软/弱/虚引用添加到这个Queue中。通过这个功能可以监控对象是否被回收。

直到JDK8为止，只存在四种引用，这些引用是由JVM创建，因此直接继承`java.lang.ref.Reference`创建自定义的引用类型是无效的，但是可以直接继承已经存在的引用类型，如`sun.misc.Cleaner`就是继承自`java.lang.ref.PhantomReference`。



接下来我们就可以源码，但是在此之前，我们得带着问题看：

-   软引用是内存不足才会回收，那么什么叫内存不足？
-   弱引用只要发生GC时就回收，但是他不是引用着对应的强引用，那么他为啥能被回收？
-   虚引用形同虚设，任何时候都会被回收，那么到底什么时候会被回收？
-   当引用对象被回收时，他们是怎么被添加到若引用队列的？

# 1. Reference.java

## 1.1 源码

首先来看下他的构造方法和成员变量：

```java
public abstract class Reference<T> {
    private T referent;         /* Treated specially by GC */
		volatile ReferenceQueue<? super T> queue;
    @SuppressWarnings("rawtypes")
    volatile Reference next;
    transient private Reference<T> discovered;  /* used by VM */
    static private class Lock { }
    private static Lock lock = new Lock();
    private static Reference<Object> pending = null;

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}
```

### 1.1.1 构造方法

构造方法有两种，一种只传入referent，另一种还要传入一个Queue，如果传入的queue为null的话，成员变量queue就等于`ReferenceQueue.NULL`。

### 1.1.2 成员变量

-   reference：代表我们传入的对象。

-   queue: ReferenceQueue，就是引用队列。

    ```java
    volatile ReferenceQueue<? super T> queue;
    ```

-   next：下一个Reference实例的引用，主要是在ReferenceQueue中使用。

    ```java
    /* When active:   NULL
     *     pending:   this
     *    Enqueued:   next reference in queue (or this if last)
     *    Inactive:   this
     */
    @SuppressWarnings("rawtypes")
    volatile Reference next;
    ```

-   discovered：注意这个属性由transient修饰，基于状态表示不同链表中的下一个待处理的对象，主要是PendingList的下一个元素，通过JVM直接调用赋值。

    ```java
    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    private transient Reference<T> discovered;
    ```

-   lock: 线程安全的锁

    ```java
    /* Object used to synchronize with the garbage collector.  The collector
     * must acquire this lock at the beginning of each collection cycle.  It is
     * therefore critical that any code holding this lock complete as quickly
     * as possible, allocate no new objects, and avoid calling user code.
     */
    static private class Lock { }
    private static Lock lock = new Lock();
    ```

-   pending：等待加入queue的Reference对象，在GC时由JVM设置，会有一个java层的线程(ReferenceHandler)源源不断从PendingList中取出元素加入到queue中

    ```java
    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    private static Reference<Object> pending = null;
    ```

### 1.1.3 方法

```java
/**
 * Returns this reference object's referent.  If this reference object has
 * been cleared, either by the program or by the garbage collector, then
 * this method returns <code>null</code>.
 *
 * @return   The object to which this reference refers, or
 *           <code>null</code> if this reference object has been cleared
 */
// 获取referent实例
@HotSpotIntrinsicCandidate
public T get() {
    return this.referent;
}

/**
 * Clears this reference object.  Invoking this method will not cause this
 * object to be enqueued.
 *
 * <p> This method is invoked only by Java code; when the garbage collector
 * clears references it does so directly, without invoking this method.
 */
// 清除referent实例
public void clear() {
    this.referent = null;
}

/* -- Queue operations -- */

/**
 * Tells whether or not this reference object has been enqueued, either by
 * the program or by the garbage collector.  If this reference object was
 * not registered with a queue when it was created, then this method will
 * always return <code>false</code>.
 *
 * @return   <code>true</code> if and only if this reference object has
 *           been enqueued
 */
// 判断引用队列是否处于ENQUEUED状态
public boolean isEnqueued() {
    return (this.queue == ReferenceQueue.ENQUEUED);
}

/**
 * Adds this reference object to the queue with which it is registered,
 * if any.
 *
 * <p> This method is invoked only by Java code; when the garbage collector
 * enqueues references it does so directly, without invoking this method.
 *
 * @return   <code>true</code> if this reference object was successfully
 *           enqueued; <code>false</code> if it was already enqueued or if
 *           it was not registered with a queue when it was created
 */
// 入队
public boolean enqueue() {
    return this.queue.enqueue(this);
}
```

## 1.2 ReferenceHandler线程

```java
/* High-priority thread to enqueue pending References
 */
private static class ReferenceHandler extends Thread {

 		// 确保该类已经加载完成，原理就是通过forName去提前加载
    private static void ensureClassInitialized(Class<?> clazz) {
        try {
            Class.forName(clazz.getName(), true, clazz.getClassLoader());
        } catch (ClassNotFoundException e) {
            throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
        }
    }

    static {
        // pre-load and initialize InterruptedException and Cleaner classes
        // so that we don't get into trouble later in the run loop if there's
        // memory shortage while loading/initializing them lazily.
      	// 提前加载这两个类
        ensureClassInitialized(InterruptedException.class);
        ensureClassInitialized(Cleaner.class);
    }

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }

    public void run() {
        while (true) {
          	// 死循环调用此方法
            tryHandlePending(true);
        }
    }
}
```

源码很简单，这个线程的主要作用就是不断的去调用`tryHandlePending()`方法

我们再看看他的启动：

```java
static {
  	// 大致意思就是获取当前线程组的上一层线程组，一层层获取上去，直到最高层的线程组，再去创建ReferenceHandler，以保证他被创建在JVM的system线程组中
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
         tgn != null;
         tg = tgn, tgn = tg.getParent());
    Thread handler = new ReferenceHandler(tg, "Reference Handler");
    /* If there were a special system-only priority greater than
     * MAX_PRIORITY, it would be used here
     */
  	// 设置高优先级
    handler.setPriority(Thread.MAX_PRIORITY);
  	// 设置为守护线程
    handler.setDaemon(true);
  	// 启动
    handler.start();

    // provide access in SharedSecrets
    SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
        @Override
        public boolean tryHandlePendingReference() {
            return tryHandlePending(false);
        }
    });
}
```

## 1.3 核心方法：tryHandlePending

```java
/**
 * Try handle pending {@link Reference} if there is one.<p>
 * Return {@code true} as a hint that there might be another
 * {@link Reference} pending or {@code false} when there are no more pending
 * {@link Reference}s at the moment and the program can do some other
 * useful work instead of looping.
 *
 * @param waitForNotify if {@code true} and there was no pending
 *                      {@link Reference}, wait until notified from VM
 *                      or interrupted; if {@code false}, return immediately
 *                      when there is no pending {@link Reference}.
 * @return {@code true} if there was a {@link Reference} pending and it
 *         was processed, or we waited for notification and either got it
 *         or thread was interrupted before being notified;
 *         {@code false} otherwise.
 */
static boolean tryHandlePending(boolean waitForNotify) {
    Reference<Object> r;
    Cleaner c;
    try {
      	// 加锁，保证线程安全
        synchronized (lock) {
            if (pending != null) {
                r = pending;
                // 'instanceof' might throw OutOfMemoryError sometimes
                // so do this before un-linking 'r' from the 'pending' chain...
              	// 判断是不是Cleaner，如果是，后续有特殊处理
                c = r instanceof Cleaner ? (Cleaner) r : null;
                // unlink 'r' from 'pending' chain
              	// 再指向下一个元素，并将当前元素置空，熟悉的链表的处理
                pending = r.discovered;
                r.discovered = null;
            } else {
              	// 如果pending不为null的话，则证明当前没有元素要被加入到queue了，则挂起当前线程，也就是ReferenceHandler
                // The waiting on the lock may cause an OutOfMemoryError
                // because it may try to allocate exception objects.
                if (waitForNotify) {
                    lock.wait();
                }
                // retry if waited
                return waitForNotify;
            }
        }
    } catch (OutOfMemoryError x) {
        // Give other threads CPU time so they hopefully drop some live references
        // and GC reclaims some space.
        // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
        // persistently throws OOME for some time...
        Thread.yield();
        // retry
        return true;
    } catch (InterruptedException x) {
        // retry
        return true;
    }

    // Fast path for cleaners
  	// 如果是Cleaner，则调用clear进行回收
    if (c != null) {
        c.clean();
        return true;
    }
		// 如果Reference有queue，则加入进去
    ReferenceQueue<? super Object> q = r.queue;
    if (q != ReferenceQueue.NULL) q.enqueue(r);
    return true;
}
```

源码很好理解，就是线程死循环源源不断的从PendingList中取出pending，然后将其加入到ReferenceQueue中去，而PendingList又是根据discovered来移动的。

并且对于Cleaner类型有特别处理，当其指向的对象被回收时，会调用clean进行资源回收。



那么，我们知道了对pending的处理，那么，那些对象什么时候会被加入到PendingList中去呢？根据注释可以得知，我们需要从native层去找到答案。

# 2. referenceProcessor.cpp

Reference对应的native代码主要在referenceProcessor.cpp文件中，这个文件在`hotspot\src\share\vm\memory`路径下。

加入PendingList的核心方法是`process_discovered_references()`:

```c++
ReferenceProcessorStats ReferenceProcessor::process_discovered_references(
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor,
  GCTimer*                     gc_timer,
  GCId                         gc_id) {
  NOT_PRODUCT(verify_ok_to_handle_reflists());

  assert(!enqueuing_is_done(), "If here enqueuing should not be complete");
  // Stop treating discovered references specially.
  disable_discovery();

  // If discovery was concurrent, someone could have modified
  // the value of the static field in the j.l.r.SoftReference
  // class that holds the soft reference timestamp clock using
  // reflection or Unsafe between when discovery was enabled and
  // now. Unconditionally update the static field in ReferenceProcessor
  // here so that we use the new value during processing of the
  // discovered soft refs.

  _soft_ref_timestamp_clock = java_lang_ref_SoftReference::clock();

  bool trace_time = PrintGCDetails && PrintReferenceGC;

  // 处理软引用
  // Soft references
  size_t soft_count = 0;
  {
    GCTraceTime tt("SoftReference", trace_time, false, gc_timer, gc_id);
    soft_count =
      process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();

  // 处理弱引用
  // Weak references
  size_t weak_count = 0;
  {
    GCTraceTime tt("WeakReference", trace_time, false, gc_timer, gc_id);
    weak_count =
      process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Final references
  //...

  // 处理虚引用
  // Phantom references
  size_t phantom_count = 0;
  {
    GCTraceTime tt("PhantomReference", trace_time, false, gc_timer, gc_id);
    phantom_count =
      process_discovered_reflist(_discoveredPhantomRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);

    // Process cleaners, but include them in phantom statistics.  We expect
    // Cleaner references to be temporary, and don't want to deal with
    // possible incompatibilities arising from making it more visible.
    phantom_count +=
      process_discovered_reflist(_discoveredCleanerRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Weak global JNI references. It would make more sense (semantically) to
  // traverse these simultaneously with the regular weak references above, but
  // that is not how the JDK1.2 specification is. See #4126360. Native code can
  // thus use JNI weak references to circumvent the phantom references and
  // resurrect a "post-mortem" object.
  {
    GCTraceTime tt("JNI Weak Reference", trace_time, false, gc_timer, gc_id);
    if (task_executor != NULL) {
      task_executor->set_single_threaded_mode();
    }
    process_phaseJNI(is_alive, keep_alive, complete_gc);
  }

  return ReferenceProcessorStats(soft_count, weak_count, final_count, phantom_count);
}
```

从上面注释，我们可以看到大致的处理过程是：

1.  处理软引用
2.  处理弱引用
3.  处理需引用

但是他们都调用了process_discovered_reflist这个方法，唯一的区别只是传入的refs_list不同。

## 2.1 process_discovered_reflist()

```c++
size_t
ReferenceProcessor::process_discovered_reflist(
  DiscoveredList               refs_lists[],
  ReferencePolicy*             policy,
  bool                         clear_referent,
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor)
{
  bool mt_processing = task_executor != NULL && _processing_is_mt;
  // If discovery used MT and a dynamic number of GC threads, then
  // the queues must be balanced for correctness if fewer than the
  // maximum number of queues were used.  The number of queue used
  // during discovery may be different than the number to be used
  // for processing so don't depend of _num_q < _max_num_q as part
  // of the test.
  bool must_balance = _discovery_is_mt;

  if ((mt_processing && ParallelRefProcBalancingEnabled) ||
      must_balance) {
    balance_queues(refs_lists);
  }

  size_t total_list_count = total_count(refs_lists);

  if (PrintReferenceGC && PrintGCDetails) {
    gclog_or_tty->print(", %u refs", total_list_count);
  }

  // Phase 1 (soft refs only):
  // . Traverse the list and remove any SoftReferences whose
  //   referents are not alive, but that should be kept alive for
  //   policy reasons. Keep alive the transitive closure of all
  //   such referents.
  // 只处理软引用
  // 将所有不存活但是还不能被回收的软引用从refs_lists中移除（只有refs_lists为软引用的时候，这里policy才不为null）
  if (policy != NULL) {
    if (mt_processing) {
      RefProcPhase1Task phase1(*this, refs_lists, policy, true /*marks_oops_alive*/);
      task_executor->execute(phase1);
    } else {
      for (uint i = 0; i < _max_num_q; i++) {
        process_phase1(refs_lists[i], policy,
                       is_alive, keep_alive, complete_gc);
      }
    }
  } else { // policy == NULL
    assert(refs_lists != _discoveredSoftRefs,
           "Policy must be specified for soft references.");
  }

  // Phase 2:
  // . Traverse the list and remove any refs whose referents are alive.
  // 移除所有指向对象还存活的引用
  if (mt_processing) {
    RefProcPhase2Task phase2(*this, refs_lists, !discovery_is_atomic() /*marks_oops_alive*/);
    task_executor->execute(phase2);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase2(refs_lists[i], is_alive, keep_alive, complete_gc);
    }
  }

  // Phase 3:
  // . Traverse the list and process referents as appropriate.
  // 根据clear_referent的值决定是否将不存活对象回收
  if (mt_processing) {
    RefProcPhase3Task phase3(*this, refs_lists, clear_referent, true /*marks_oops_alive*/);
    task_executor->execute(phase3);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase3(refs_lists[i], clear_referent,
                     is_alive, keep_alive, complete_gc);
    }
  }

  return total_list_count;
}
```

## 2.2 process_phase1()

```c++
void
ReferenceProcessor::process_phase1(DiscoveredList&    refs_list,
                                   ReferencePolicy*   policy,
                                   BoolObjectClosure* is_alive,
                                   OopClosure*        keep_alive,
                                   VoidClosure*       complete_gc) {
  assert(policy != NULL, "Must have a non-NULL policy");
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  // Decide which softly reachable refs should be kept alive.
  // 遍历refs_list中的所有元素
  while (iter.has_next()) {
    iter.load_ptrs(DEBUG_ONLY(!discovery_is_atomic() /* allow_null_referent */));
    // 判断所引用的对象是否存活
    bool referent_is_dead = (iter.referent() != NULL) && !iter.is_referent_alive();
    // 如果已经不存活，则根据ReferencePolicy去判断是否应该回收，should_clear_reference返回false，则从refs_list中移除，也就是不回收
    if (referent_is_dead &&
        !policy->should_clear_reference(iter.obj(), _soft_ref_timestamp_clock)) {
      // 回收对象
      if (TraceReferenceGC) {
        gclog_or_tty->print_cr("Dropping reference (" INTPTR_FORMAT ": %s"  ") by policy",
                               (void *)iter.obj(), iter.obj()->klass()->internal_name());
      }
      // Remove Reference object from list
      iter.remove();
      // Make the Reference object active again
      iter.make_active();
      // keep the referent around
      iter.make_referent_alive();
      iter.move_to_next();
    } else {
      // 跳过当前，继续遍历
      iter.next();
    }
  }
  // Close the reachable set
  complete_gc->do_void();
  NOT_PRODUCT(
    if (PrintGCDetails && TraceReferenceGC) {
      gclog_or_tty->print_cr(" Dropped %d dead Refs out of %d "
        "discovered Refs by policy, from list " INTPTR_FORMAT,
        iter.removed(), iter.processed(), (address)refs_list.head());
    }
  )
}
```

这段代码很好理解，遍历refs_list，取出里面的每一个元素，并判断是否应该移除，判断的依据包括这个元素所引用的对象是否存活以及ReferencePolicy。那么这个ReferencePolicy是啥？

### 2.2.1 referencePolicy.hpp

打开这个文件，我们可以看到ReferencePolicy的定义，除了这个类以外，还有4个他的子类：

-   NeverClearPolicy
-   AlwaysClearPolicy
-   LRUCurrentHeapPolicy
-   LRUMaxHeapPolicy

对于前两个类的should_clear_reference()方法很简单，一个永远返回false，一个永远返回true。那下面两个是啥？

#### referencePolicy.cpp

截取下主要代码：

```C++
// M = 1024 * 1024
// SoftRefLRUPolicyMSPerMB = 1000

// Capture state (of-the-VM) information needed to evaluate the policy
void LRUCurrentHeapPolicy::setup() {
  _max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

// The oop passed in is the SoftReference object, and not
// the object the SoftReference points to.
bool LRUCurrentHeapPolicy::should_clear_reference(oop p,
                                                  jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) {
    return false;
  }

  return true;
}

// Capture state (of-the-VM) information needed to evaluate the policy
void LRUMaxHeapPolicy::setup() {
  size_t max_heap = MaxHeapSize;
  max_heap -= Universe::get_heap_used_at_last_gc();
  max_heap /= M;

  _max_interval = max_heap * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

// The oop passed in is the SoftReference object, and not
// the object the SoftReference points to.
bool LRUMaxHeapPolicy::should_clear_reference(oop p,
                                             jlong timestamp_clock) {
// 此方法和上面的一模一样，所以不再单独拿出来
```

单纯看他两的setup方法：

```c++
_max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
_max_interval = (MaxHeapSize - Universe::get_heap_used_at_last_gc()) / M * SoftRefLRUPolicyMSPerMB;
```

前者计算的是上次GC后的可用堆大小，后者计算的是(堆大小-上次GC时使用的大小)

而should_clear_reference方法：

```C++
jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
assert(interval >= 0, "Sanity check");

// The interval will be zero if the ref was accessed since the last scavenge/gc.
if(interval <= _max_interval) {
  return false;
}

return true;
```

但是，`timestamp_clock`和` java_lang_ref_SoftReference::timestamp(p)`是什么呢？

第一个不说，第二个看这个形式，是不是很熟悉，这不就是jni嘛，我们再次回到java层。

#### 软引用、弱引用和虚引用

Reference主要的三个子类，弱引用和虚引用虽然继承自Reference，但是他们改动不大：

```java
public class WeakReference<T> extends Reference<T> {
    public WeakReference(T referent) {
        super(referent);
    }
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
public class PhantomReference<T> extends Reference<T> {
    public T get() {
        return null;
    }
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

反倒是弱引用，改动了较多：

```java
public class SoftReference<T> extends Reference<T> {
  	// 时间戳时钟，由GC更新。
    static private long clock;
  	// 每次调用get方法所更新的时间戳。虚拟机在选择要清除的软引用时，可以使用此字段，但不要求这样做。
    private long timestamp;
    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }
    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }
    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }
}
```

`clock`就是上面native代码的`timestamp_clock`，`timestamp`就是`java_lang_ref_SoftReference::timestamp(p)`。如果上次GC时有调用过`get()`那么interval为0，否则就是他们之间的差值。也就是说，如果GC间隔时间太长了，就回被回收。



所以我们更新下，判断软引用是否改被移除的依据包括这个元素所引用的对象是否存活、Policy策略以及存活时间。



## 2.3 process_phase2()

```c++
// Traverse the list and remove any Refs that are not active, or
// whose referents are either alive or NULL.
void
ReferenceProcessor::pp2_work(DiscoveredList&    refs_list,
                             BoolObjectClosure* is_alive,
                             OopClosure*        keep_alive) {
  assert(discovery_is_atomic(), "Error");
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  // 遍历refs_list中的所有元素
  while (iter.has_next()) {
    iter.load_ptrs(DEBUG_ONLY(false /* allow_null_referent */));
    DEBUG_ONLY(oop next = java_lang_ref_Reference::next(iter.obj());)
    assert(next == NULL, "Should not discover inactive Reference");
    // 判断引用是否存活，存活则从refs_list中移除，否则遍历下一个元素
    if (iter.is_referent_alive()) {
      if (TraceReferenceGC) {
        gclog_or_tty->print_cr("Dropping strongly reachable reference (" INTPTR_FORMAT ": %s)",
                               (void *)iter.obj(), iter.obj()->klass()->internal_name());
      }
      // The referent is reachable after all.
      // Remove Reference object from list.
      iter.remove();
      // Update the referent pointer as necessary: Note that this
      // should not entail any recursive marking because the
      // referent must already have been traversed.
      iter.make_referent_alive();
      iter.move_to_next();
    } else {
      iter.next();
    }
  }
  NOT_PRODUCT(
    if (PrintGCDetails && TraceReferenceGC && (iter.processed() > 0)) {
      gclog_or_tty->print_cr(" Dropped %d active Refs out of %d "
        "Refs in discovered list " INTPTR_FORMAT,
        iter.removed(), iter.processed(), (address)refs_list.head());
    }
  )
}
```

这个代码很简单，就是遍历refs_list，如果所指向的对象还存活，则从list中移除，否则保留着继续遍历。

## 2.4 process_phase3()

```c++
// Traverse the list and process the referents, by either
// clearing them or keeping them (and their reachable
// closure) alive.
void
ReferenceProcessor::process_phase3(DiscoveredList&    refs_list,
                                   bool               clear_referent,
                                   BoolObjectClosure* is_alive,
                                   OopClosure*        keep_alive,
                                   VoidClosure*       complete_gc) {
  ResourceMark rm;
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  // 遍历refs_list，此时这里已经是经过phase1和2过滤后的剩下的元素
  while (iter.has_next()) {
    // 更新discovered变量，discovered的值更新为上一个元素的discovered
    iter.update_discovered();
    iter.load_ptrs(DEBUG_ONLY(false /* allow_null_referent */));
    if (clear_referent) {
      // NULL out referent pointer
      // 清除引用，之后会被GC回收
      iter.clear_referent();
    } else {
      // keep the referent around
      // 标记引用的对象为存活，该对象在这次GC不会被回收
      iter.make_referent_alive();
    }
    if (TraceReferenceGC) {
      gclog_or_tty->print_cr("Adding %sreference (" INTPTR_FORMAT ": %s) as pending",
                             clear_referent ? "cleared " : "",
                             (void *)iter.obj(), iter.obj()->klass()->internal_name());
    }
    assert(iter.obj()->is_oop(UseConcMarkSweepGC), "Adding a bad reference");
    iter.next();
  }
  // Remember to update the next pointer of the last ref.
  // 更新discovered变量
  iter.update_discovered();
  // Close the reachable set
  complete_gc->do_void();
}
```

# 3. 总结

看完了代码，我们来回答下之前提出的问题：

-   软引用是内存不足才会回收，那么什么叫内存不足？

    根据`process_phase1()`中ReferencePolicy的四个子类，我们可以得知，内存不足的定义和该引用对象get的时间以及当前堆可用内存大小都有关系。具体可以看LRUCurrentHeapPolicy和LRUMaxHeapPolicy

-   弱引用只要发生GC时就回收，但是他不是引用着对应的强引用，那么他为啥能被回收？

    这个问题可以回顾`process_phase3()`，它里面会有一个if判断，如果`clear_referent`为true，就回收，并把`reference`置为null，否则不回收。那么这个变量是在哪被赋值的呢？我们一个个调用看上去，就会神奇的发现，在`process_discovered_references()`方法中，软引用和弱引用的`clear_referent`直接为true，虚引用为false。所以只要发生了GC，弱引用就会被回收。(而软引用如果内存充足的情况下，在`process_phase1()`时就已经被从队列中移除了，并不会走到`process_phase2()`)

-   虚引用形同虚设，任何时候都会被回收，那么到底什么时候会被回收？

    虚引用是会影响对象生命周期的，如果不做任何处理，只要虚引用不被回收，那其引用的对象永远不会被回收。所以一般来说，从ReferenceQueue中获得PhantomReference对象后，如果PhantomReference对象不会被回收的话（比如被其他GC ROOT可达的对象引用），需要调用`clear`方法解除PhantomReference和其引用对象的引用关系。

-   当引用对象被回收时，他们是怎么被添加到若引用队列的？

    看`tryHandlePending()`。