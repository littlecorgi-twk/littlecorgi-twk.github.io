---
title: Android源码之SharedPreferences
categories:
  - Android
tags:
  - Android
  - 源码
  - SharedPreferences
  - 存储
cover: 'https://dignitas.digital/wp-content/uploads/2019/02/SharedAndroid2-e1550504097495.png'
abbrlink: 24e17cf4
date: 2020-07-05 22:09:27
updated: 2020-07-05 22:09:27
---
# SharedPreferences源码分析
# 0. 前言

SharedPreferences可以说是Android中最常用的一种存数据到文件的方式。他的数据是以键值对的方式存储在 `~/data/data/包名/shared_prefs` 这个文件夹中的。

这个存储框架是非常轻量级的，如果我们需要存一些小数据或者是一个小型的可序列化的Bean实体类的，使用SharedPreferences是最明智的选择。

<!--More-->
# 1. 使用方法
## 1.1 获取SharedPreferences
在使用SharedPreferences前，我们得先获取到它。

由于SharedPreferences是Android内置的一个框架，所以我们想要获取到它非常的简单，不需要导入任何依赖，直接写代码就行。下面我们就来介绍下获取对象的三个方式：

### 1.1.1 Context # getSharedPreferences()
首先就是可以说是最常用的方法，通过Context的`getSharedPreferences()` 方法去获取到SharedPreferences对象。由于是通过Context获取的，所以基本上Android的所有场景我们都可以通过这个方法获取到。

```java
public abstract SharedPreferences getSharedPreferences (String name, 
                int mode)
```

这个方法接收两个参数，分别是`name`和`mode`：
- `name`：name就是我们要存储的SharedPreferences本地文件的名字，这个可以自定义。但是如果使用同样的name的话，永远只能获取到同一个SharedPreferences的对象。
- `mode`：mode就是我们要获取的这个SharedPreferences的访问模式，Android给我们提供了挺多的模式的，但是由于其余的模式或多或少存在着安全隐患(因为其他应用也可以直接获取到)，所以就全部都弃用了，现在就只有一个`MODE_PRIVATE`模式。

此外，这个方法是线程安全的。

`Mode`的可选参数：
-  `MODE_PRIVATE`：私有模式，该SharedPreferences只会被调用他的APP去使用，其他的APP无法获取到这个SharedPreferences。
- ~~`MODE_WORLD_READABLE`~~：API17被弃用。使用这个模式，所有的APP都可以对这个SharedPreferences进行读操作。所以这个模式被Android官方严厉警告禁止使用（It is strongly discouraged），并推荐使用`ContentProvider`、`BroadcastReceiver`和`Service`。
- ~~`MODE_WORLD_WRITEABLE`~~：API17被弃用。和上面类似，这个是可以被所有APP进行写操作。同样也是被严厉警告禁止使用。
- ~~`MODE_MULTI_PROCESS`~~：API23被弃用。使用了这个模式，允许多个进程对同一个SharedPreferences进行操作，但是后来也被启用了，原因是因为在某些Android版本下，这个模式不能可靠的运行，官方建议如果多进程建议使用`ContentProvider`去操作。在后面我们会说为啥多进程下不可靠。

### 1.1.2 Activity # getPreferences()
这个方法只能在Activity中或者通过Activity对象去使用。

```java
public SharedPreferences getPreferences (int mode)
```

这个方法需要传入一个`mode`参数，这个参数和上面的`context#getSharedPreferences()`的`mode`参数是一样的。其实这个方法和上面Context的那个方法是一样的，他两都是调用的`SharedPreferences getSharedPreferences(String name, int mode)`。只不过Context的需要你去指定文件名，而这个方法你不需要手动去指定，而是会自动将当前Activity的类名作为了文件名。

### 1.1.3 PreferencesManager # getDefaultSharedPreferences()
这个一般用在Android的设置页面上，或者说，我们也只有在构建设置页面的时候才会去使用这个。

```java
public static SharedPreferences getDefaultSharedPreferences (Context context)
```

他承接一个context参数，并自动将当前应用的报名作为前缀来命名文件。

## 1.2 存数据
如果需要往SharedPreferences中存储数据的话，我们并不能直接对SharedPreferences对象进行操作，因为SharedPreferences没有提供存储或者修改数据的接口。

如果想要对SharedPreferences存储的数据进行修改，需要通过`SharedPreferences.edit()`方法去获取到SharedPreferences.Editor对象来进行操作。

