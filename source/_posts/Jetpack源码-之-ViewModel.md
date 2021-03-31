---
title: Jetpack源码 之 ViewModel
categories:
  - Android
tags:
  - Android
  - Jetpack
  - MVVM
  - ViewModel
cover: 'https://cdn.littlecorgi.top/Android%20Jetpack%E5%A4%B4%E5%9B%BE.png'
abbrlink: b309316d
date: 2020-09-10 11:23:56 
updated: 2020-09-10 11:23:56
---
# 0. 前言
ViewModel作为MVVM中最重要的一层，他的作用就是对数据状态的持有与维护。

根据源码里面的注释，我们可以知道，它在Android中事实上是为了解决一下两个问题：
- UI组件间实现数据共享
- Activity配置更改重建时保留数据

对于第一条，如果不同VM，那么各个UI组件都需要持有共享数据的引用，这样会带来两个麻烦：
- 如果新增共享数据，则各个UI组件需要再次声明并初始化新增的共享数据
- 某个组件对于数据的修改，没办法直接通知其他UI组件，需手动实现观察者模式

对于第二条，如果不使用VM，那么还是可以通过onSaveInstanceState保存的，但是如果数据量比较大，数据的序列化和反序列化都会产生一定的性能开销。

所以我们看ViewModel的源码，就需要从这两个问题入手：
- ViewModel是如何解决UI组件间共享数据的
- ViewModel是怎么解决重建时保留数据的


# 1. ViewModel是什么
我们直接看下源码：
```java
public abstract class ViewModel {
    @Nullable
    private final Map<String, Object> mBagOfTags = new HashMap<>();
    // 用于标记当前ViewModel是否已经不再被使用
    private volatile boolean mCleared = false;

    // 这个方法会在ViewModel不再被使用并且即将被销毁时调用
    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }
    
    // 注释里面写到：
    //   由于clear()方法是final的，所以这个方法仍然会被在模拟对象中被调用，
    //   并且在这种情况下，mBagOfTags为null
    @MainThread
    final void clear() {
        mCleared = true;
        if (mBagOfTags != null) {
            synchronized (mBagOfTags) {
                for (Object value : mBagOfTags.values()) {
                    closeWithRuntimeException(value);
                }
            }
        }
        onCleared();
    }

    @SuppressWarnings("unchecked")
    <T> T setTagIfAbsent(String key, T newValue) {
        T previous;
        // 当当前key不存在时才将数据保存进去
        // 类似于ConcurrentHashMap的 putIfAbsent()
        synchronized (mBagOfTags) {
            previous = (T) mBagOfTags.get(key);
            if (previous == null) {
                mBagOfTags.put(key, newValue);
            }
        }
        
        T result = previous == null ? newValue : previous;
        // 当mCleared时，异常
        if (mCleared) {
            closeWithRuntimeException(result);
        }
        return result;
    }
    @SuppressWarnings({"TypeParameterUnusedInFormals", "unchecked"})
    <T> T getTag(String key) {
        if (mBagOfTags == null) {
            return null;
        }
        synchronized (mBagOfTags) {
            return (T) mBagOfTags.get(key);
        }
    }

    /**
     * 用于关闭
     */
    private static void closeWithRuntimeException(Object obj) {
        if (obj instanceof Closeable) {
            try {
                ((Closeable) obj).close();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```
上面基本上对ViewModel这个抽象类有了一个大致的了解，那接下来我们来看下ViewModel是如何被创建的。

# 2. ViewModel的创建
一般情况下，我们都会通过这个代码创建ViewModel：
```kotlin
mViewModel = ViewModelProvider(this).get(MvvmViewModel::class.java)
```

这里的this要求传入ViewModelStoreOwner，一般我们的Activity和Fragment都实现了这个接口。我们来看下这个ViewModelProvider的构造方法：
## 2.1 ViewModelProvider
```java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}

public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
    this(owner.getViewModelStore(), factory);
}

public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}

/**
 * 注释1：ComponentActivity # getDefaultViewModelProviderFactory()
 */
public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mDefaultFactory == null) {
        mDefaultFactory = new SavedStateViewModelFactory(
                getApplication(),
                this,
                getIntent() != null ? getIntent().getExtras() : null);
    }
    return mDefaultFactory;
}

/**
 * 注释2：Fragment # getDefaultViewModelProviderFactory()
 */
public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
    if (mFragmentManager == null) {
        throw new IllegalStateException("Can't access ViewModels from detached fragment");
    }
    if (mDefaultFactory == null) {
        Application application = null;
        Context appContext = requireContext().getApplicationContext();
        while (appContext instanceof ContextWrapper) {
            if (appContext instanceof Application) {
                application = (Application) appContext;
                break;
            }
            appContext = ((ContextWrapper) appContext).getBaseContext();
        }
        if (application == null && FragmentManager.isLoggingEnabled(Log.DEBUG)) {
            Log.d(FragmentManager.TAG, "Could not find Application instance from "
                    + "Context " + requireContext().getApplicationContext() + ", you will "
                    + "not be able to use AndroidViewModel with the default "
                    + "ViewModelProvider.Factory");
        }
        mDefaultFactory = new SavedStateViewModelFactory(
                application,
                this,
                getArguments());
    }
    return mDefaultFactory;
}
```

