---
title: Jetpack源码 之 LiveData
categories:
  - Android
tags:
  - Android
  - Jetpack
  - MVVM
  - LiveData
cover: 'https://cdn.littlecorgi.top/Android%20Jetpack%E5%A4%B4%E5%9B%BE.png'
abbrlink: 28011e4e
date: 2020-08-22 16:24:50
updated: 2020-08-22 16:24:50
description: LiveData是Jetpack中一个响应式开发框架，官方文档对它的说明是一种可观察的数据存储器类，具有生命周期感知能力。有点类似于感知生命周期的RxJava。
---
# Jetpack源码 之 LiveData
# 0. 前言
LiveData是Jetpack中一个响应式开发框架，官方文档对它的说明是一种可观察的数据存储器类，具有生命周期感知能力。有点类似于感知生命周期的RxJava。

## 0.1 用法
通常LiveData都是结合着ViewModel使用的，一般都是在ViewModel中创建LiveData：
```kotlin
class MvvmViewModel : ViewModel() {

    // 通过MutableLiveData创建一个可读可写的LiveData
    // 设置为Private，避免外部对数据直接进行修改，并暴露对外接口，让外部通过接口来修改
    private val _count = MutableLiveData(0)
    // 暴露给外部一个只读的LiveData副本，让外部监听数据通过此LiveData监听
    val count: LiveData<Int>
        get() = _count

    fun increaseCount() {
        _count.value = _count.value?.plus(1)
    }

    fun clearCount() {
        _count.value = 0
    }
}
```
## 0.2 源码
LiveData源码其实挺简单的，但是在看他的源码之前得先了解Lifecycle的源码，因为LiveData其实是大量通过Lifecycle实现的。关于Lifecycle的源码我们之前看过了，所以此篇博客不会讨论Lifecycle的相关问题。

我们在阅读源码前，首先得清除我们需要从源码里面搞懂哪些问题：
1. 首先，我们在用法上有MutableLiveData和LiveData， 那么他们的区别是啥
2. 官方给LiveData定义是一个可被观察的数据存储类，那么他的可被观察是怎么实现的
3. 官方还说他是生命周期感知的，那么是怎么实现的

带着这三个问题，我们来看源码：

# 1. MutableLiveData
这个类是我们用来可读可写的LiveData，我们对值的修改都是通过这个类的，那我们来看下他的源码：
```java
public class MutableLiveData<T> extends LiveData<T> {

    public MutableLiveData(T value) {
        super(value);
    }
    
    MutableLiveData() {
        super();
    }

    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```
源码就这么点，全是调用父类的方法，而他的父类就是LiveData类。
那么既然这些方法都是调用的LiveData的，那么为什么我们不直接使用LiveData而要去使用MutableLiveData呢？

当你看到LiveData源码时就能知道，LiveData虽然有这些方法，但是他是一个抽象类，没办法直接构造对象，所以我们就需要通过MutableLiveData来操作。
同时，LiveData的`setValue()`和`postValue()`方法都是被protected修饰的，所以我们在外部并没有办法直接访问到，而MutableLiveData的这两个方法是public的，所以可以在外部直接调用。