获取到Editor对象后，我们就可以调用他的`putXXX()`方法进行存储了，存储之后一定记得通过`apply()`和`commit()`方法去将数据提交。

至于`commit`和`apply`的区别我们后面会说。

```java
 //步骤1：创建一个SharedPreferences对象
 SharedPreferences sharedPreferences= getSharedPreferences("data",Context.MODE_PRIVATE);
 //步骤2： 实例化SharedPreferences.Editor对象
 SharedPreferences.Editor editor = sharedPreferences.edit();
 //步骤3：将获取过来的值放入文件
 editor.putString("name", “Tom”);
 editor.putInt("age", 28);
 editor.putBoolean("marrid",false);
 //步骤4：提交               
 editor.commit();
 
// 删除指定数据
 editor.remove("name");
 editor.commit();
 
// 清空数据
 editor.clear();
 editor.commit();
```
## 1.3 取数据
取值就很简单了，构建出SharedPreferences的对象后，就直接调用SharedPreferences的`getXXX()`方法就行。

```java
SharedPreferences sharedPreferences = getSharedPreferences("data", Context .MODE_PRIVATE);
String userId = sharedPreferences.getString("name", "");
```

# 2. 源码分析
## 2.1 获取SharedPreferences实例
我们上面说到，获取SharedPreferences实例最常用的方法就是`Context#getSharedPreferences()`。那我们就从这个方法入手，看到底是怎么获取到SharedPreferences实例的。

我们先看下这个方法的实现：
```java
public class ContextWrapper extends Context {
    @UnsupportedAppUsage
    Context mBase;
    
    // ...
    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        return mBase.getSharedPreferences(name, mode);
    }
}
```
可以看到他又调用了Context的`getSharedPreferences()`方法：
```java
public abstract SharedPreferences getSharedPreferences(String name, @PreferencesMode int mode);
```

然后我们就会惊喜的发现，这是一个抽象方法。我开始还想去找一个`ContextWrapper`的构造的地方，看看`mBase`传入的是啥，后来找了一圈没找到，直接上网搜索，立马得到答案：`ContextImpl`，这个可以说是`Context`在Android中的唯一实现类，所有的操作又得经过这个类。那么我们就来看下这个类中的`getSharedPreferences()`方法的实现：
```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
    // At least one application in the world actually passes in a null
    // name.  This happened to work because when we generated the file name
    // we would stringify it to "null.xml".  Nice. 
            // ps:这个nice很精髓😂
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }

    File file;
    // 加了一个类锁，保证同步
    synchronized (ContextImpl.class) {
        // mSharedPrefsPaths是一个保存了name和file对应关系的ArrayMap
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        // 根据name从里面找有没有缓存的file
        file = mSharedPrefsPaths.get(name);
        // 如果没有，那就调用getSharedPreferencesPath去找
        if (file == null) {
            // ->>> 重点1. getSharedPreferencesPath(name)
            file = getSharedPreferencesPath(name);
            // 并保存到mSharedPrefsPaths
            mSharedPrefsPaths.put(name, file);
        }
    }
    // 获取到file后，再调用getSharedPreferences
    return getSharedPreferences(file, mode);
}

/**
 * 重点1. ContextImpl # getSharedPreferencesPath(String name)
 *   根据PreferencesDir和name.xml去创建了这个文件
 */
@Override
public File getSharedPreferencesPath(String name) {
    return makeFilename(getPreferencesDir(), name + ".xml");
}
```

