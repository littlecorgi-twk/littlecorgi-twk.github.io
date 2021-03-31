---
title: Java锁机制
categories:
  - Java
tags:
  - Java
  - synchronized
  - lock
  - 锁
cover: https://cdn.littlecorgi.top/blog/Java%E9%94%81%E6%9C%BA%E5%88%B6.jpg
abbrlink: d88880aa
date: 2019-11-28 22:35:04
updated: 2019-11-28 22:35:04
description:
---
# 悲观锁和乐观锁
乐观锁和悲观锁都是一种广义上的概念。

对于同一个数据的并发操作，悲观锁确实很悲观，它时时刻刻都非常担心自己在读取或者写入数据的时候有其他线程来修改数据，所以为了安心，他一般就直接从读取数据的时候就加锁，确保自己操作的时候不会被其它线程打扰。最典型的悲观锁就是synchronized和Lock。

而乐观锁确实很乐观，它认为自己在操作数据的时候是不会有其他线程来修改数据的，所以丝毫不担心，完全不会添加锁。只是在更新数据的时候判断有没有其他线程更改了数据，如果这个数据没有被更新，当前线程再把自己的数据写入；如果数据已经更新，则根据不通过的实现方式执行不同的操作。

看到这，大家就懂了，乐观锁不就是缓存一致性以及CAS算法嘛。

那什么是CAS算法呢？

CAS，比较与交换，一中著名的无所算法。CAS的主要就设计到三个值：
* 需要读写的内存值V
* 进行比较的值A
* 要写入的新值B

当且仅当V的值等于A时，CAS通过原子操作去用新值B来更新V的值，否则不会执行操作。

但是CAS也有缺点：
* 循环时间长开销大：如果一个线程已知都发现自己的值都不是A，那么他会已知自旋
* 只能保证一个共享变量的原子操作
* ABA问题
    * 如果一个线程将数据从A改成了B又改成了A，这样虽然另一个线程通过CAS去更改数据的时候，他就只能发现变量值还是A没有发生变化，就断定这个数据没有其他线程更新过。


# 自旋锁和适应性自旋锁
在说自旋锁之前我们首先来想一个场景：
一般情况下，如果两个线程需要请求同一个资源，其中一个线程先于另一个线程得到资源，那么另一个线程就会阻塞，等到占用资源线程执行完毕释放资源并且CPU正在运行的这个线程运行完毕后重新把它调用上来，这时他才会去执行。

一般咱们都会通过这种方式去达到高效的多线程。但是如果你线程执行的代码很短，甚至需要执行的时间比你CPU调度线程所需的时间都要短，那么这样一直长时间切换短时间执行是不是很不划算。

但是现在电脑基本上都是多核CPU，同一时间各个CPU都可以处理各自的一个线程，我们就可以让后面那个线程先不放弃CPU的执行时间，看看持有资源的线程是否能很快就释放。

为了让当前线程稍等一下，我们就可以给他加个自旋锁，让他自旋，如果自旋完成时前面锁定了资源的线程释放了锁，那么当前线程就可以不避阻塞而是直接获取资源，避免切换线程的开销，这就是自旋锁。