# 2. LiveData
## 2.1 基本属性
```java
public abstract class LiveData<T> {
    // 数据锁，通过Synchronized线程同步
    @SuppressWarnings("WeakerAccess") /* synthetic access */
    final Object mDataLock = new Object();
    // LiveData的初始化版本
    // 如果构造方法中没有给值，那么 mVersion 直接使用此值
    // 如果给值，那么 mVersion=START_VERSION+1
    // 见下面构造方法
    static final int START_VERSION = -1;
    // mData的默认值
    // 如果构造方法传值，那么mData使用传入的值
    // 如果没有，则使用此值
    // 见下面构造方法
    @SuppressWarnings("WeakerAccess") /* synthetic access */
    static final Object NOT_SET = new Object();

    // 一个SafeIterableMap，用来保存监听的对象，key是观察者对象，value是观察者和mActive、mVersion构成的一个对象
    // SafeIterableMap是一个可以安全递归的HashMap
    private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();

    // 活跃的Obsever的数量
    @SuppressWarnings("WeakerAccess") /* synthetic access */
    int mActiveCount = 0;
    // 当前LiveData的数据
    private volatile Object mData;
    // 当setData被调用时，我们就设置挂起的数据，实际的数据交换发生在主线程上
    // 咋一看有点迷，看不懂这个注释的意思，但是当我们看完postValue方法后再来看这个就能理解了
    @SuppressWarnings("WeakerAccess") /* synthetic access */
    volatile Object mPendingData = NOT_SET;
    // LiveData版本
    private int mVersion;

    // 是否正在分发数据的flag
    private boolean mDispatchingValue;
    // 是否分发无效的flag
    @SuppressWarnings("FieldCanBeLocal")
    private boolean mDispatchInvalidated;
    // 实现同步锁更新数据的线程，在postValue中被调用，会被发送到主线程中去执行
    private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            // 将postValue发过来的数据进行修改
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            // 调用setValue更新数据
            setValue((T) newValue);
        }
    };
    
    public LiveData(T value) {
        mData = value;
        mVersion = START_VERSION + 1;
    }

    public LiveData() {
        mData = NOT_SET;
        mVersion = START_VERSION;
    }
```

## 2.2 LiveData # setValue()
```java
// 通过注解表明只能在主线程使用
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    // 版本增加
    mVersion++;
    // 更新数据
    mData = value;
    // 分发事件
    dispatchingValue(null);
}
```
这个代码很简单，增加版本、更新数据、分发事件。
因为这个方法只能在主线程中被调用(@MainThread)，所以实现才这么简单，不需要考虑线程同步啥的。

## 2.3 LiveData # postValue()
```java
protected void postValue(T value) {
    boolean postTask;
    // 对数据更新加锁
    synchronized (mDataLock) {
        // 如果没有设置过值，那么mPendingData == NOT_SET
        // 先将值保存到mPendingData，在通过下面的Runable来更新数据，将mPendingData的值更新到mValue
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    // postTask为false，就代表数据还没有更新到mValue中去，就不让更新数据
    // 因为postValue和mPostValueRunnable执行之间还有一定的时间间隔
    if (!postTask) {
        return;
    }
    // 通过线程池将数据更新发送到主线程中去更新数据
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```

postValue会比setValue复杂一点，因为我们可以从最后一行看出，postValue是提供给我们在其它线程中调用的，然后调用之后他就会通过线程池传回主线程，再在主线程中更新数据并调用setValue。

那么为什么会需要mPendingData这个中间量呢?
原因也很简单，因为我们最终是要在主线程中调用setValue的，那么我们怎么把最新的值传递给setValue呢，况且这块还涉及到了线程切换，所以我们就必须通过一个中间变量来将这个数据从子线程传输到主线程了。并且由于为了避免多线程情况下对数据进行修改造成数据混乱，所以才对mPendingData加了锁，并且还在postValue处如果数据还没更新的话，后面来的数据都直接丢弃了。

## 2.4 LiveData # observer()
上面说到了设置数据，那么我们数据设置之后谁来相应呢？
我们在使用的时候都是通过observer方法来注册观察者的，那我们就看下这个方法：
```java
// 只许主线程中调用
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    // Activity和Fragment都实现了Owner接口，所以这个Owner就相当于他们
    // 当他们的状态是DESTROYED时，代表刚创建或者要摧毁了，此时就不需要注册了
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    // 构造一个LifecycleBundleOvserver
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    // 存入mObservers这个map中去
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    // 如果已经存在了，并且他的owner并不是我们调用这个方法时传入的owner，就抛出异常
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    // 如果已经存在，就没必要再添加了
    if (existing != null) {
        return;
    }
    // 通过Lifecycle去监听和分发事件
    owner.getLifecycle().addObserver(wrapper);
}
```
observer方法的源码很简单。