那我们在看下`getSharedPreferences(File file, int mode)`的实现：
```java
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    // SharedPreferences唯一实现类SharedPreferencesImpl的实例
    SharedPreferencesImpl sp;
    // 同样的加类锁
    synchronized (ContextImpl.class) {
        // 构造了一个File-SharedPreferencesImpl对应关系的ArrayMap
        // 调用getSharedPreferencesCacheLocked方法区获取cahce
        // ->>> 重点1
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        // 从file-SharedPreferencesImpl键值对中根据当前file去过去SharedPreferencesImpl实例
        sp = cache.get(file);
        // 如果没有，那就需要新建一个
        if (sp == null) {
            // 检查mode，如果是MODE_WORLD_WRITEABLE或者MODE_MULTI_PROCESS则直接抛异常
            checkMode(mode);
            if (getApplicationInfo().targetSdkVersion >= android.os.Build.VERSION_CODES.O) {
                if (isCredentialProtectedStorage()
                        && !getSystemService(UserManager.class)
                                .isUserUnlockingOrUnlocked(UserHandle.myUserId())) {
                    throw new IllegalStateException("SharedPreferences in credential encrypted "
                            + "storage are not available until after user is unlocked");
                }
            }
            // 调用构造方法去构造SharedPreferencesImpl对象
            sp = new SharedPreferencesImpl(file, mode);
            // 将对象和file的键值对存入cache中
            cache.put(file, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}

/**
 * 重点1. ContextImap # getSharedPreferencesCacheLocked()
 *   根据当前的包名，去获取到由此应用创建的File-SharedPreferencesImpl的Map对象，
 *       而这个对象里面就存放了这个应用创建的所有的SharedPreferencesImpl和File的对应关系
 */
@GuardedBy("ContextImpl.class")
private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    // 如果sSharedPrefsCache为空就构造一个ArrayMap
    // sSharedPrefsCache就是一个存放String-String, ArrayMap<File, SharedPreferencesImpl>的Map
    // 换句话说，也就是存放包名-packagePrefs对应关系的Map
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    // 获取包名
    final String packageName = getPackageName();
    // 到sSharedPrefsCache中找
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    // 如果找不到，就构建一个然后存进去
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    // 找得到就返回
    return packagePrefs;
}
```

## 2.2 构建SharedPreferencesImpl
### 2.2.1 SharedPreferencesImpl构造方法
我们先来看下这个类的构造方法：
```java
@UnsupportedAppUsage
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    // file的备份文件
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    // 从磁盘加载的标志，当需要从磁盘加载时将其设为true，这样如果有其他线程也调用了SharedPreferences的加载方法时，就会因为其为true而直接返回也就不执行加载方法
    // 保证了全局只有一个线程在加载
    mLoaded = false;
    // SharedPreferences中的数据
    mMap = null;
    // 保存的错误信息
    mThrowable = null;
    startLoadFromDisk();
}
```

初始化参数后立马调用了`startLoadFromDisk()`方法：
### 2.2.2 startLoadFromDisk()
```java
@UnsupportedAppUsage
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    // 开启一个新线程来加载数据
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}
```

### 2.2.3 loadFromDIsk()
`loadFromDisk()`:
```java
private void loadFromDisk() {
    synchronized (mLock) {
        // 如果已经家在过了，就直接退出
        if (mLoaded) {
            return;
        }
        // 如果备份文件已经存在，那就删除源文件，并将备份文件替换为源文件
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    // Debugging
    if (mFile.exists() && !mFile.canRead()) {
        Log.w(TAG, "Attempt to read preferences file " + mFile + " without permission");
    }

    // 存储聚的map
    Map<String, Object> map = null;
    // 文件信息，对应的是C语言stat.h中的struct stat
    StructStat stat = null;
    Throwable thrown = null;
    try {
        // 通过文件路径去构建StructStat对象
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            // 从XML中把数据读出来，并把数据转化成Map类型
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                map = (Map<String, Object>) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;

        // It's important that we always signal waiters, even if we'll make
        // them fail with an exception. The try-finally is pretty wide, but
        // better safe than sorry.
        try {
            if (thrown == null) {
                // 文件里拿到的数据为空就重建，存在就赋值
                if (map != null) {
                    // 将数据存储放置到具体类的一个全局变量中
                    // 稍微记一下这个关键点
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
            // In case of a thrown exception, we retain the old map. That allows
            // any open editors to commit and store updates.
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            mLock.notifyAll();
        }
    }
}
```
到目前来说，就完成的SharedPreferencesImpl的构建过程。

## 2.3 读数据 SharedPreferences # getXXX()
相对来说，读数据涉及到的方法比写数据简单得多，所以我们先来看下读数据：
我们以`getString()`为例
### 2.3.1 getString
```java
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        // 见2.3.2
        awaitLoadedLocked();
        // 从map中获取数据
        String v = (String)mMap.get(key);
        // 如果获取到数据，就返回数据，否则返回方法参数中给定的默认值
        return v != null ? v : defValue;
    }
}
```

### 2.3.2 awaitLoadedLocked
```java
@GuardedBy("mLock")
private void awaitLoadedLocked() {
    // 如果没有加载过，则进行加载
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    // 如果没有加载过，则等待
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```
这个方法简单点来说就是如果`mLoad`不为true也就是没有加载完成的话，就等待加载完成。


