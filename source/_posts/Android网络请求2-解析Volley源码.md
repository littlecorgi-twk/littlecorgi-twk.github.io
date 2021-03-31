---
title: Android网络请求2--解析Volley源码
categories:
  - Android
tags:
  - Android
  - kotlin
  - Volley
  - 源码
comments: true
abbrlink: df8fd75e
date: 2019-07-25 17:24:14
updated: 2019-07-25 17:24:14
copyright: true
---
>本文大篇幅参考[此篇文章](https://www.jianshu.com/p/d8500e377f3e)，大家可以结合两篇文章看一下

# 1. Volley简介
在很早以前，如果Android开发者想使用网络请求的话，必须自己通过`HttpClient`或者`HttpURLConnection`编写代码来访问。但是他两的用法还是很复杂的，如果不适当的封装的话，就会有很多多余代码甚至效率降低。所以当时出现了很多第三方网络通信框架，但是都是第三方的，而谷歌官方一直没有作为。
最终在2013年，谷歌终于意识到了问题，于是他们推出了一个官方的全新的网络框架——Volley。Volley它又能非常简单的进行HTTP通信，又能轻松加载网络上的图片。他的设计目的就是应对数据量不大但是频发的网络操作，但是对于下载等需要大数据量的网络操作，他就不太适合。
<!-- More -->

# 2. 源码解析
## 2.1 从RequestQueue入手
如果你使用Volley的话，就会发现Volley不管进行什么操作，首先第一步就是先创建RequestQueue对象。
所以我们就可以认定他为Volley的入口。
创建RequestQueue的方法是`RequestQueue mQueue = Volley.newRequestQueue(getApplicationContext());`,我们就看看`newRequestQueue`干了什么：
```java
public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
    BasicNetwork network;
    if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            network = new BasicNetwork(new HurlStack());
        } else {
            // 在Android 2.3之前，HttpURLConnection是不可靠的。
            // 请参阅：http://android-developers.blogspot.com/2011/09/androids-http-clients.html
            // 在将来的某个时候，我们将把minsdkversion移到Android 2.2之上，
            // 并可以删除这个回退（连同所有ApacheHTTP代码）。            
            String userAgent = "volley/0";
            try {
                String packageName = context.getPackageName();
                PackageInfo info =
             context.getPackageManager().getPackageInfo(packageName, /* flags= */ 0);
                    userAgent = packageName + "/" + info.versionCode;
            } catch (NameNotFoundException e) {
            }

        network =
                new BasicNetwork(
                        new HttpClientStack(AndroidHttpClient.newInstance(userAgent)));
        }
    } else {
        network = new BasicNetwork(stack);
    }

    return newRequestQueue(context, network);
}
```
调用方法后，先查看Android版本是否大于等于2.3，如果大于则调用基于`HttpURLConnection`的`HurlStack`，否则调用基于`HttpClient`的`HttpClientStack`。接下来创建`BasicNetwork`并调用`newRequestQueue(context, network)`方，我们再来看看这个`newRequestQueue()`方法：
```java
private static RequestQueue newRequestQueue(Context context, Network network) {
    final Context appContext = context.getApplicationContext();
    // 对缓存目录使用惰性供应商，以便可以在主线程上调用newRequestQueue（），
    // 而不会导致严格的模式冲突。    
    DiskBasedCache.FileSupplier cacheSupplier =
            new DiskBasedCache.FileSupplier() {
                private File cacheDir = null;

                @Override
                public File get() {
                    if (cacheDir == null) {
                        cacheDir = new File(appContext.getCacheDir(), DEFAULT_CACHE_DIR);
                    }
                    return cacheDir;
                }
            };
    RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheSupplier), network);
    queue.start();
    return queue;
}
```
可以看到，这个方法主要为Volley创建了一个硬盘缓存`DiskBasedCache`，然后通过这个磁盘缓存和`Network`创建了一个`RequestQueue`对象，并调用了`start()`方法，接下来我们看下`start()`方法:
```java
public void start() {
    stop(); 
    // 确保当前运行的所有调度程序都已停止。
    // 创建缓存调度器并开始它。
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
    mCacheDispatcher.start();

    // 创建达到池大小的网络调度程序（和相应的线程）。
    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher =
                    new NetworkDispatcher(mNetworkQueue, mNetwork, mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}
```
`CacheDispatcher`是一个缓存调度线程，并调用了`start()`方法。在循环中调用`NetworkDispatcher`的`start()`方法。`NetworkDispatcher`是网络调度线程，默认情况下`mDispatchers.length`为4，默认开启了4个调度线程，外加1个缓存调度线程，总共5个线程。
接下来Volley会创建各种`Request`，并调用`RequestQueue`的`add()`方法：
```java
public <T> Request<T> add(Request<T> request) {
    // 将请求标记为属于此队列，并将其添加到当前请求集。
    request.setRequestQueue(this);
    synchronized (mCurrentRequests) {
        //mCurrentRequests是一个HashSet
        mCurrentRequests.add(request);
    }

    // 按添加的顺序处理请求。
    request.setSequence(getSequenceNumber());
    request.addMarker("add-to-queue");
    sendRequestEvent(request, RequestEvent.REQUEST_QUEUED);

    // 如果请求是不可执行的，跳过缓存队列，然后直接进入网络。
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }
    mCacheQueue.add(request);
    return request;
}
```
这块地方的代码就很简单了，就是根据`request`的`shouldCache()`方法来返回`request`的`mShouldCache`属性来判断是否可以缓存，默认是可以的。如果能缓存，将此请求加入`mCacheQueue`队列，不再重复请求；不可以的话就将请求加入网络请求队列`mNetworkQueue`。

## 2.2 CacheDispatcher缓存调度线程
`RequestQueue`的`add()`方法并没有请求网络或者对缓存进行操作。当将请求添加到网络请求队 列或者缓存队列时，在后台的网络调度线程和缓存调度线程轮询各自的请求队列，若发现有请求任务则开 始执行。下面先看看缓存调度线程。

首先先来看看`CacheDispatcher`的`add()`方法：
```java
@Override
public void run() {
    if (DEBUG) 
        VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    // Process.THREAD_PRIORITY_BACKGROUND默认值为10

    // 进行阻塞调用以初始化缓存。
    mCache.initialize();

    while (true) {
        try {
            processRequest();
        } catch (InterruptedException e) {
            // 可能被打断了，因为是时候要退出了。
            if (mQuit) {
                Thread.currentThread().interrupt();
                    return;
            }
            VolleyLog.e(
                    "Ignoring spurious interrupt of CacheDispatcher thread; "
                            + "use quit() to terminate it");
            // 忽略cachedispatcher线程的假中断；
            // 使用quit（）终止它
        }
    }
}
```
这块可以看出主要就是初始化了缓存队列，然后开了个死循环，一直调用`processRequest()`，我们来看看这个方法：
```java
private void processRequest() throws InterruptedException {
    // 从CacheQueue中取出一个可用的request
    final Request<?> request = mCacheQueue.take();
    processRequest(request);
}

@VisibleForTesting
void processRequest(final Request<?> request) throws InterruptedException {
    request.addMarker("cache-queue-take");
    request.sendEvent(RequestQueue.RequestEvent.REQUEST_CACHE_LOOKUP_STARTED);

    try {
        //request如果被取消了，就直接返回
        if (request.isCanceled()) {
            request.finish("cache-discard-canceled");
            return;
        }

        Cache.Entry entry = mCache.get(request.getCacheKey());
        // 没有缓存就把request添加到NetworkQueue中
        if (entry == null) {
            request.addMarker("cache-miss");
            // 没有缓存，并且等待队列中也没有此request，那么就直接加入到NetworkQueue中
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                mNetworkQueue.put(request);
            }
            return;
        }

        // 如果缓存过期了，也是一样把request添加到NetworkQueue中
        if (entry.isExpired()) {
            request.addMarker("cache-hit-expired");
            request.setCacheEntry(entry);
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                mNetworkQueue.put(request);
            }
            return;
        }

        // 有缓存并且没有过期
        request.addMarker("cache-hit");
        // 根据缓存的内容解析
        Response<?> response =
                request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
        request.addMarker("cache-hit-parsed");
        // 是否需要更新
        if (!entry.refreshNeeded()) {
            // 不需要更新，直接将结果调度到主线程
            mDelivery.postResponse(request, response);
        } else {
            request.addMarker("cache-hit-refresh-needed");
            request.setCacheEntry(entry);
            response.intermediate = true;
            // 判断是否有相同缓存键的任务在执行
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                // 需要更新结果，先将结果调度到主线程，然后执行new runnable(){}
                // runnable中就是将request添加到NetworkQueue中，更新一下内容
                mDelivery.postResponse(
                        request,
                        response,
                        new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    mNetworkQueue.put(request);
                                } catch (InterruptedException e) {
                                    // Restore the interrupted status
                                    Thread.currentThread().interrupt();
                                }
                            }
                        });
            } else {
                // request已经加入到mWaitingRequests中
                // 直接把结果调度到主线程
                mDelivery.postResponse(request, response);
            }
        }
    } finally {
        request.sendEvent(RequestQueue.RequestEvent.REQUEST_CACHE_LOOKUP_FINISHED);
    }
}
```
我们在`processRequest`中可以看到有一个方法经常出现，那就是`mWaitingRequestManager.maybeAddToWaitingRequests(request)`，它的作用是判断当前这个`request`是否有存在相同缓存键的请求已经处于运行状态，如果有，那么就将这个`request`加入到一个等待队列中，等到相同缓存键的请求完成。

总结一下`CacheDispatcher`主要步骤：

- 从`CacheQueue`中循环取出`request`；
- 如果缓存丢失，加入到`NetworkQueue`中；
- 如果缓存过期，加入到`NetworkQueue`中；
- 将缓存中的数据解析成`Response`对象；
- 如果不需要更新，直接将结果回调到主线程，回调操作等介绍完`NetworkDispatcher`之后一起深入剖析；
- 如果需要更新，先将结果回调到主线程，然后再将`request`加入到`NetworkQueue`中。

下面来看看网络调度线程。

## 2.3 NetWorkDispatcher网络调度线程
NetworkDispatcher的run方法代码如下所示:
```java
@Override
public void run() {
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    while (true) {
        try {
            processRequest();
        } catch (InterruptedException e) {
            // 我们可能被打断了，因为是时候退出了。
            if (mQuit) {
                Thread.currentThread().interrupt();
                return;
            }
            VolleyLog.e(
                    "Ignoring spurious interrupt of NetworkDispatcher thread; "
                            + "use quit() to terminate it");
        }
    }
}
```
由此可以看出，`NetWordDispatch`和`CacheDispatch`非常类似。
他的`run()`方法和`CacheDispatch`的方法基本一样，这就不多做介绍，下面来看看他的`processRequest()`方法。
```java
private void processRequest() throws InterruptedException {
    // 从NetworkQueue中取出request
    Request<?> request = mQueue.take();
    processRequest(request);
}

@VisibleForTesting
void processRequest(Request<?> request) {
    long startTimeMs = SystemClock.elapsedRealtime();
    request.sendEvent(RequestQueue.RequestEvent.REQUEST_NETWORK_DISPATCH_STARTED);
    try {
        request.addMarker("network-queue-take");

        // 如果request被取消了，那么就不执行此request
        if (request.isCanceled()) {
            request.finish("network-discard-cancelled");
            request.notifyListenerResponseNotUsable();
            return;
        }

        addTrafficStatsTag(request);

        // 还记得这个mNetwork么，它就是Volley.newRequestQueue()方法里的BasicNetwork对象，一会我们来看看mNetwork.performRequest()方法是如何得到NetworkResponse的
        NetworkResponse networkResponse = mNetwork.performRequest(request);
        request.addMarker("network-http-complete");

        // notModified是服务端返回304，hasHadResponseDelivered()是request已经回调过了
        if (networkResponse.notModified && request.hasHadResponseDelivered()) {
            request.finish("not-modified");
            request.notifyListenerResponseNotUsable();
            return;
        }

        // 将NetworkResponse解析成Response对象，在子线程中执行
        Response<?> response = request.parseNetworkResponse(networkResponse);
        request.addMarker("network-parse-complete");

        // 将request写入缓存
        if (request.shouldCache() && response.cacheEntry != null) {
            mCache.put(request.getCacheKey(), response.cacheEntry);
            request.addMarker("network-cache-written");
        }

        request.markDelivered();
        // 回调结果至主线程
        mDelivery.postResponse(request, response);
        request.notifyListenerResponseReceived(response);
    } 
    // 以下都是处理异常错误，然后也需要回调至主线程
    catch (VolleyError volleyError) {
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        parseAndDeliverNetworkError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    } catch (Exception e) {
        VolleyLog.e(e, "Unhandled exception %s", e.toString());
        VolleyError volleyError = new VolleyError(e);
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        mDelivery.postError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    } finally {
        request.sendEvent(RequestQueue.RequestEvent.REQUEST_NETWORK_DISPATCH_FINISHED);
    }
}
```
通过`NetworkDispatcher.processRequest()`方法可以发现，主要分为以下几步：

- 通过`BasicNetwork.performRequest(request)`得到`NetworkResponse`对象；
- 通过`request.parseNetworkResponse(networkResponse)`解析得到`Response`对象；
- 通过`mDelivery`将成功结果或者失败结果回调到主线程。

现在我们依次来分析这三步：
- 请求网络，得到`NetworkResponse`
    我们看看`BasicNetwork`的`performRequest()`方法：
    
```java
@Override
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
    long requestStart = SystemClock.elapsedRealtime();
    while (true) {
        HttpResponse httpResponse = null;
        byte[] responseContents = null;
        List<Header> responseHeaders = Collections.emptyList();
        try {
            // Gather headers.
            Map<String, String> additionalRequestHeaders =
                    getCacheHeaders(request.getCacheEntry());
            httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders);
            int statusCode = httpResponse.getStatusCode();

            responseHeaders = httpResponse.getHeaders();
            // Handle cache validation.
            if (statusCode == HttpURLConnection.HTTP_NOT_MODIFIED) {
                Entry entry = request.getCacheEntry();
                if (entry == null) {
                    return new NetworkResponse(
                            HttpURLConnection.HTTP_NOT_MODIFIED,
                            /* data= */ null,
                            /* notModified= */ true,
                            SystemClock.elapsedRealtime() - requestStart,
                            responseHeaders);
            }
                    …………省略
```
通过上面源码可以看出，`BasicNetwork`就是封装了一下`NetworkResponse`，并没有涉及到网络请求，我们继续深入到`BaseHttpStack.executeRequest(request, additionalRequestHeaders)`源码中。
```java
public abstract HttpResponse executeRequest(
        Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError;
```
这时发现这个`BaseHttpStack`就是一个抽象类，而这个`executeRequest()`也就是一个抽象方法。我当时就卡在这了，调用了一个抽象类的抽象方法，这咋操作嘛。然后我就好好再看了一遍，找到`BasicNetwork`的构造函数中对`mBaseHttpStck`定义的地方，发现这个是构造函数传进来的，然后就想到了在调用`Volley.newRequestQueue()`时，是根据Android版本传入了不同的`Stack`，那我们就来看看`HurlStack.executeRequest()`方法：
```java
@Override
public HttpResponse executeRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError {
    String url = request.getUrl();
    HashMap<String, String> map = new HashMap<>();
    map.putAll(additionalHeaders);
    // request.getheaders（）优先于给定的附加（缓存）头.
    map.putAll(request.getHeaders());
    if (mUrlRewriter != null) {
        String rewritten = mUrlRewriter.rewriteUrl(url);
        if (rewritten == null) {
            throw new IOException("URL blocked by rewriter: " + url);
        }
        url = rewritten;
    }
    URL parsedUrl = new URL(url);
    HttpURLConnection connection = openConnection(parsedUrl, request);
    boolean keepConnectionOpen = false;
    try {
        for (String headerName : map.keySet()) {
            connection.setRequestProperty(headerName, map.get(headerName));
        }
        setConnectionParametersForRequest(connection, request);
        // 使用来自httpurlConnection的数据初始化httpResponse。
        int responseCode = connection.getResponseCode();
        if (responseCode == -1) {
            // 如果无法检索响应代码，getResponseCode（）将返回-1。
            // 向呼叫者发出信号，说明连接有问题。
            throw new IOException("Could not retrieve response code from HttpUrlConnection.");
        }

        if (!hasResponseBody(request.getMethod(), responseCode)) {
            return new HttpResponse(responseCode, convertHeaders(connection.getHeaderFields()));
        }

        // 需要保持连接打开，直到调用方使用流。包装流，以便close（）将断开连接。
        keepConnectionOpen = true;
        return new HttpResponse(
                responseCode,
                convertHeaders(connection.getHeaderFields()),
                connection.getContentLength(),
                new UrlConnectionInputStream(connection));
    } finally {
        if (!keepConnectionOpen) {
            connection.disconnect();
        }
    }
}
```
可以看到，主要就是借助了`HttpURLConnection`对象来请求网络，并根据不同的条件返回不同的`HttpResponse`对象。
- 解析`NetworkResponse`, 得到`Response`：
解析过程是定义在`Request`类中，但是他是一个抽象类，不同的Request都有自己的实现，我们现在就以JsonRequest为例看看：
然后发现他又是一个抽象类，那我们就看看`JsonRequest`其中一个实现类`JsonObjectRequest`的`parseNetworkResponse()`方法：

```java
@Override
protected Response<JSONObject> parseNetworkResponse(NetworkResponse response) {
    try {
        String jsonString =
                new String(
                        response.data,
                        HttpHeaderParser.parseCharset(response.headers, PROTOCOL_CHARSET));
        return Response.success(
                new JSONObject(jsonString), HttpHeaderParser.parseCacheHeaders(response));
    } catch (UnsupportedEncodingException e) {
        return Response.error(new ParseError(e));
    } catch (JSONException je) {
        return Response.error(new ParseError(je));
    }
}
```
这个就不用多说了，根据返回来的`response`建了一个`String`然后把这个`String`放到`Response`里面去然后再返回去。
- 回调主线程
回调主要是通过`Delivery`的`postResponse()`方法实现的，我们来看看这个方法,找过去又找到了一个`ResponseDelivery`抽象类，然后又得找他的实现类，这时大家应该记得`RequestQueue()`的时候初始化了一个`Delivery`：

```java
public RequestQueue(Cache cache, Network network, int threadPoolSize) {
    this(
            cache,
            network,
            threadPoolSize,
            new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}
```
他返回了一个`ExecutorDelivery`，我们来看看这个类，然后就惊喜的发现，我们终于找到我们需要的东西了：
```java
/**
 * Creates a new response delivery interface.
 *
 * @param handler {@link Handler} to post responses on
 */
public ExecutorDelivery(final Handler handler) {
    // Make an Executor that just wraps the handler.
    mResponsePoster =
            new Executor() {
                @Override
                public void execute(Runnable command) {
                        handler.post(command);
                }
            };
}
```
知道了它的初始化，我们再来看看它是如何实现回调的：

`Volley`中回调是通过`postResponse()`方法的 :

```java
public void postResponse(Request<?> request, Response<?> response) {
    postResponse(request, response, null);
}

@Override
public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
    request.markDelivered();
    request.addMarker("post-response");
    mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
}
```

`postResponse()`最终会调用`mResponsePoster`对象的`execute()`方法，传入了一个`ResponseDeliveryRunnable`对象，它实现了`Runnable`接口，`execute()`方法会通过`Handler.post(runnable)`将`ResponseDeliveryRunnable`放入消息队列。最后我们来看看这个`ResponseDeliveryRunnable`的`run()`方法在主线程中做了什么操作：

```java
@SuppressWarnings("unchecked")
@Override
public void run() {

    // If this request has canceled, finish it and don't deliver.
    if (mRequest.isCanceled()) {
        mRequest.finish("canceled-at-delivery");
        return;
    }

    if (mResponse.isSuccess()) {
        // 执行成功的回调，在具体的Request实现类中，比如StringRequest就会调用listener.onResponse(string)回调
        mRequest.deliverResponse(mResponse.result);
    } else {
        // 执行失败的回调，在request中，直接回调了listener.onErrorResponse(error)
        mRequest.deliverError(mResponse.error);
    }

    // intermediate默认为false，但是在CacheDispatcher的run()中，如果需要更新缓存，那么就会置为true
    if (mResponse.intermediate) {
        mRequest.addMarker("intermediate-response");
    } else {
        mRequest.finish("done");
    }

    // 如果传入了runnable不为空，那就就执行runnable.run()方法
    // 回忆下在CacheDispatcher的run()方法中，如果request有缓存，但是需要更新缓存的时候，mDelivery是不是调用的带runnable的方法
    if (mRunnable != null) {
        mRunnable.run();
    }
}
```

# 3 请求流程图
最后附上Volley的请求流程图
![VolleyRequestProcess](https://cdn.littlecorgi.top/mweb/2019-07-23/VolleyRequestProcess.jpg)