![](https://cdn.littlecorgi.top/mweb/2019-11-28/15748583404329.jpg)

但是自旋锁也有缺点，而且缺点很明显：如果前面那个占用资源的线程需要执行的时间很长，那么这个自旋的线程就会一直占着CPU，CPU就不能去执行其他线程，这样不尽效率没提上去，反而还下降了。

所以自旋锁一定得有限制，如果自旋的次数超过了一定的次数，就会让他取消自旋，挂起线程。

当然了，既然有静态的设置死的限制次数，肯定就有动态的设定灵活的限制次数。旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。


# 无锁、偏向锁、轻量级锁、重量级锁
其实这几个锁都是对于synchronized来说的。再说这个之前，我们前来了解几个概念：
* Java对象头：以HotSpot为例，对象头分为两部分信息：Mark Word和类型指针
    * Mark Word：这块主要用于存储对象自身运行时的数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁等信息。由于这些信息都是与对象自身定义的数据无关，所以考虑到虚拟机空间效率，Mark Word被设计成一个非固定的数据结构以便在绩效的空间内存储尽量多的信息，它会根据对象的状态复用自己的空间。
    * 类型指针：即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这是哪个类的对象。
* Monitor：如果看过操作系统一定不会陌生，这个单词翻译过来就是大名鼎鼎的管程。Java为了保障线程的安全性，就提供了同步机制、互斥锁机制。而这个机制的保障就来自于监视锁Monitor，每个对象都拥有自己的监视锁。他的义务就是控制所有需要访问某个数据的线程，通过调度他们保证只有一个线程能访问受保护的数据和代码。

现在回到synchronized，synchronized通过Monitor来实现线程同步，而Monitor则是依赖于底层的操作系统的MutexLock互斥锁来实现的。

我们在自旋锁中提到，阻塞或唤醒一个线程需要操作系统切换CPU来完成，这种状态的转换需要耗费大量的时间，如果此时加锁的代码过与简单，执行时间就可能比切换时间还要短。这种就是JDK1.6之前的synchronized实现方式，这种锁我们称为重量级锁。而为了减少加锁和释放锁所带来的大量资源的消耗，JDK1.6之后引入了偏向锁和轻量级锁。

这4中锁级别由低到高依次是：无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁
可以由低级升为高级，但是不能由高级降为低级。

|锁状态|存储的内容|存储内容|
|:-:|:-:|:-:|
|无锁|对象的hashCode、对象分代年龄、是否是偏向锁(0)|01|
|偏向锁|偏向线程ID、偏向时间戳、对象分代年龄、是否是偏向锁(1)|01|
|轻量级锁|指向占中锁纪录的指针|00|
|重量级锁|指向互斥量的指针|10|

## 无锁
无锁顾名思义，就是不加锁，不对资源进行锁定，任何线程都可以访问并修改，但是同一时刻只有一个线程能修改成功。

如果一个线程想要访问该资源时，发现该资源正在被访问，他就进行等待，直到资源没有被访问。这就是个循环，一直循环，知道能访问资源。而其他不能访问的线程也一直循环，知道能访问。

CAS就是无锁。

## 偏向锁
如果一段同步代码一直被同一个线程访问，那么这个线程就会自动加上偏向锁。

因为一直都被同一个线程访问，所以他不需要担心有其他线程和他竞争，又由于这段代码加了锁，所以就给他加上偏向锁，避免加更重量级的锁带来的资源消耗，提高效率。

当一个线程访问同步代码块并获取锁时，会在Mark Word里存储锁偏向的线程ID。在线程进入和退出同步块时不再通过CAS操作来加锁和解锁，而是检测Mark Word里是否存储着指向当前线程的偏向锁。引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令即可。

偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态。撤销偏向锁后恢复到无锁（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

## 轻量级锁
当锁是偏向锁但是被另外线程访问的时候，就会自动升级成轻量级锁，这时其它线程会通过自旋的方式来尝试获取锁，不会阻塞，从而提高性能。

指当锁是偏向锁的时候，被另外的线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。

在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，然后拷贝对象头中的Mark Word复制到锁记录中。

拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock Record里的owner指针指向对象的Mark Word。

如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，表示此对象处于轻量级锁定状态。

如果轻量级锁的更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行，否则说明多个线程竞争锁。

若当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁。

## 重量级锁

升级为重量级锁时，锁标志的状态值变为“10”，此时Mark Word中存储的是指向重量级锁的指针，此时等待锁的线程都会进入阻塞状态。

# 公平锁和非公平锁
公平锁就是很公平，按照多个线程的请求锁的顺序依次加入到请求队列中，依次排队。

而非公平锁就是提供了一个插队的机会，如果在一个线程想要去请求锁的时候，刚好这时候锁也没有线程占用，刚好可用，那就直接给这个线程去使用。

公平锁的好处就是等待锁的线程不会饿死，都会按照顺序一个个获取锁；但是缺点就是吞吐量较非公平锁来说较低，等待队列除了第一个可以拿到锁之外其他都得阻塞，直到第一个释放锁，CPU唤醒阻塞线程的开销比非公平锁大。

而非公平锁的好处就是吞吐量比公平锁大，有效减少唤起线程的开销（因为线程可以直接得到锁，不需要再加入到队列），整体吞吐量高；但是缺点是出现后申请的锁反而先拿到资源，而且可能会导致等待队列里面的线程一直拿不到锁，可能会饿死。

其实我觉得理解公平锁的最简单方法就是看图:

你可以看下面的例子，来自于[美团技术团队的博客](https://tech.meituan.com/2018/11/15/java-lock.html)，或者是这个:[一张图读懂非公平锁与公平锁](https://www.jianshu.com/p/f584799f1c77)

公平锁和非公平锁可以形象化为一群人打水。

如图所示，假设有一口水井，有管理员看守，管理员有一把锁，只有拿到锁的人才能够打水，打完水要把锁还给管理员。每个过来打水的人都要管理员的允许并拿到锁之后才能去打水，如果前面有人正在打水，那么这个想要打水的人就必须排队。管理员会查看下一个要去打水的人是不是队伍里排最前面的人，如果是的话，才会给你锁让你去打水；如果你不是排第一的人，就必须去队尾排队，这就是公平锁。
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749265754437.png)

但是对于非公平锁，管理员对打水的人没有要求。即使等待队伍里有排队等待的人，但如果在上一个人刚打完水把锁还给管理员而且管理员还没有允许等待队伍里下一个人去打水时，刚好来了一个插队的人，这个插队的人是可以直接从管理员那里拿到锁去打水，不需要排队，原本排队等待的人只能继续等待。如下图所示：
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749266850342.png)