我们调用的是第一个构造方法，这个方法中调用了第三个构造方法，然后通过`getViewModelStore()`获取到了ViewModelStore，并判断我们这个`owner`是不是HasDefaultViewModelProviderFactory子类，如果是就调用HasDefaultViewModelProviderFactory的`getDefaultViewModelProviderFactory()`否则调用`NewInstanceFactory.getInstance()`。

>这块还是别看AndroidStudio自带的SDK的源码了，这个时候最新的源码是API30，但是ComponentActivity并没有实现HasDefaultViewModelProviderFactory，而从[官方Github](https://github.com/androidx/androidx/blob/androidx-master-dev/activity/activity/src/main/java/androidx/activity/ComponentActivity.java)看到的源码是已经实现了的。

而我们的Activity和Fragment都是实现了这个接口的。
他们的`getDefaultViewModelProviderFactory()`最后都创建了这样一个东西：
```java
mDefaultFactory = new SavedStateViewModelFactory(
        getApplication(),
        this,
        getIntent() != null ? getIntent().getExtras() : null);
```


那我们再来看下`get()`方法：
## 2.1 ViewModelProvider # get()
```java
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    // 通过Class获取到他的类名
    String canonicalName = modelClass.getCanonicalName();
    // 如果没有名字，则证明是局部内部类或者匿名内部类，则抛异常
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    // 从ViewModelStore中取值，刚开始是取不到的，因为没有存内容
    // 如果是已经调用过get方法后再调用这个方法，这个时候是能取出值的
    ViewModel viewModel = mViewModelStore.get(key);

    // 因为第一次进入还没有值，那么这个时候取出的是null，isInstance也就返回false了
    // 如果是已经被缓存之后再调用的这个就能取出值，那么这个时候一般返回true
    if (modelClass.isInstance(viewModel)) {
        // SavedStateViewModelFactory是OnRequeryFactory的孙类(子类的子类)
        // 所以返回true，进入if
        if (mFactory instanceof OnRequeryFactory) {
            // 获取到ViewModel
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        // create方法中通过反射的newInstance去创建对象
        // ->> 注释1
        viewModel = (mFactory).create(modelClass);
    }
    // 存起来
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}

/**
 * 注释1：NewInstanceFactory # create()
 * 作用：通过反射去创建对象
 */
@NonNull
@Override
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    try {
        return modelClass.newInstance();
    } catch (InstantiationException e) {
        throw new RuntimeException("Cannot create an instance of " + modelClass, e);
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Cannot create an instance of " + modelClass, e);
    }
}
```

到这块，我们的创建ViewModel就说完了，最后总结下：
- ViewModelStore是用来保存ViewModel对象的HashMap，Activity和Fragment都会构造这个的对象；
- ViewModelProvider中会通过`get()`方法创建一个ViewModel，创建之前会检测ViewModelStore中有没有缓存了的，如果有直接返回，没有就通过反射去创建。

# 3. 重建时如何保证ViewModel不会重建
了解到VieModel的创建后，我们现在来看第一个问题：ViewModel是怎么解决重建时保留数据的？

我们回顾下刚刚创建的ViewModel的流程，在`get()`方法中主要是通过ViewModelStore来帮我们保存ViewModel的，那么ViewModelStore是怎么创建的？

## 3.1 ViewModelStore的创建
### 3.1.1 Activity中
在ComponentActivity中有`getViewModelStore()`这样的一个方法：
```java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // 通过NonConfigurationInstances获取到ViewModelStore
            mViewModelStore = nc.viewModelStore;
        }
        // 如果获取不到，则新建一个ViewmodelStore
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
```

这块就涉及到了一个NonConfigurationInstances，而熟悉Activity重建机制的小伙伴应该会很熟悉这个，这个就是与我们的重建机制有关。

当我们需要重建Activity的时候，除了通过`onSaveInstanceState()`保存数据之外，也可以通过`onRetainNonConfigurationInstance()`这个方法：
#### ComponentActivity # onRetainNonConfigurationInstance()
```java
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();

    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // 如果是null，说明以前没有调用过 getViewModelStore()方法,
        // 也就是没有调用过ViewModelProvider(requireActivity()).get(DemoViewModel::class.java)的方法来获取ViewModel。
        // 所以我们看一下最后一个的NonConfigurationInstance里面是否存在viewModelStore
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        // 如果nc 不等于null，就证明以前存储过，所以从这里取出来
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }

    if (viewModelStore == null && custom == null) {
        return null;
    }
    // 如果viewModelStore不是null，也就是说最后一个NonConfigurationInstance里面有值，直接new出来NonConfigurationInstances并赋值，返回出去
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

#### 那么onRetainNonConfigurationInstance()什么时候回被调用呢
在Activity的启动流程中，当ActivityThread执行到`performDestroyActivity()`这个方法时，就会调用Activity的`retainNonConfigurationInstances()`方法将保存的数据保存到ActivityClientRecord中：
```java
NonConfigurationInstances retainNonConfigurationInstances() {
    // 调用了 onRetainNonConfigurationInstance() 获取到 NonConfigurationInstances
    Object activity = onRetainNonConfigurationInstance();
    // ...

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.activity = activity;
    nci.children = children;
    nci.fragments = fragments;
    nci.loaders = loaders;
    // ...
    return nci;
}
```
可以看到他先调用`onRetainNonConfigurationInstance()`获取到ComponentActivity返回来的`nci`，然后由构建了一个`nci`，并将之前ComponentActivity的那个`nci`保存到了这个`nci`的`activity`中。

#### ActivityThread # performDestroyActivity() 保存nci
```java
ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
        int configChanges, boolean getNonConfigInstance, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    // ...
    if (r != null) {
        // ...
        if (getNonConfigInstance) {
            try {
                r.lastNonConfigurationInstances
                        = r.activity.retainNonConfigurationInstances();
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException(
                            "Unable to retain activity "
                            + r.intent.getComponent().toShortString()
                            + ": " + e.toString(), e);
                }
            }
        }
        // ...
    }
    // ...
    return r;
}
```

#### 那么什么时候恢复呢
当页面重构完成，就会调用ActivityThread的`performLaunchActivity()`：
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);
    // ...
    return activity;
}
```

