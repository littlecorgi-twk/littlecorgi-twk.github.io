---
title: Jetpack源码 之 Lifecycle
categories:
  - Android
tags:
  - Android
  - Jetpack
  - Lifecycle
cover: 'https://cdn.littlecorgi.top/Android%20Jetpack%E5%A4%B4%E5%9B%BE.png'
description: Lifecycle是Jetpack中最基础的一类库，他的主要作用就是帮我们实现自动的生命周期监听，那么她是怎么去实现的呢？就让我们一起来看看吧
abbrlink: 53c3ccc1
date: 2020-08-14 16:49:20
updated: 2020-08-14 16:49:20
---
# 0. 前言
## 0.1 用法
Lifecycle可以说是Jetpack中最基础的一个库，他的主要作用就是帮我们实现的生命周期监听。

对于他的用法也很简单，由于我们的Activity(间接通过`ComponentActivity`实现)、Fragment(直接实现)都已经实现了`LifecycleOwner`接口，所以我们可以直接在他们中调用`getLifecycle()`获得到`Lifecycle`对象，然后调用他的`addObserver()`将我们自定义的`LifecycleObserver`传入进入即可。
```kotlin
/* 以Activity为例 */

// MainActivity
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_lifecycle_test)
        lifecycle.addObserver(LifecycleObserverTest)
    }
    //...
}

// LifecycleObserverTest
object LifecycleObserverTest : LifecycleObserver {
    private const val TAG = "LifecycleTest"

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun prepare() {
        // todo 播放器准备工作
        Log.d(TAG, "prepare: Create时播放器准备工作")
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun release() {
        // todo 释放资源
        Log.d(TAG, "release: Destroy时释放资源")
    }

    fun play(context: Context) {
        // todo 开始播放
        Log.d(TAG, "play")
    }
}
```
这样当MainActivity执行`onCreate()`时会自动执行`prepare()`方法，执行`onDestroy()`时会自动执行`release()`方法，就不需要我们手动的去调用了。

但是这块需要注意的是，这些添加了生命周期的方法，一定不要传入任何参数，因为都已经是自动调用的了，也就我们手动干预不了，那么系统怎么知道你要传入的参数是谁呢。没有添加生命周期的方法不受影响。