先根据传入的owner和observer构造一个LifecycleBoundObserver，然后把他们存入到mObservers这个SafeIterableMap中去，然后在存的时候，如果原本就有值，并且这个的owner还不是我们这次传入的这个owner，就抛出异常，如果原本有值，那么就直接返回，也就不存了。然后最后还是调用Lifecycle的addObserver存入到Lifecycle中去。

## 2.5 LiveData # dispatchingValue()
```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    // 是否有事件正在被分发
    if (mDispatchingValue) {
        // 分发无效
        mDispatchInvalidated = true;
        return;
    }
    // 由于当前需要分发事件，于是这个flag为true
    mDispatchingValue = true;
    do {
        // 正在分发，所以此flag为false
        mDispatchInvalidated = false;
        // 传入的不为null
        if (initiator != null) {
            // 通过considerNotify去分发
            considerNotify(initiator);
            initiator = null;
        } else {
        // 传入的为null，也就是setValue中的调用
            //遍历mObservers，并且还是能遍历被添加的数据
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                // 通过considerNotify去分发
                considerNotify(iterator.next().getValue());
                // 分发无效就break
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    // 分发结束，设置为false
    mDispatchingValue = false;
}
```

## 2.6 LiveData # considerNotify()
```java
private void considerNotify(ObserverWrapper observer) {
    // 如果不是active状态了，就不用分发了
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    // 如果状态不是STARTED之后的话，那么就更新状态
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    // 如果Observer的Version高于当前Version，那么就没必要去分发
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    // 更新Version
    observer.mLastVersion = mVersion;
    // 事件分发，调用我们实现的Observer的onChanged方法，也就是我们写的代码
    observer.mObserver.onChanged((T) mData);
}
```


# 3. LifecycleWrapper
## 3.1 LifecycleWrapper
```java
private abstract class ObserverWrapper {
    // Observer
    final Observer<? super T> mObserver;
    // 当前Observer的状态
    boolean mActive;
    // Observer的最新的Version
    int mLastVersion = START_VERSION;
    // 构造方法
    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }
    
    abstract boolean shouldBeActive();
    
    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        // 更新状态
        mActive = newActive;
        // 如果mActivityCount==0那就意味着没有Observer
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        // 更新mActiveCount
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        if (wasInactive && mActive) {
            // 如果mActive=1，那么就onActive去更新
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            // 如果当前mActiveCount刚刚为0，那么就onInactive
            onInactive();
        }
        // 如果mActive=true，那么就分发
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

## 3.2 LifecycleBoundObserver
```java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        // 判断当前Lifecycle的状态是不是在STARTED之后
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        // 监听Lifecycle的状态发生变化
        // 如果状态时DESTROYED，
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            // 移除掉这个Observer
            removeObserver(mObserver);
            return;
        }
        // 更新Active
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

# 4. 总结
## 4.1 总结和流程
LiveData主要由2个类构成：
- `LiveData`：抽象类，实现类就是`MutableLiveData`，只不过`MutableLiveData`中的所有的方法都是直接调用了`LiveData`的方法，本质上和`LiveData`没啥区别。
- `LifecycleWrapper`：抽象类，主要用来对`Observer`和`Version`、`Active`进行管理，而我们使用的都是`LifecycleBoundObserver`，这个类继承自`LifecycleWrapper`，并实现了`LifecycleEventObserver`接口。

并且底层还是通过Lifecycle去实现的，就是对Lifecycle进行了一下封装。

LiveData的操作主要分为两种：
- 添加观察者：`observer()`
- 设置数据：
    - `setValue()`
    - `postValue()`

### 4.1.1 添加观察者
1. 调用`observer()`方法，
    1. 在这个方法中会先构造一个LifecycleBoundObserver的对象，这个对象保存了这个`mObserver`以及激活状态`mActive`；
    2. 接着把这个对象通过`mObservers.putIfAbsent()`添加到mObservers这个map中去；
    3. 并调用`owner.getLifecycle().addObserver(wrapper)`添加到Lifecycle中进行监听。