#### Activity # attach()
```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    // ...
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    // ...
}
```

这块就将值赋给了Activity的`mLastNonConfigurationInstances`。

然后我们在回到3.1.1节的`getViewModelStore()`方法中，如果调用` getLastNonConfigurationInstance()`返回的`nc`不为null的话，就直接取出他的`viewModelStore`，这样我们不就实现了Activity重建但是ViewModel仍然不会重建的问题嘛。

### 3.1.2 Fragment中
一样的，也来看下Fragment的`getViewModelStore()`方法
```java
public ViewModelStore getViewModelStore() {
    if (mFragmentManager == null) {
        throw new IllegalStateException("Can't access ViewModels from detached fragment");
    }
    return mFragmentManager.getViewModelStore(this);
}
```

可以看到他里面调用了FragmentManager的`getViewModelStore()`方法:

#### FragmentManagerImpl # getViewModelStore()
```java
ViewModelStore getViewModelStore(@NonNull Fragment f) {
    return mNonConfig.getViewModelStore(f);
}
```

这里面直接调用了mNonConfig的getViewModelStore方法。

那么这个mNonConfig是个啥呢？

```java
private FragmentManagerViewModel mNonConfig;
```

就是一个FragmentManagerViewModel，那么他是在哪赋值的：

#### FragmentManagerImpl # attachController()
```java
public void attachController(@NonNull FragmentHostCallback host,
        @NonNull FragmentContainer container, @Nullable final Fragment parent) {
    // ...
    // 当parent不为null时，就获取到他的FragmentManager的getChildNonConfig
    if (parent != null) {
        mNonConfig = parent.mFragmentManager.getChildNonConfig(parent);
    } else if (host instanceof ViewModelStoreOwner) {
        // 一般都会进入到这个分支中，执行下面代码
        ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
        mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
    } else {
        mNonConfig = new FragmentManagerViewModel(false);
    }
}
```

这个方法中就是根据不同的分支返回不同的mNonConfig，而这个方法则会被FragmentController的attachHost调用:

#### FragmentController # attachHost()
```java
public void attachHost(@Nullable Fragment parent) {
    mHost.mFragmentManager.attachController(
            mHost, mHost /*container*/, parent);
}
```

而这个方法接着会被FragmetActivity中的onCreate调用。

#### FragmentActivity # onCreate()

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);
    // ...
}
```

而attachHost中的那个host则是在FragmentController构造方法中传入的：
```java
private FragmentController(FragmentHostCallback<?> callbacks) {
    mHost = callbacks;
}

public static FragmentController createController(@NonNull FragmentHostCallback<?> callbacks) {
    return new FragmentController(checkNotNull(callbacks, "callbacks == null"));
}

final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
```

而这个HostCallbacks则实现了ViewModelStoreOwner：
```java
class HostCallbacks extends FragmentHostCallback<FragmentActivity> implements
        ViewModelStoreOwner,
        OnBackPressedDispatcherOwner {
```

# 4. UI组件间数据共享
只要我们在调用`ViewModelProvider(this).get(MvvmViewModel::class.java)`时传入的this是同一个，那么就能获取到同一个ViewModelStore，对应的返回的ViewModel也是同一个了。