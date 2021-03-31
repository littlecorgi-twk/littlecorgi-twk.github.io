---
title: Android网络请求4--解析Retrofit源码
categories:
  - Android
tags:
  - Android
  - kotlin
  - OkHttp
  - Retrofit
  - 源码
comments: true
copyright: true
abbrlink: a2a8de56
date: 2019-08-06 14:34:05
updated: 2019-08-06 14:34:05
---
# 1. Retrofit简介
- [Retrofit - github](https://github.com/square/retrofit)
- [Retrofit - Doc](https://square.github.io/retrofit/)

`Retrofit`是Square公司的又一力作，针对Android网络请求的框架，遵循`Restful`设计风格，底层基于`OkHttp`。
<!-- More -->

他对比其他框架
- 性能最好
- 封装程度高，拓展性差
- 简介易用，代码简单
- 解耦彻底
- 可以非常方便的与RxJava连用

# 2. Retrofit用法（异步）
## 2.1 添加依赖
可以在Retrofit Github库页面里面找到最新版本号，我写这篇博客时最新版导入方式

在你项目的app的`build.gradle`里面添加
`implementation 'com.squareup.retrofit2:retrofit:2.6.1'`

同时，如果你需要配套的数据转换器还需要导入以下的依赖
- Gson: `com.squareup.retrofit2:converter-gson`
- Jackson: `com.squareup.retrofit2:converter-jackson`
- Moshi: `com.squareup.retrofit2:converter-moshi`
- Protobuf: `com.squareup.retrofit2:converter-protobuf`
- Wire: `com.squareup.retrofit2:converter-wire`
- Simple XML: `com.squareup.retrofit2:converter-simplexml`
- Scalars (primitives, boxed, and String): `com.squareup.retrofit2:converter-scalars`

## 2.2 添加网络权限
在你APP的AndroidManifest.xml里添加
`<uses-permission android:name="android.permission.INTERNET"/>
`

## 2.3 创建 接收服务器返回数据 的类
*Reception.java*
```java
puublic class Reception {
    // 根据返回数据的格式和数据解析方式定义
    ...
}
```

## 2.4 创建 用于描述网络请求 的接口
*GetRequest_Interface.interface*
```java
public interface GetRequest_Interface {
    @GET("openapi.do?keyfrom=Yanzhikai&key=2032414398&type=data&doctype=json&version=1.1&q=car")
    Call<Translation>  getCall();
    // @GET注解的作用:采用Get方法发送网络请求
 
    // getCall() = 接收网络请求数据的方法
    // 其中返回类型为Call<*>，*是接收数据的类（即上面定义的Translation类）
    // 如果想直接获得Responsebody中的内容，可以定义网络请求返回值为Call<ResponseBody>
}
```

## 2.5 创建Retrofit实例
```java
 Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://fanyi.youdao.com/") // 设置网络请求的Url地址
                .addConverterFactory(GsonConverterFactory.create()) // 设置数据解析器
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create()) // 支持RxJava平台
                .build();
```

## 2.6 创建网络请求接口实例
```java
// 创建 网络请求接口 的实例
GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

//对 发送请求 进行封装
Call<Reception> call = request.getCall();
```

## 2.7 发送网络请求
```java
//发送网络请求(异步)
call.enqueue(new Callback<Translation>() {
    //请求成功时回调
    @Override
    public void onResponse(Call<Translation> call, Response<Translation> response) {
        //请求处理,输出结果
        response.body().show();
    }

    //请求失败时候的回调
    @Override
    public void onFailure(Call<Translation> call, Throwable throwable) {
        System.out.println("连接失败");
    }
});
```

## 2.8 处理返回数据
```java
//发送网络请求(异步)
call.enqueue(new Callback<Translation>() {
    //请求成功时回调
    @Override
    public void onResponse(Call<Translation> call, Response<Translation> response) {
        // 对返回数据进行处理
        response.body().show();
    }

    //请求失败时候的回调
    @Override
    public void onFailure(Call<Translation> call, Throwable throwable) {
        System.out.println("连接失败");
    }
});
```

由于本文的核心不是讲用法，所以关于用法这块，我并没有多讲，大家若想多了解，可以看此博客：[这是一份很详细的 Retrofit 2.0 使用教程（含实例讲解）](https://blog.csdn.net/carson_ho/article/details/73732076)

# 3. Retrofit源码
## 3.1 Retrofit对象构造源码
应对一个框架的源码首先从使用它的地方开始，我们先来看`Retrofit`的创建代码：
```java
 Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("http://fanyi.youdao.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```

我们可以把它分为5个部分来分析源码：
![源码分析步骤](https://user-gold-cdn.xitu.io/2018/3/7/161fdeda6c0120c1?imageslim)

我们首先来看第一步

### 3.1.1 步骤1：Retrofit类
```java
public final class Retrofit {
    private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();
  
    // 网络请求工厂
    final okhttp3.Call.Factory callFactory;
    // 请求的Url地址
    final HttpUrl baseUrl;
    // 数据转换器工厂的集合
    final List<Converter.Factory> converterFactories;
    // 网络请求适配器工厂的集合
    final List<CallAdapter.Factory> callAdapterFactories;
    // 回调
    final @Nullable Executor callbackExecutor;
    // 是否提前对业务接口中的注解进行验证转换的标志位
    final boolean validateEagerly;

    Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
        List<Converter.Factory> converterFactories, List<CallAdapter.Factory> callAdapterFactories,
        @Nullable Executor callbackExecutor, boolean validateEagerly) {
        this.callFactory = callFactory;
        this.baseUrl = baseUrl;
        this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
        this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
        this.callbackExecutor = callbackExecutor;
        this.validateEagerly = validateEagerly;
    }
    ...// 省略后面的代码
}
```
可以看到，`Retrofit`的构造方法需要一次性把需要的数据全部准备好。
而且这块有特别多的工厂，使用了工厂模式。这个我们之后再讲，接下来看步骤2：

### 3.1.2 步骤2：Builder()方法
我们先来看下`Builder()`方法的源码：
```java
public Builder() {
    this(Platform.get());
}
```

这里调用了`Rlatform.get()`方法：
```java
static Platform get() {
    return PLATFORM;
}
```

`PLATFORM`的定义就在上面：
```java
private static final Platform PLATFORM = findPlatform();
```

我们再来看下`findPlatform()`方法：
```java
private static Platform findPlatform() {
    try {
        Class.forName("android.os.Build");
        if (Build.VERSION.SDK_INT != 0) {
            return new Android();
        }
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform(true);
}
```

这块创建了一个`Android`对象，接着返回了传入`true`创建的`Platform`对象。我们再来看下`Android`类：
```java
static final class Android extends Platform {
    Android() {
        super(Build.VERSION.SDK_INT >= 24);
    }

    @Override public Executor defaultCallbackExecutor() {
        // 返回一个默认的回调方法执行器
        // 该执行器作用：切换线程（子->>主线程），并在主线程（UI线程）中执行回调方法
        return new MainThreadExecutor();
    }

    static class MainThreadExecutor implements Executor {
        // 获取与Android 主线程绑定的Handler 
        private final Handler handler = new Handler(Looper.getMainLooper());

        @Override public void execute(Runnable r) {
            // 该Handler是上面获取的与Android 主线程绑定的Handler 
            // 在UI线程进行对网络请求返回数据处理等操作。
            handler.post(r);
        }
    }
}
```

### 3.1.3 步骤3：baseUrl()方法
```java
public Builder baseUrl(String baseUrl) {
    Objects.requireNonNull(baseUrl, "baseUrl == null");
    return baseUrl(HttpUrl.get(baseUrl));
}
```

最后还是调用了一个`baseUrl()`方法，我们来看看这个`baseUrl()`方法：
```java
public Builder baseUrl(HttpUrl baseUrl) {
    Objects.requireNonNull(baseUrl, "baseUrl == null");
    // 把URL参数分割成几个路径碎片
    List<String> pathSegments = baseUrl.pathSegments();
    // 检测最后一个碎片来检查URL参数是不是以"/"结尾,不是就抛出异常	
    if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
    }
    this.baseUrl = baseUrl;
    return this;
}
```
步骤3用于设置网络请求的`Url`

### 3.1.4 步骤4：addConverterFactory()方法
我们先来看看`GsonConverterFactory.create()`方法：
```java
public static GsonConverterFactory create() {
    return create(new Gson());
}
```

又调用了另一个`create()`方法：
```java
public static GsonConverterFactory create(Gson gson) {
    if (gson == null) throw new NullPointerException("gson == null");
    return new GsonConverterFactory(gson);
}
```

如果传入的`gson`为空就报异常，不为空就调用`GsonConverterFactory`的构造方法：
```java
private GsonConverterFactory(Gson gson) {
    this.gson = gson;
}
```

所以这个方法本质就是返回了一个`gson`对象给了`addConverterFactory()`方法，那我们再来看看这个方法：
```java
/** 为对象的序列化和反序列化添加转换器工厂。 */
public Builder addConverterFactory(Converter.Factory factory) {
    converterFactories.add(Objects.requireNonNull(factory, "factory == null"));
    return this;
}
```
步骤4创建了一个含有`Gson`实例的`GsonConverterFactory`对象，并放入了`ConverterFactory`。

### 3.1.5 步骤5：build()方法
```java
public Retrofit build() {
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }
    // 配置网络请求执行器
    okhttp3.Call.Factory callFactory = this.callFactory;
    // 如果没有指定callFactory，则创建OkHttpClient
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }
    // 配置回调执行器
    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // 配置网络请求适配器工厂
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

    // 配置数据转换器工厂
    List<Converter.Factory> converterFactories = new ArrayList<>(
        1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

    // 首先添加内置转换器工厂。这可以防止重写它的行为，但也可以确保在使用使用使用所有类型的转换器时正确的行为。
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);
    converterFactories.addAll(platform.defaultConverterFactories());

    return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
        unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```
按前面配置的变量，将`Retrofit`所有的变量都配置好，最后完成创建`Retrofit`实例。

到这Retrofit的创建部分就完了。
Retrofit通过建造者模式创建了一个Retrofit实例：
- 请求工厂callFactory：默认是OkHttpClient
- 数据转换器工厂converterFactories
- 网络请求适配器工厂callAdapterFactories：默认是ExecutorCallAdapterFactory
- 回调执行器callbackExecutor

![Retrofit对象构建流程](https://cdn.littlecorgi.top/mweb/Retrofit对象构建流程.jpg)


## 3.2 创建网络请求接口的实例
大致创建方法如下：
```java
<!-- Reception.java -->
puublic class Reception {
    // 根据返回数据的格式和数据解析方式定义
    ...
}

<!-- GetRequest_Interface.interface -->
public interface GetRequest_Interface {
    @GET("openapi.do?keyfrom=Yanzhikai&key=2032414398&type=data&doctype=json&version=1.1&q=car")
    Call<Translation>  getCall();
    // @GET注解的作用:采用Get方法发送网络请求
 
    // getCall() = 接收网络请求数据的方法
    // 其中返回类型为Call<*>，*是接收数据的类（即上面定义的Translation类）
    // 如果想直接获得Responsebody中的内容，可以定义网络请求返回值为Call<ResponseBody>
}

<!-- MainActivity.java -->
// 创建 网络请求接口 的实例
GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);
//对 发送请求 进行封装
Call<Reception> call = request.getCall();
```

我们先看看`retrofit.create()`干了啥：
```java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override public @Nullable Object invoke(Object proxy, Method method,
                    @Nullable Object[] args) throws Throwable {
                // 如果该方法是来自对象的方法，则推迟到正常调用。
                if (method.getDeclaringClass() == Object.class) {
                    return method.invoke(this, args);
                }
                if (platform.isDefaultMethod(method)) {
                    return platform.invokeDefaultMethod(method, service, proxy, args);
                }
                return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
            }
        });
}
```
首先调用了`validateServiceInterface()`这个方法，我们来看看这个方法：
```java
private void validateServiceInterface(Class<?> service) {
    // 如果不是接口就抛异常
    if (!service.isInterface()) {
        throw new IllegalArgumentException("API declarations must be interfaces.");
    }

    // 构造了一个容量下限为1的空双向队列来存储数据
    Deque<Class<?>> check = new ArrayDeque<>(1);
    // 添加到队列
    check.add(service);
    while (!check.isEmpty()) {
        // 将队列的正向第一个元素移除
        Class<?> candidate = check.removeFirst();
        // getTypeParameters()的作用是得到泛型类型，如果泛型的数量不为0的话进入
        if (candidate.getTypeParameters().length != 0) {
            StringBuilder message = new StringBuilder("Type parameters are unsupported on ")
                .append(candidate.getName());
            if (candidate != service) {
                message.append(" which is an interface of ")
                    .append(service.getName());
            }
            throw new IllegalArgumentException(message.toString());
        }
        Collections.addAll(check, candidate.getInterfaces());
    }

    if (validateEagerly) {
        Platform platform = Platform.get();
        for (Method method : service.getDeclaredMethods()) {
            if (!platform.isDefaultMethod(method) && !Modifier.isStatic(method.getModifiers())) {
                loadServiceMethod(method);
            }
        }
    }
}
```
获取到`Platform`（平台），然后再遍历接口中的方法，调用`loadServiceMethod()`方法：
```java
ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = ServiceMethod.parseAnnotations(this, method);
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```
这个方法主要就是通过`serviceMethodCache`来存储转换为`ServiceMethod`类型的`Method`。

那我们在回过去看`InvocationHandler`的`invoke()`方法。
```java
new InvocationHandler() {
    private final Platform platform = Platform.get();
    private final Object[] emptyArgs = new Object[0];

    @Override public @Nullable Object invoke(Object proxy, Method method,
            @Nullable Object[] args) throws Throwable {
        // 如果该方法是来自对象的方法，则推迟到正常调用。
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
        }
        if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
    }
});
```
两个if先不管，先看最后的`return`，可以看到他调用了一个`loadServiceMethod()`方法，而这个方法在前面就说过，主要就返回了一个`ServiceMethod`。那我们接着来看`invoke()`方法：
按住`command+鼠标左键`进入后，显示的是：
```java
abstract class ServiceMethod<T> {
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }

  abstract @Nullable T invoke(Object[] args);
}
```
这个`invoke()`根本就是个抽象方法啊，然后我又回去找看看有没有说是哪个继承自`ServiceMethod`的类实现了这个方法，结果也没找到，最后想到了AndroidStudio可以`command+左键`找实现了这个类的地方啊。于是我就点了下这个类，结果成功找到了`HttpServiceMethod`这个类，他是`ServiceMethod`的子类中唯一一个实现了`invoke()`方法的：
```java
@Override final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
}
```

可以看到这块他创建了一个OkHttpCall对象，此处不做详解，可以看我关于OkHttp的一篇博客：[Android网络请求3--解析OkHttp源码](https://www.littlecorgi.top/posts/151ac78a.html)。然后调用了adapt方法，我们来看看这个方法:
```java
protected abstract @Nullable ReturnT adapt(Call<ResponseT> call, Object[] args);
```
这又是一个抽象方法。只不过好的是，这个方法下面我们就找到了他的实现：
```java
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
    private final CallAdapter<ResponseT, ReturnT> callAdapter;

    CallAdapted(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
            Converter<ResponseBody, ResponseT> responseConverter,
            CallAdapter<ResponseT, ReturnT> callAdapter) {
        super(requestFactory, callFactory, responseConverter);
        this.callAdapter = callAdapter;
    }

    @Override protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
        return callAdapter.adapt(call);
    }
}
```
可是他又返回给了一个`CallAdapter`接口的实现类的`adapt()`方法。。。。艹

所以，这块我就不懂了，我不知道他到底咋实现这个的这个方法的，我看了《Android进阶之光》和其它很多博客，他们都是老版本的`Retrofit`，`create()`方法不同，而且调用的`adapt()`方法也不同。。。

![Retrofit网络请求接口创建](https://¡cdn.littlecorgi.top/mweb/Retrofit网络请求接口创建.jpg)


## 3.3 网络请求
网络请求这块Retrofit实际上就是用OkHttp实现的，所以这块我就不在这多说了，大家要了解的可以看我之前写的博客：[Android网络请求3--解析OkHttp源码](https://www.littlecorgi.top/posts/151ac78a.html)。