## 2.4 写数据
### 2.4.1 SharedPreferences.Editor
但是光构建了对象还不够，我们还得能对她进行操作。我们前面说到过，SharedPreferences并不提供修改的功能，如果你想对她进行修改，必须通过`SharedPreferences.Editor`来实现。

我们来看下`SharedPreferences.edit()`:
```java
@Override
public Editor edit() {
    // TODO: remove the need to call awaitLoadedLocked() when
    // requesting an editor.  will require some work on the
    // Editor, but then we should be able to do:
    //
    //      context.getSharedPreferences(..).edit().putString(..).apply()
    //
    // ... all without blocking.
    synchronized (mLock) {
        // ->>> 重点1
        awaitLoadedLocked();
    }

    // 创建了一个EditorImpl的对象，
    // 但是这块需要注意下，我们想对SharedPreferences进行修改，就必须调用edit()方法，就会去构建一个新的EditorImpl对象
    // 所以为了避免不必要的开销，我们在使用时最好一次性完成对数据的操作
    return new EditorImpl();
}

/**
 * 重点1：SharedPreferencesImpl # awaitLoadedLocked()
 */
@GuardedBy("mLock")
private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            // 如果还没有加载完成，就进入等待状态
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```

### 2.4.2 EditorImpl
#### 2.4.2.1 putXXX()
那我们再来看下`putXXX()`方法，我们以`putString()`来举例：
```java
public final class EditorImpl implements Editor {
    private final Object mEditorLock = new Object();

    // 存数据的HashMap
    @GuardedBy("mEditorLock")
    private final Map<String, Object> mModified = new HashMap<>();

    @GuardedBy("mEditorLock")
    private boolean mClear = false;

    @Override
    public Editor putString(String key, @Nullable String value) {
        synchronized (mEditorLock) {
            mModified.put(key, value);
            return this;
        }
    }

```

`putString()`方法很简单，直接将数据put到存数据的HashMap中去就行了。或者说，所有的`putXXX()`都是这么简单。

但是，如果我们想将修改提交到SharedPreferences里面去的话，还需要调用`apply()`或者`commit()`方法，那我们现在来看下这两个方法。

#### 2.4.2.2 apply()
```java
@Override
public void apply() {
    // 获取当前时间
    final long startTime = System.currentTimeMillis();

    // 见2.2.3.4
    // 构建了一个MemoryCommitResult的对象
    final MemoryCommitResult mcr = commitToMemory();
    // 新建一个线程，因为数据操作是很耗时的
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    // 进入等待状态
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }

                if (DEBUG && mcr.wasWritten) {
                    Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                            + " applied after " + (System.currentTimeMillis() - startTime)
                            + " ms");
                }
            }
        };

    // 将awaitCommit添加到Queue的Word中去
    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                // 执行操作，并从QueuedWord中删除
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
```

#### 2.4.2.3 commit()
```java
@Override
public boolean commit() {
    long startTime = 0;

    if (DEBUG) {
        startTime = System.currentTimeMillis();
    }

    // 见2.2.3.4
    // 构建了一个MemoryCommitResult对象
    MemoryCommitResult mcr = commitToMemory();
    // 将内存数据同步到文件
    // 见
    SharedPreferencesImpl.this.enqueueDiskWrite(
        mcr, null /* sync write on this thread okay */);
    try {
        // 进入等待状态, 直到写入文件的操作完成
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
        if (DEBUG) {
            Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                    + " committed after " + (System.currentTimeMillis() - startTime)
                    + " ms");
        }
    }
    // 通知监听则, 并在主线程回调onSharedPreferenceChanged()方法
    notifyListeners(mcr);
    // 返回文件操作的结果数据
    return mcr.writeToDiskResult;
}
```