接下来我们通过ReentrantLock源码来讲一下公平锁和非公平锁在算法上的区别：
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749283538129.jpg)

我们可以看到，ReentrantLock里面有一个内部类Sync，Symc继承自AbstractQueuedSynchronizer，添加锁和释放锁的大部分操作实际上都是在Sync中实现的。他有公平锁FairSync和非公平锁NonFairSync。

我们来看下公平锁FairSync和非公平锁NonFairSync的源码：(左边是非公平锁，右边是公平锁)
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749285866543.jpg)

从源码中我们可以清楚地看到，两个代码唯一地区别在于第4行的那个if，公平锁较非公平锁多了一个限制条件：`!hasQueuedPredecessors()`。

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

判断head和tail不相等（说明有等待线程）并且（head.next为null=>说明有线程正在入队列的中间状态，肯定不是当前线程, 因为一个线程一个时间只能做一件事 或者 head.next.thread不是当前线程)

这个方法的作用就是判断当前线程是不是唯一锁队列的第一个。如果是返true，不是返回false。

#可重入锁和非可重入锁
在讲这连个锁的概念前，我们先来看一个例子：
```java
class test {
    public synchronized void doSomethings() {
        doOthers();
    }
    
    public synchronized void doOthers() {

    }
}
```

上面代码中，两个方法都是synchronized修饰的，`doSomethings()`中调用了`doOthers()`。

如果是非可重入锁的话，那么我们执行`doSomethings()`时，会给他加锁，然后去执行`doOthers()`，又由于执行`doOthers()`时也需要加锁，并且这个锁和`doSomethings()`时加的锁还不一样，所以线程必须把原来的锁给释放掉，但是`doSomethings()`还没执行完，他还不能释放掉锁，这个时候就会形成死锁。

但是如果是可重入锁的话，就能解决这个问题。可重入锁就能保证他在调用`doOthers()`的时候直接获取当前对象的锁，进入`doOthers()`进行操作。

这块还是引用[这篇博客](https://tech.meituan.com/2018/11/15/java-lock.html)的图片：

还是打水的例子，有多个人在排队打水，此时管理员允许锁和同一个人的多个水桶绑定。这个人用多个水桶打水时，第一个水桶和锁绑定并打完水之后，第二个水桶也可以直接和锁绑定并开始打水，所有的水桶都打完水之后打水人才会将锁还给管理员。这个人的所有打水流程都能够成功执行，后续等待的人也能够打到水。这就是可重入锁。
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749464893016.png)

但如果是非可重入锁的话，此时管理员只允许锁和同一个人的一个水桶绑定。第一个水桶和锁绑定打完水之后并不会释放锁，导致第二个水桶不能和锁绑定也无法打水。当前线程出现死锁，整个等待队列中的所有线程都无法被唤醒。
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749464984252.png)

那下面我们来看下源码，看ReentrantLock的获取锁和释放锁的方法：
```java
/** 
 * 获取锁，也是非公平锁的获取锁的方法
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取State变量
    int c = getState();
    // 如果state变量为0，那么没有其他线程在执行同步方法，就将state置1，当前线程开始执行
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 判断当前线程是否是获取到锁的线程，如果是就进入if
    else if (current == getExclusiveOwnerThread()) {
        // 让c的值加上acquires的值，代表可以再次获取锁
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 跟新state
        setState(nextc);
        return true;
    }
    return false;
}

/** 
 * 释放锁
 */
protected final boolean tryRelease(int releases) {
    // state-1
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果state-1为0，代表当前线程所有重复获取锁的操作都已经执行完毕，就真正的释放锁
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 更新state
    setState(c);
    return free;
}
```