另外一个注意的点是，我们调用`addObserver()`并不是一定得在`onCreate()`方法中调用，我们在任何地方任何生命周期时调用即可，比方说我们上面在`onCreate()`中调用了，这样打印出来的log就如下所示：
![Jetpack源码 之 Lifecycle-1-](https://cdn.littlecorgi.top/mweb/2020-08-14/Jetpack%E6%BA%90%E7%A0%81%20%E4%B9%8B%20Lifecycle-1-.png)

但是如果我们把`addObserver()`放在`onResume()`中调用，结果就变成了这样：
![Jetpack源码 之 Lifecycle-2-](https://cdn.littlecorgi.top/mweb/2020-08-14/Jetpack%E6%BA%90%E7%A0%81%20%E4%B9%8B%20Lifecycle-2-.png)
我们就会发现，原本应该在`onCreate()`时执行的方法却到了`onResume()`才执行。

所以这点我们一定要注意，我们调用`addObserver()`一定得在我们监听的生命周期里面或者之前。

## 0.2 真 · 前言
看完刚刚的用法，我们能得到的第一个要素就是Lifecycle一定是通过观察者模式实现的，这个从`addObserver()`就能看出来。

所以我们可以大胆猜想下Lifecycle的实现原理：
当调动`addObserver()`之后，Lifecycle就通过一种数据接口将这个LifecycleObserver的对象保存了起来，当Activity生命周期变化时，他就会遍历这个数据结构，然后调用每一个的对应的生命周期的回调代码。

对应的也就分为两部分：
1. 注册
2. 分发
3. 执行回调

(PS:突然感觉好像EventBus😂)

接下来我们就可以开始看源码，来验证我们的猜想到底正不正确。

# 1. 注册
注册肯定是从`getLifecycle().addObserver(LifecycleObserver observer)`开始嘛。

首先，`getLifecycle()`是谁的方法？

直接通过AndroidStudio的跳转功能就能看到，我们调用的`getLifecycle()`其实是ComponentActivity的一个方法，进一步跳转就能看到其实是LifecycleOwner这个接口的一个抽象方法。

所以也就是说，Activity继承自ComponentActivity，而ComponentActivity实现了LifecycleOwner接口，这个接口中只有一个方法，那就是`getLifecycle()`。(Fragment同理)

而这个方法返回的是Lifecycle，那么Lifecycle里面有些啥东西？

## 1.1 Lifecycle抽象类
进入Lifecycle的源码就能看到它是一个抽象类，代码也很少：
```java
public abstract class Lifecycle {

    AtomicReference<Object> mInternalScopeRef = new AtomicReference<>();
   
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    @MainThread
    public abstract State getCurrentState();

    @SuppressWarnings("WeakerAccess")
    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY
    }

    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```

Lifecycle中有三个方法和两个枚举：
- 三个方法
    - `addObserver()`：Adds a LifecycleObserver that will be notified when the LifecycleOwner changes.  
    （添加一个LifecycleObserver，这个LifecycleObserver在LifecycleOwner变化时能得到通知）
    - `removeObserver()`：Removes the given observer from the observers list.  
    （从observers的集合中移除这个observer）
    - `getCurrentState()`：Returns the current state of the Lifecycle.  
    （返回当前的生命周期状态(State)）
- 两个枚举
    - `Event`：这个不用多说，对应着Activity、Fragment的基本生命周期
    - `State`：这个是返回的当前Activity、Fragment的状态，具体转换看下图：

![关于这个图的来源可以看LifecycleRegistry.getStateAfter()源码](https://cdn.littlecorgi.top/mweb/2020-08-14/15973104823930.jpg)

从上可得知，Lifecycle这个抽象类主要的作用就是
- 添加和移除Observer
- 获取当前LifecycleOwner的状态，并负责对应的状态和事件的转换

那么我们回过头来，`getLifecycle()`要求返回一个Lifecycle对象，但是Lifecycle是一个抽象类，没办法直接构造对象，那么这个方法返回的是谁？
我们直接看ComponentActivity的`getLifecycle()`方法：
```java
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```
而这个`mLifecycleRegistry`是LifecycleRegistry的对象，所以可以得知，Lifecycle其中一个(实际上是唯一)实现类是LifecycleRegistry。

## 1.2 LifecycleRegistry
Lifecycle只有唯一一个实现类，那就是LifecycleRegistry。

刚刚说到，注册肯定是有一个Observer的集合，刚刚Lifecycle源码中的注释也说明了这一点，所以我们先从这个类的属性开始看起：
```java
public class LifecycleRegistry extends Lifecycle {

    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
    
    private State mState;
    
    private final WeakReference<LifecycleOwner> mLifecycleOwner;

    private int mAddingObserverCounter = 0;

    private boolean mHandlingEvent = false;
    private boolean mNewEventOccurred = false;

    // ...
}
```
- `mObserverMap`：用于保存Observer的集合，是一个FastSafeIterableMap集合，而这个FastSafeIterableMap根据源码注释，是类似于LinkedHashMap的，并且它是线程不安全，允许使用迭代器时修改集合的。
- `mState`：当前的状态(State)。就是Lifecycle中的那个State枚举类。
- `mLifecycleOwner`：就是这个LifecycleRegistry的持有者，这块使用了弱引用，是为了避免对Fragment、Activity直接引用而造成内存泄漏。
- `mAddingObserverCounter`：正在添加到mObserverMap中的Observer的数量。
- `mHandlingEvent`：是否正在分发事件的标记。
- `mNewEventOccurred`：是否有新的事件发生的标记。


接着就来分析addObserver方法：
### 1.2.1 addObserver(@NonNull LifecycleObserver observer)
```java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    // 根据状态和observer构造出一个statefulObserver
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    // 通过observer去mObserverMap查找，如果没有这个key或者值为null，就将value保存进去，如果有value，就取出来赋值给previous
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
    // 如果previous不为null，就证明mObserverMap中已经保存了这个observer，就直接return
    if (previous != null) {
        return;
    }
    // 获取到lifecycleOwner弱引用引用对象
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    // 为null，就证明这个Activity或者Fragment已经Destroy了
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }
    // 如果有observer正在被加入或者正在分发时间，这个标记就会为true
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    // 计算出目前的State——targetState。
    // ->>> 见1.2.2
    State targetState = calculateTargetState(observer);
    // 代表有一个Observer正在被添加
    mAddingObserverCounter++;
    // 如果当前这个Observer的状态低于targetState并且mObserverMap中还有这个Observer的话，就需要同步到targetState
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        // 同步时先push进去，临时保存下
        pushParentState(statefulObserver.mState);
        // 分发事件
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        // 分发完成再pop出来
        popParentState();
        // mState / subling may have been changed recalculate
        // 重新计算targetState
        targetState = calculateTargetState(observer);
    }
    // 如果不可重入，就执行sync方法
    if (!isReentrance) {
        // we do sync only on the top level.
        // sync涉及到了事件分发，我们放到后面说
        // ->>> 见 2.2.4
        sync();
    }
    mAddingObserverCounter--;
}
```
所以这个方法主要执行了下面几件事
1. 先构造一个默认状态，要么是`DESTROYED`要么是`INITIALIZED`
2. 然后根据这个`observer`和这个状态构建出一个`statefulObserver`
3. 就去计算`targetState`，并对于这个`statefulObserver`，如果他的`state`小于`targetState`，就进行分发时间，并重新计算`targetState`
4. 如果是不可重入状态，则执行`sync()`方法

### 1.2.2 calculateTargetState(LifecycleObserver observer)
```java
private State calculateTargetState(LifecycleObserver observer) {
    // 取出当前observer的前一个Observer
    Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

    // 拿到previous的State
    State siblingState = previous != null ? previous.getValue().mState : null;
    // 取出parentState最后一个元素的State
    State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
            : null;
    // 取出这些中最小的State
    return min(min(mState, siblingState), parentState);
}                                                                                          
```
所以这个方法的主要作用就是计算`targetState`，这个`targetState`一定小于等于当前`mState`。
也就是说，我们可以添加多个Observer，但是每次添加新的Observer的时候，初始状态都是`INITIALIZED`，这个时候就需要把它同步到当前的生命周期状态。

并且在更新状态的时候，每次更新之后都会调用这个方法再重新计算`targetState`。

上面就完成了事件的添加了，那我们现在再来看下它是怎么分发事件的。

# 2. 分发
我们再次回到ComponentActivity，他在`onCreate()`方法中执行了一行代码：
```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mSavedStateRegistryController.performRestore(savedInstanceState);
    ReportFragment.injectIfNeededIn(this); // !!!
    if (mContentLayoutId != 0) {
        setContentView(mContentLayoutId);
    }
}
```
那我们看一下这个ReportFragment到底是啥：
## 2.1 ReportFragment
### 2.1.1 ReportFragment # injectIfNeededIn()
```java
public static void injectIfNeededIn(Activity activity) {
    // ProcessLifecycleOwner should always correctly work and some activities may not extend
    // FragmentActivity from support lib, so we use framework fragments for activities
    // 获取到FragmentManager
    android.app.FragmentManager manager = activity.getFragmentManager();
    // 判断当前的FragmentManager里面有没有我们需要的Fragment，没有的话就添加进去
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    }
}
```
看到这块是不是感觉很眼熟，Android中最常用的监听生命周期的方法就是往Activity中添加一个没有界面的Fragment，这块正是这个操作，所以我们具体看一下ReportFragment的实现。
### 2.1.2 ReportFragment # 生命周期监听
```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}

@Override
public void onStart() {
    super.onStart();
    dispatchStart(mProcessListener);
    dispatch(Lifecycle.Event.ON_START);
}

@Override
public void onResume() {
    super.onResume();
    dispatchResume(mProcessListener);
    dispatch(Lifecycle.Event.ON_RESUME);
}

@Override
public void onPause() {
    super.onPause();
    dispatch(Lifecycle.Event.ON_PAUSE);
}

@Override
public void onStop() {
    super.onStop();
    dispatch(Lifecycle.Event.ON_STOP);
}

@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
    // just want to be sure that we won't leak reference to an activity
    mProcessListener = null;
}
```

这是ReportFragment中对生命周期监听的所有方法，我们可以看到这些方法都有两个共性：
- 他们都调用了`dispatch()`方法去分发生命周期
- 他们都通知了`mProcessListener`

对于`mProcessListener`，这个是处理应用程序进程的生命周期的，这个我们先不去管它，我们需要重视的是这个`dispatch()`方法

### 2.1.3 ReportFragment # dispatch()
```java
private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```
由于我们探究的是ComponentActivity，所以肯定是走的第8行的if，然后他就会通过`getLifecycle()`拿到`mLifecycleRegistry`，然后调用了他的`handleLifecycleEvent(event)`

## 2.2 LifecycleRegistry
### 2.2.1 LifecycleRegistry # handleLifecycleEvent()
```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```
他首先调用了`getSateAfter()`方法，获取到了当前`event`对应的State（这个对应关系见上面那个State和Event的对应关系图）。
然后调用`moveToSate()`去分发事件以及移动状态。

### 2.2.2 LifecycleRegistry # getStateAfter()
```java
static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}
```
这个也就是我们上面说的Event和State对照图的来源。

### 2.2.3 LifecycleRegistry # moveToState()
```java
private void moveToState(State next) {
    // 当前状态已经是目标状态了，就不需要改变
    if (mState == next) {
        return;
    }
    // 改变当前状态
    mState = next;
    // 如果正在分发状态，或者有Observer正在添加的话
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    // 分发状态标志位设置为true
    mHandlingEvent = true;
    // 调用sync方法进行事件分发
    sync();
    // 分发状态标志位设置为false
    mHandlingEvent = false;
}
```
这个方法就是将`next`同步到`mState`，并进行对应的状态的分发。

### 2.2.4 LifecycleRegistry # sync()
```java
private void sync() {
    // 取出当前的lifecycleOwner，按照我们上面的流程，此处取出的是ComponentActivity
    // (并不一定是这个Activity，只是因为我们是从ComponentActivity入手分析的，也有可能是Fragment)
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    // 如果Owner不存在了，就抛出异常
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    // 通过isSynced去判断，
    // 这个方法主要是判断mObserverMap的队头和队尾元素的State是否相等，以及队尾元素的State是否等于mState
    // 如果均相等则返回true(不需要进行分发)，否则返回false
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        // 如果 mState 小于 mObserverMap 中队头元素的状态值，调用 backwardPass()
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        // 取出队尾元素
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        // 如果mState的状态值大于newest的状态值，则调用forwardPass()
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```
由于这是`mState`已经更新，但是Observers的状态还没更新过，所以肯定是进入第23行的if，也就执行`forwardPass()`方法

### 2.2.5 LifecycleRegistry # forwardPass()
```java
private void forwardPass(LifecycleOwner lifecycleOwner) {
    // 通过iteratorWithAdditions去安全的并在遍历时能遍历到所有新增的元素的遍历方式去遍历mObservers集合
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        // 向上传递事件，直到 observer 的状态值等于当前状态值
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            // 和addObserver中一样的操作
            // 先临时保存
            pushParentState(observer.mState);
            // 分发事件
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            // 移除
            popParentState();
        }
    }
}

/**
 * upEvent()
 * state升级所需要经历的事件
 */
private static Event upEvent(State state) {
    switch (state) {
        case INITIALIZED:
        case DESTROYED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

到这块我们就差不多能理解他的一个流程了，假设我们上面的流程是从`ON_CREATE`到`ON_RESUME`，那么这个流程就是：
1. ReportFragment通过监听生命周期变化，调用了`dispatch(Lifecycle.Event.ON_RESUME)`；
2. 在`dispatch()`中则执行了LifecycleRegistry的`handleLifecycleEvent()`方法，参数是`ON_RESUME`；
3. 通过`getStateAfter()`返回`RESUMED`，在调用`moveToState()`跳转到`RESUMED`；
4. 这时将`mSate`更新为`RESUMED`，然后调用`sync()`方法
5. 在`sync()`方法中，由于`mSate`是`RESUMED`状态，而`mObserverMap`中的状态都是`STARTED`，那么`mState`大于`mObserverMap`，就执行`forwardPass()`方法
6. 最后就会调用`observer.dispatchEvent()`去分发事件

### 2.2.6 LifecycleRegistry # backwardPass()
刚刚看完了上面的代码和流程，那么既然有`forwardPass()`，对应的`backwardPass()`有啥作用呢？

我们还是假设一下流程，我们刚刚说到了`ON_RESUME`，那么我们假设从`ON_RESUME`到`ON_PAUSE`，再来看一下刚刚的流程，和刚刚的流程一样的：
1. ReportFragment通过监听生命周期变化，调用了`dispatch(Lifecycle.Event.ON_PAUSE)`；
2. 在`dispatch()`中则执行了LifecycleRegistry的`handleLifecycleEvent()`方法，参数是`ON_PAUSE`；
3. 通过`getStateAfter()`返回`STARTED`，在调用`moveToState()`跳转到`STARTED`；
4. 这时将`mSate`更新为`STARTED`，然后调用`sync()`方法
5. 在`sync()`方法中，由于`mSate`是`STARTED`状态，而`mObserverMap`中的状态都是`RESUMED`，那么`mState`小于`mObserverMap`，就执行`backwardPass()`方法
6. 最后就会调用`observer.dispatchEvent()`去分发事件

```java
// 和 forwardPass() 差不多
private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            // 
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}

/**
 * downEvent()
 * state降级所需经历的事件
 */
private static Event downEvent(State state) {
    switch (state) {
        case INITIALIZED:
            throw new IllegalArgumentException();
        case CREATED:
            return ON_DESTROY;
        case STARTED:
            return ON_STOP;
        case RESUMED:
            return ON_PAUSE;
        case DESTROYED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

上面就是事件分发的过程。

那么剩下最后一个问题，怎么执行回调方法？

# 3. 执行回调
其实这块我们刚刚已经提到过了，那就是在`sync()`方法中会调用`forwardPass()`和`backwardPass()`方法，这两个方法中都会调用`observer.dispatchEvent(lifecycleOwner, event)`方法，所以我们就从这个入手。

首先我们回顾下，我们添加进去的`observer`到底是谁？
回到`addObserver()`方法，在这个方法内部的第二行和第三行：
```java
ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
```
可以看到我们取出来的`observer`其实就是这块的这个`statefulObserver`，他是调用了ObserverWithState的构造方法构造出来的，并且我们后面也是调用的他的`dispatchEvent()`方法，那我们就直接来看一下这个类：
## 3.1 ObserverWithState
```java
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```
通过这个源码可以看到，它里面有两个参数，一个是`mSate`，这个就是这个Observer当前的状态，另一个就是`mLifecycleObserver`，这个也就是我们传入的Observer，但是他并不是直接拿着我们传入的`observer`使用，而是调用`Lifecycling.lifecycleEventObserver()`返回了一个值，那我们看一下这个方法到底是啥：
### 3.1.1 Lifecycling
#### 3.1.1.1 Lifecycling # lifecycleEventObserver()
```java
@NonNull
static LifecycleEventObserver lifecycleEventObserver(Object object) {
    // 判断是不是这两种类型的对象
    boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
    boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
    // 判断是哪种，返回对应的
    // 但是我们是构造的LifecycleObserver的子类，所以不满足下面这三种if
    if (isLifecycleEventObserver && isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                (LifecycleEventObserver) object);
    }
    if (isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
    }

    if (isLifecycleEventObserver) {
        return (LifecycleEventObserver) object;
    }

    // 获得到他的class对象
    final Class<?> klass = object.getClass();
    // ->>> 备注1
    // 获取 type
    //   GENERATED_CALLBACK 表示注解生成的代码
    //   REFLECTIVE_CALLBACK 表示使用反射
    int type = getObserverConstructorType(klass);
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object);
}

/**
 * 备注1
 * 返回构造方法的类型
 *   先从缓存(sCallbackCache)中找，有则直接返回
 *   没有则调用resolveObserverCallbackType方法获取并将结果缓存
 */
private static int getObserverConstructorType(Class<?> klass) {
    // 判断缓存里面有没有，如果有则直接返回
    Integer callbackCache = sCallbackCache.get(klass);
    if (callbackCache != null) {
        return callbackCache;
    }
    // ->>> 备注2
    int type = resolveObserverCallbackType(klass);
    sCallbackCache.put(klass, type);
    return type;
}

/**
 * 备注2
 */
private static int resolveObserverCallbackType(Class<?> klass) {
    // anonymous class bug:35073837
    // 获得包名，如果为null，则是匿名内部类
    if (klass.getCanonicalName() == null) {
        // 对应返回，表示使用反射
        return REFLECTIVE_CALLBACK;
    }

    // 寻找注解生成的 GeneratedAdapter 类
    Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
    if (constructor != null) {
        sClassToAdapters.put(klass, Collections
                .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
        return GENERATED_CALLBACK;
    }

    // 寻找被OnLifecycleEvent注解的方法
    boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
    if (hasLifecycleMethods) {
        return REFLECTIVE_CALLBACK;
    }

    // 没有找到注解生成的 GeneratedAdapter 类，也没有找到 OnLifecycleEvent 注解，
    // 则向上寻找父类
    Class<?> superclass = klass.getSuperclass();
    List<Constructor<? extends GeneratedAdapter>> adapterConstructors = null;
    if (isLifecycleParent(superclass)) {
        if (getObserverConstructorType(superclass) == REFLECTIVE_CALLBACK) {
            return REFLECTIVE_CALLBACK;
        }
        adapterConstructors = new ArrayList<>(sClassToAdapters.get(superclass));
    }

    // 寻找是否有接口实现
    for (Class<?> intrface : klass.getInterfaces()) {
        if (!isLifecycleParent(intrface)) {
            continue;
        }
        if (getObserverConstructorType(intrface) == REFLECTIVE_CALLBACK) {
            return REFLECTIVE_CALLBACK;
        }
        if (adapterConstructors == null) {
            adapterConstructors = new ArrayList<>();
        }
        adapterConstructors.addAll(sClassToAdapters.get(intrface));
    }
    if (adapterConstructors != null) {
        sClassToAdapters.put(klass, adapterConstructors);
        return GENERATED_CALLBACK;
    }

    return REFLECTIVE_CALLBACK;
}
```
其中我们需要注意的是`hasLifecycleMethods`：
```java
boolean hasLifecycleMethods(Class klass) {
    Boolean hasLifecycleMethods = mHasLifecycleMethods.get(klass);
    if (hasLifecycleMethods != null) {
        return hasLifecycleMethods;
    }

    Method[] methods = getDeclaredMethods(klass);
    for (Method method : methods) {
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation != null) {
            // Optimization for reflection, we know that this method is called
            // when there is no generated adapter. But there are methods with @OnLifecycleEvent
            // so we know that will use ReflectiveGenericLifecycleObserver,
            // so we createInfo in advance.
            // CreateInfo always initialize mHasLifecycleMethods for a class, so we don't do it
            // here.
            createInfo(klass, methods);
            return true;
        }
    }
    mHasLifecycleMethods.put(klass, false);
    return false;
}
```
这个代码就没啥好说的，单纯通过反射去找OnLifecycleEvent注解，所以综上所述，我们通过OnLifecycleEvent注解实现的Observer则返回的是`REFLECTIVE_CALLBACK`类型，对应的`lifecycleEventObserver()`方法返回的也是`new ReflectiveGenericLifecycleObserver(observer)`。

### 3.1.2 ReflectiveGenericLifecycleObserver
```java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    // 我们传入的observer对象
    private final Object mWrapped;
    // 通过反射从这个observer对象中获取的信息
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        // 通过反射来获取信息，但是代码太长，也很简单，就不详细去说了
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```
而我们之前的`observer.dispatchEvent()`方法中实际上调用的是`mLifecycleObserver.onStateChanged(owner, event)`，所以最后会交给ReflectiveGenericLifecycleObserver的`onStateChanged()`方法来执行，而这个方法中又调用了`mInfo.invokeCallbacks()`：
```java
void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
    invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
    invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
            target);
}

private static void invokeMethodsForEvent(List<MethodReference> handlers,
        LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
    if (handlers != null) {
        for (int i = handlers.size() - 1; i >= 0; i--) {
            handlers.get(i).invokeCallback(source, event, mWrapped);
        }
    }
}

void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
    //noinspection TryWithIdenticalCatches
    try {
        switch (mCallType) {
            case CALL_TYPE_NO_ARG:
                mMethod.invoke(target);
                break;
            case CALL_TYPE_PROVIDER:
                mMethod.invoke(target, source);
                break;
            case CALL_TYPE_PROVIDER_WITH_EVENT:
                mMethod.invoke(target, source, event);
                break;
        }
    } catch (InvocationTargetException e) {
        throw new RuntimeException("Failed to call observer method", e.getCause());
    } catch (IllegalAccessException e) {
        throw new RuntimeException(e);
    }
}
```
这个代码就很简单了，就是利用反射去执行。