#### 2.4.2.4 commitToMemory()
```java
// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
    // 当前Memory的状态，其实也就是当需要提交数据到内存的时候，他的值就加一
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    // 存数据的Map
    Map<String, Object> mapToWriteToDisk;

    synchronized (SharedPreferencesImpl.this.mLock) {
        // We optimistically don't make a deep copy until
        // a memory commit comes in when we're already
        // writing to disk.
        // 如果有数据待被提交到硬盘
        if (mDiskWritesInFlight > 0) {
            // We can't modify our mMap as a currently
            // in-flight write owns it.  Clone it before
            // modifying it.
            // noinspection unchecked
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        // 2.2.3.5的关键点
        mDiskWritesInFlight++;

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (mEditorLock) {
            boolean changesMade = false;
            
            // 如果mClear为true，就清空mapToWriteToDisk
            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                // "this" is the magic value for a removal mutation. In addition,
                // setting a value to "null" for a given key is specified to be
                // equivalent to calling remove on that key.
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }

                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }

            mModified.clear();

            if (changesMade) {
                mCurrentMemoryStateGeneration++;
            }

            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}
```
这段代码刚开始看的时候有点晕，但是看完后就瞬间懂了，这段代码主要执行了一下的功能：
- 将`mMap`赋值给`mapToWriteToDisk`
- 当`mClear`为true的时候，清空`mapToWriteToDisk`
- 遍历`mModified`，`mModified`也就是我们上面说到的保存本次edit的数据的HashMap
    - 当当前的`value`为null或者`this`的时候，移除对应的k
- 构建了一个`MemoryCommitResult`对象

#### 2.4.2.5 SharedPreferencesImpl # enqueueDiskWrite()
```java
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    // 这个方法这块就不讲了，太长了，大家感兴趣可以看下
                    // 主要功能就是
                    //  1. 当没有key没有改变，则直接返回了；否则执行下一步
                    //  2. 将mMap全部信息写入文件，如果写入成功则删除备份文件，如果写入失败则删除mFile
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) {
                    // 当写入成功后，将标志位减1
                    mDiskWritesInFlight--;
                }
                // 此时postWriteRunnable为null不执行该方法
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    // 如果是commit则进入
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (mLock) {
            // 由于commitToMemory会让mDiskWritesInFlight+1，则wasEmpty为true
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            // 在执行一遍上面的操作，保证将commit的内容也保存
            writeToDiskRunnable.run();
            return;
        }
    }
    // 如果是apply()方法，则会将任务放入单线程的线程池中去执行
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

所以从这个方法我们可以看到：
- `commit()`是直接同步执行的，有数据就存入磁盘
- `apply()`是先将`awaitCommit`放入`QueuedWork`，然后在单线程的线程池中去执行，执行完毕后再将`awaitCommit`从`QeueudWork`中移除。

# 3. 知识点
## 3.1 apply和commit的区别
- apply没有返回值, commit有返回值能知道修改是否提交成功
- apply是将修改提交到内存，再异步提交到磁盘文件; commit是同步的提交到磁盘文件;
- 多并发的提交commit时，需等待正在处理的commit数据更新到磁盘文件后才会继续往下执行，从而降低效率; 而apply只是原子更新到内存，后调用apply函数会直接覆盖前面内存数据，从一定程度上提高很多效率。

## 3.2 多进程的问题
我们前面说到了，SP提供了多进程访问，虽说没有像World模式那样会直接抛异常，但是官方不建议多进程下使用SP。

那么我们不禁会好奇，多进程下访问SP会有什么问题呢？

探究这个问题，我们得先回到`ContextImpl#getSharedPreferences(File file, int mode)`方法：
```java
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
    // ...前面的代码省略的，如果大家想回忆下，可以跳转到2.1节
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        // ->>> 重点1
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}

/**
 * 重点1 SharedPreferencesImpl # startReloadIfChangedUnexpectedly()
 */
void startReloadIfChangedUnexpectedly() {
    synchronized (mLock) {
        // TODO: wait for any pending writes to disk?
        // ->>> 重点2
        if (!hasFileChangedUnexpectedly()) {
            return;
        }
        // ->>> 重点3
        startLoadFromDisk();
    }
}

/**
 * 重点2 SharedPreferencesImpl # hasFileChangedUnexpectedly()
 *  如果文件发生了预期之外的修改，也就是说有其他进程在修改，就返回true，否则false
 */
private boolean hasFileChangedUnexpectedly() {
    synchronized (mLock) {
        // 如果mDiskWritesInFlight大于0，就证明是在当前进程中修改的，那就不用重新读取
        if (mDiskWritesInFlight > 0) {
            // If we know we caused it, it's not unexpected.
            if (DEBUG) Log.d(TAG, "disk write in flight, not unexpected.");
            return false;
        }
    }

    final StructStat stat;
    try {
        /*
         * Metadata operations don't usually count as a block guard
         * violation, but we explicitly want this one.
         */
        BlockGuard.getThreadPolicy().onReadFromDisk();
        stat = Os.stat(mFile.getPath());
    } catch (ErrnoException e) {
        return true;
    }

    synchronized (mLock) {
        return !stat.st_mtim.equals(mStatTimestamp) || mStatSize != stat.st_size;
    }
}

/**
 * 重点3 SharedPreferencesImpl # startLoadFromDisk()
 */
private void startLoadFromDisk() {
    synchronized (mLock) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            // ->>> 重点4，这块代码可以回到2.2.3看一下
            loadFromDisk();
        }
    }.start();
}
```
我们可以看到：每次获取SharedPreferences实例的时候尝试从磁盘中加载数据，并且是在异步线程中，因此一个线程的修改最终会反映到另一个线程，但不能立即反映到另一个进程，所以通过SharedPreferences无法实现多进程同步。