# 独享锁和共享锁
独享锁也是排他锁，也就是说该锁一次只能被一个线程持有。如果线程对数据加上排他锁之后，其它线程就不能在对数据加任何的锁。获得排他锁的线程既能读数据也能写数据。

而共享锁则可被多个线程持有。当线程对数据加上共享锁之后，其它线程也能对该数据加锁，但是只能加共享锁，这时获得共享锁的线程只能读数据不能写数据。

在Java中，ReentrantLock是共享锁，ReentrantReadWriteLock是排他锁。

先来看ReentrantReadWriteLock：
```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** Inner class providing readlock */
    // 读锁
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    // 写锁
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;
```
我们可以看到，在ReentranReadWriteLock中有两个锁：读锁ReaderLock和写锁WriterLock。

```java
-------------------读锁-------------------
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
    private final Sync sync;
    
    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

-------------------写锁-------------------
public static class WriteLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -4992448646407690164L;
    private final Sync sync;

    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }
```

他两都是通过内部的Sync实现的锁。但是读锁和写锁的加锁方式不一样，读锁是共享锁，写锁是独享所。读锁的共享锁可保证并发读非常高效，而读写、写读、写写的过程互斥，因为读锁和写锁是分离的。所以ReentrantReadWriteLock的并发性相比一般的互斥锁有了很大提升。

而且他两的state也很有特色，我们上面说过，state用于描述有多少个线程持有该锁。在独享锁中，这个线程通常是0或者1，而共享锁中就是持有锁的数量。但是ReentrantReadWriteLock中有两把锁，但是他们都通过同一个state来描述，所以这地方就需要有点特别的东西了。

ReentrantReadWriteLock将state分隔成了两部分，高16位代表读锁，低16位代表写锁：
![](https://cdn.littlecorgi.top/mweb/2019-11-28/15749488946683.png)


接下来再来看加锁的代码：
```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough: （Google机翻）
     * 1. 如果读取计数非零或写入计数非零并且所有者是另一个线程，则失败。
     * 2. 如果计数饱和，则失败。 
     *       (只有在count已经不为零时，才可能发生这种情况。)
     * 3. 否则，
     *       如果该线程是可重入获取或队列策略允许的话，则有资格进行锁定。 
     *       如果是这样，请更新状态并设置所有者。
     */
    Thread current = Thread.currentThread();
    // 获取state
    int c = getState();
    // 取写锁的个数w
    int w = exclusiveCount(c);
    // 如果已经有线程持有了锁
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // 如果w为0，也就是存在读锁，或者持有所的线程不是当前线程，返回false
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 如果写入锁的数量大于最大数就抛出一个Error
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    // 如果当且写线程数为0，并且当前线程需要阻塞那么就返回失败；或者如果通过CAS增加写线程数失败也返回失败
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 如果c=0，w=0或者c>0，w>0（重入），则设置当前线程或锁的拥有者
    setExclusiveOwnerThread(current);
    return true;
}
```

> * 这段代码首先取到当前锁的个数c，然后再通过c来获取写锁的个数w。因为写锁是低16位，所以取低16位的最大值与当前的c做与运算（ int w = exclusiveCount©; ），高16位和0与运算后是0，剩下的就是低位运算的值，同时也是持有写锁的线程数目。
> * 在取到写锁线程的数目后，首先判断是否已经有线程持有了锁。如果已经有线程持有了锁(c!=0)，则查看当前写锁线程的数目，如果写线程数为0（即此时存在读锁）或者持有锁的线程不是当前线程就返回失败（涉及到公平锁和非公平锁的实现）。
> * 如果写入锁的数量大于最大数（65535，2的16次方-1）就抛出一个Error。
> * 如果当且写线程数为0（那么读线程也应该为0，因为上面已经处理c!=0的情况），并且当前线程需要阻塞那么就返回失败；如果通过CAS增加写线程数失败也返回失败。
> * 如果c=0,w=0或者c>0,w>0（重入），则设置当前线程或锁的拥有者，返回成功！

tryAcquire()除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。如果存在读锁，则写锁不能被获取，原因在于：必须确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。

因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，然后等待的读写线程才能够继续访问读写锁，同时前次写线程的修改对后续的读写线程可见。

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    // 获取state
    int c = getState();
    // 如果其他线程已经获取了写锁，则当前线程获取读锁失败。
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 获取读锁state
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

可以看到在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是“1<<16”。所以读写锁才能实现读读的过程共享，而读写、写读、写写的过程互斥。