2. 之后每当`owner`生命周期变化时，就会调用LifecycleBoundObserver的`onStateChanged()`方法 (具体可见Lifecycle的源码)
    1. 如果当前`owner`的状态为`DESTROYED`的时候，就会将这个`observer`从` mObservers`中移除
    2. 如果不是，就调用`activeStateChanged()`去更新`mActive`

### 4.1.2 设置数据
1. `postValue()`：这个方法主要是提供给我们在非主线程中去调用的
    1. 这个方法中会先加锁，然后将数据保存到`mPendingData`中，并根据`mPendingData`的值判断上次post的值是否更新到`mData`中了：
        1. 如果更新到了，则通过`ArchTaskExecutor`将`mPostValueRunnable`这个Runnable`post`到了主线程中执行
            1. `mPostValueRunnable`中会将加锁，将`mPendingData`中的值更新到`newData`中，并将`mPendingData`值设置为`NOT_SET`，然后会调用`setValue(newData)`
        2. 如果没有更新，则会直接return退出这次`postValue()`，不更新数据，直接丢弃
2. `setValue()`：这个方法只能在主线程中调用
    1. 他就会直接调用`dispatchingValue(null)`进行分发
    2. 这个方法中如果传入的是`null`，就代表需要分发给所有的Observer，那么他会遍历`mObservers`，对每一个Observer调用`considerNotify(iterator.next().getValue());`
    3. 在`considerNotify(observer)`中会先判断传入的这个Observer的`mActive`是否为`false`，如果是的话，就代表没有激活，也就是他的`owner`处于`STARTED`之后，就没有必要去更新View了；而如果不是就继续往下执行：
        1. 先调用`shouldBeActive`判断`owner`是否处于`STARTED`之后，如果是就调用`activeStateChanged(false)`去更新`mActive`为false，并return，不分发事件
        2. 在判断`observer`的`mLastVersion`是否大于当前的`mVersion`，如果大于等于，则也没有必要继续分发
        3. 之后更新`observer`的`mLastVersion`为当前的`mVersion`
        4. 调用`observer.mObserver.onChanged((T) mData);`去执行我们实现的Observer

### 4.1.3 流程图
对于流程我们可以用下图表示：
![Jetpack源码-之-LiveData-LiveData流程图](https://cdn.littlecorgi.top/mweb/2020-08-22/Jetpack%E6%BA%90%E7%A0%81-%E4%B9%8B-LiveData-LiveData%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)


## 4.2 LiveData真的可以被用于事件分发吗
LiveData本质上是可观察的数据存储类，但是可观察这个特性不就可以被用来事件分发吗(EventBus)，而且还通过Lifecycle实现了生命周期感知，这些相对来说比EventBus更加简单。

但是LiveData真的有表面上看上去的这个简单吗？真的有这么好吗？

### 4.2.1 postValue 数据丢失
在postValue方法中，如果mPendingData == NOT_SET的话，就会直接丢弃调这次的数据。也就是说，如果上次的事件还在分发，还没完成的话，那么这次的事件就会直接丢弃。

这样就导致了事件丢失。

### 4.2.2 considerNotify 不回调观察者
除了postValue存放事件这块有问题，considerNotify通知事件也有问题。

considerNotify这个方法第一行就是如果Observer处于非激活状态(mActive == false)，那么这个时候他就不会回调这个Observer去分发事件。

只有当Observer处于激活状态，他才会进行分发。

这样就造成了事件丢失，中间传输的数据都无法收到。

### 4.2.3 LiveData本来就不是为事件分发打造的
在官方文档上第一句话就表明了，LiveData只是一个可观察的数据存储类，他的核心在于他能将最新的数据通知给观察者，也就是说，他本来就不会在意中间状态，它只要保证当View处于激活状态能得到最新的数据保证UI时正确的就行，如果还要去纠结中间状态的话，那么UI展示岂不会变得很奇怪；并且如果对于没有显示的View都要去通知这个View的话，这样不就会显得很多此一举。