`loadFromDisk()`方法中我们最需要关注的是这一段：
```java
// 如果备份文件已经存在，那就删除源文件，并将备份文件替换为源文件
if (mBackupFile.exists()) {
    mFile.delete();
    mBackupFile.renameTo(mFile);
}
```

这块判断了`mBackupFile`是否存在，那`mBackupFile`我们是在哪创建的呢？
整个SharedPreferencesImpl中有两处：
- 构造方法：会调用`makeBackupFile()`给传入的`file`构造一个`mBackupFile`
- writeToFile()：在写入到磁盘的文件时，如果没有`mBackupFile`，就会根据当前的`mFile`重命名为`mBackupFile`

而`writeToFile()`在`enqueueDiskWrite()`中被调用，这个方法太长了，我截取下关键信息：
```java
@GuardedBy("mWritingToDiskLock")
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    // ...
    boolean fileExists = mFile.exists();
    // ...
    
    // Rename the current file so it may be used as a backup during the next read
    if (fileExists) {
        // ...
        boolean backupFileExists = mBackupFile.exists();
        // ...
        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                Log.e(TAG, "Couldn't rename file " + mFile
                      + " to backup file " + mBackupFile);
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {
            mFile.delete();
        }
    }

    // Attempt to write the file, delete the backup and return true as atomically as
    // possible.  If any exception occurs, delete the new file; next time we will restore
    // from the backup.
    try {
        FileOutputStream str = createFileOutputStream(mFile);
        // ...
        XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

        writeTime = System.currentTimeMillis();

        FileUtils.sync(str);

        fsyncTime = System.currentTimeMillis();

        str.close();
        ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

        // ...

        try {
            final StructStat stat = Os.stat(mFile.getPath());
            synchronized (mLock) {
                mStatTimestamp = stat.st_mtim;
                mStatSize = stat.st_size;
            }
        } catch (ErrnoException e) {
            // Do nothing
        }

        if (DEBUG) {
            fstatTime = System.currentTimeMillis();
        }

        // Writing was successful, delete the backup file if there is one.
        mBackupFile.delete();
        
        // ...
```
所以我们大致总结下这个方法的功能：
- 如果源文件`mFIle`存在并且备份文件`mBackupFile`不存在，就将源文件重命名为备份文件，如果源文件存在并且备份文件存在，就删除源文件
- 重新创建源文件`mFile`，并将内容写进去
- 删除`mBackupFile`

结合一下`loadFromDisk()`和`writeToFile()`两个方法，我们可以推测出：当存在两个进程，一个读进程，一个写进程，由于只有在创建`SharedPreferencesImpl`的时候创建了一个备份进程，此时读进程会将源文件删除，并将备份文件重命名为源文件，这样的结果就是，读进程永远只会看到写之前的内容。并且由于写文件需要调用`createFileOutputStream(mFile)`，但是这个时候由于源文件被读进程删除了，所以导致写进程的`mFIle`没有了引用，也就会创建失败，导致修改的数据无法更新到文件上，进而导致数据丢失。


## 3.3 建议优化
- 不要在SP中存储较大的key或者value
- 只是用MODE_PRIVATE模式，其它模式都不要使用(也被弃用了)
- 可以的话，尽量获取一次Editor然后提交所有的数据
- 不要高频使用apply，因为他每次都会新建一个线程；使用commit的时需谨慎，因为他在主线程中操作(对，就是主线程，主线程并不是只能更新UI，但是还是就把主线程当做更新UI的为好，我们的耗时操作最好不要在主线程中)
- 如果需要在多进程中存储数据，建议使用ContentProvider