---
title: Android网络请求3--解析OkHttp源码
categories:
  - Android
tags:
  - Android
  - kotlin
  - OkHttp
  - 源码
comments: true
abbrlink: 151ac78a
date: 2019-07-26 17:11:17
updated: 2019-07-26 17:11:17
copyright: true
---
# 1. OkHttp简介
`okhttp`是一个第三方类库，用于android中请求网络。

这是一个开源项目,是安卓端最火热的轻量级框架,由移动支付Square公司贡献。用于替代因移除了`HttpClient`而导致没用的`Volley`。

目前更多人选择了`Retrofit`。
<!-- More -->

# 2. 源码解析
>本文对OkHttp的探讨全部基于目前的最新版OkHttp:4.0.1，而这个版本作者已经使用kotlin对源码进行了重写，所以有些小伙伴可能阅读稍微有点问题，但是别担心，本文中所涉及的源码阅读起来基本上和Java一样，所以请不会kotlin的小伙伴还是耐心看下去，不太懂的语法就百度下，同时我也会对某些语法作注释

## 2.1 OkHttp请求流程
### 2.1.1 从请求处理开始分析
我们无论在使用`OkHttp`进行什么请求的时候都会创建`OkHttpClient`对象并调用他的`newCall()`方法，那我们就从这个方法看起：

```kotlin
override fun newCall(request: Request): Call {
return RealCall.newRealCall(this, request, forWebSocket = false)
}
```
可以看到返回了一个`RealCall`对象，所以也就意味着我们使用`OkHttpClient`对象调用的`execute()`操作实际上是`RealCall`的`execute()`操作，那我们就来看`RealCall`的`execute()`方法：
```kotlin
override fun execute(): Response {
    // 添加同步锁
    synchronized(this) {
        // check()是kotlin特有的一个方法，他本质上就是一个if，
        // 但是当他的判断语句是false的话，
        // 他就会抛出一个IllegalStateException异常，异常的内容就是后面的语句
        check(!executed) { "Already Executed" }
        executed = true
        // executed是一个布尔值，他的作用就是判断是不是执行过了，
        // 如果执行过了还执行了这个方法的话就抛异常
    }
    // transmitter用于连接OKHTTP的应用程序和网络层，不用多管
    transmitter.timeoutEnter()
    transmitter.callStart()
    try {
        client.dispatcher.executed(this)
        return getResponseWithInterceptorChain()
    } finally {
        client.dispatcher.finished(this)
    }
}
```
这块又调用了`client.dispatcher`，然后找回去找到`OkHttpClient`的`dispatcher`对象，发现他就是`Dispatcher`类的一个对象，接着我们继续看`Dispatcher`类。

### 2.1.2 Dispatcher任务调度
进入`Dispatcher`类，我们可以看到如下成员变量定义:
>注：kotlin中一个成员变量的@get和@set分别对应了Java中get和set方法，所以这块我没有完完全全复制粘贴到这，我只取了定义部分

```kotlin
// 最大并发请求数
var maxRequests = 64
// 每个主机的最大请求数
var maxRequestsPerHost = 5
// 消费者线程
val executorService: ExecutorService
// 将要运行的异步请求队列
private val readyAsyncCalls = ArrayDeque<AsyncCall>()
// 正在运行的异步请求队队列
private val runningAsyncCalls = ArrayDeque<AsyncCall>()
// 正在运行的同步请求队列
private val runningSyncCalls = ArrayDeque<RealCall>()
```

接下来我们看看`Dispatcher`的构造方法
```kotlin
// 主构造方法，没写具体实现
class Dispatcher constructor() {} 

constructor(executorService: ExecutorService) : this() {
    this.executorServiceOrNull = executorService
}
```
我们可以看到他将传进来的`executorService`传给了`executorServiceOrNull`，那我们来看看`executorServiceOrNull`的定义：
```kotlin
// executorServiceOrNull这个应该是因为kotlin的空安全检查特性而定义的，本质上就是executorService
private var executorServiceOrNull: ExecutorService? = null

@get:Synchronized
@get:JvmName("executorService") val executorService: ExecutorService
    get() {
        if (executorServiceOrNull == null) {
            executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
                SynchronousQueue(), threadFactory("OkHttp Dispatcher", false))
        }
        return executorServiceOrNull!!
    }
```
我们可以看到`executorService`的`set`方法，就是创建了一个线程池。再结合他有两个构造器就知道：如果没有给`Dispatcher`传入一个线程池他就会自己创建一个线程池。这个线程池适合执行大量且耗时较少的任务。

构造器我们看完了，我们就来看他的`enqueue()`方法：
```kotlin
internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
        readyAsyncCalls.add(call)

        // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
        // the same host.
        if (!call.get().forWebSocket) {
            val existingCall = findExistingCallWithHost(call.host())
            if (existingCall != null)call.reuseCallsPerHostFrom(existingCall)
        }
    }
    promoteAndExecute()
}
```
来一个请求就把他添加到就绪请求队列中去，然后就来判断`forWebSocket`这个属性。看到这个属性我还迷了下，有点搞不懂她是干嘛的，然后经过我的一番搜索后，发现原来`OkHttp`还可以进行`WebSocket`通信，而这个属性就是为`WebSocket`通信准备的。于是我就到`RealCall`里面找在哪儿定义了这个属性了，然后我就发现了在`RealCall`的`newRealCall()`方法这块，这个方法传入的参数中有一个`Boolean`值名字就叫`forWebSocket`。
```kotlin
companion object {
    fun newRealCall(
        client: OkHttpClient,
        originalRequest: Request,
        forWebSocket: Boolean
    ): RealCall {
        // Safely publish the Call instance to the EventListener.
        return RealCall(client, originalRequest, forWebSocket).apply {
            transmitter = Transmitter(client, this)
        }
    }
}
```
不知道大家有没有印象，咱们在上面说过，执行`OkHttpClient.newCall()`方法实际上是返回了一个`RealCall`对象，于是在那找到了这个的答案，`forWebSocket=false`
所以说这个`if`咱们不用管，直接看`promoteAndExecute()`方法：
```kotlin
private fun promoteAndExecute(): Boolean {
    // 不知道大家还记不记得咱们之前说的kotlin里面的check()语法，
    // 这个和check也一样，只不过抛出的是AssertionError异常
    assert(!Thread.holdsLock(this))

    // mutableListOf是kotlin里面的可变list集合
    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
        val i = readyAsyncCalls.iterator()
            while (i.hasNext()) {
                val asyncCall = i.next()

                if (runningAsyncCalls.size >= this.maxRequests) break // 最大容量
                if (asyncCall.callsPerHost().get() >= this.maxRequestsPerHost) continue // 主机最大容量

                i.remove()
                asyncCall.callsPerHost().incrementAndGet()
                executableCalls.add(asyncCall)
                runningAsyncCalls.add(asyncCall)
            }
        isRunning = runningCallsCount() > 0
    }

    for (i in 0 until executableCalls.size) {
        val asyncCall = executableCalls[i]
        asyncCall.executeOn(executorService)
    }

    return isRunning
}
```
首先将已就绪队列遍历一遍，判断正在运行的数量是不是大于定义的最大请求数，如果大于的话直接退出循环；如果不大于则在判断这个请求的主机请求数是不是大于定义的每个主机最大请求数，如果大于就跳过这个请求换下一个请求；不大于就把它调入正在运行的请求队列里面，直到遍历完成。然后判断还有没有正在运行的请求，如果有就`isRunning`置`true`。接着再取出`executableCalls`里的每一个元素，然后执行`executteOn()`方法。我们继续来看`AsyncCall`的`executeOn()`方法：
```kotlin
fun executeOn(executorService: ExecutorService) {
    assert(!Thread.holdsLock(client.dispatcher))
    var success = false
    try {
        executorService.execute(this)
        success = true
    } catch (e: RejectedExecutionException) {
        val ioException = InterruptedIOException("executor rejected")
        ioException.initCause(e)
        transmitter.noMoreExchanges(ioException)
        responseCallback.onFailure(this@RealCall, ioException)
    } finally {
        if (!success) {
            client.dispatcher.finished(this) // This call is no longer running!
        }
    }
}
```
这段代码就是在执行线程池中的线程，如果成功执行就将`success`置`true`，如果不成功，则抛异常并返回给`responseCallback`的`onFailure()`方法。并且如果没有成功执行也就是`success`为`false`，那么在`finally`中就会执行`client.dispatcher.finished()`方法:
```kotlin
internal fun finished(call: AsyncCall) {
    call.callsPerHost().decrementAndGet()
    finished(runningAsyncCalls, call)
  }
```
这个方法先将传入的`AsyncCall`的`callsPerHost`给减1，然后再调用了`finished()`方法，我们再来看这个`finished()`方法：

```kotlin
private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
    synchronized(this) {
        if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
        idleCallback = this.idleCallback
    }

    val isRunning = promoteAndExecute()

    if (!isRunning && idleCallback != null) {
        idleCallback.run()
    }
}
```
他讲此次请求从`runningAsyncCalls`中移除，然后执行了`promoteAndExecute()`方法，咱们在上面说过这个方法，他的返回值就是判断当前这个运行队列中还有没有请求，如果还有就返回`true`，没有就`false`。接着一个`if`，判断`isRunning`和`idleCallback`的，那么如果当前这个请求还没有执行的话，就调用`run()`方法执行当前请求。这样每个请求都执行完毕了。
那我们再来看看他的`run()`方法：
```kotlin
override fun run() {
      threadName("OkHttp ${redactedUrl()}") {
            var signalledCallback = false
            transmitter.timeoutEnter()
            try {
                val response = getResponseWithInterceptorChain()
                signalledCallback = true
                responseCallback.onResponse(this@RealCall, response)
            } catch (e: IOException) {
                if (signalledCallback) {
                    // Do not signal the callback twice!
                    Platform.get().log(INFO, "Callback failure for ${toLoggableString()}", e)
                } else {
                    responseCallback.onFailure(this@RealCall, e)
                }
            } finally {
                client.dispatcher.finished(this)
            }
      }
}
```
这块调用了一个`getResponseWithInterceptorChain()`方法，并返回了`response`，并将它返回给了`responseCallback.onResponse()`方法。如果失败了就将结果返回给`responseCallback.onFailure()`方法。最后调用`client.dispatcher.finished()`方法。

### 2.1.3 Interceptor拦截器
#### 2.1.3.1 getResponseWithInterceptorChain()方法
首先我们看看`getResponseWithInterceptorChain()`方法：
```kotlin
@Throws(IOException::class)
fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
        interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    val chain = RealInterceptorChain(interceptors, transmitter, null, 0, originalRequest, this,
        client.connectTimeoutMillis, client.readTimeoutMillis, client.writeTimeoutMillis)

    var calledNoMoreExchanges = false
    try {
        val response = chain.proceed(originalRequest)
        if (transmitter.isCanceled) {
            response.closeQuietly()
            throw IOException("Canceled")
        }
        return response
    } catch (e: IOException) {
        calledNoMoreExchanges = true
        throw transmitter.noMoreExchanges(e) as Throwable
    } finally {
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null)
        }
    }
}
```
首先就创建了一大堆的连接器并添加到`interceptors`集合中。然后创建了一个`RealInterceptorChain`对象，并调用了他的`proceed()`方法，接着主要目的就是讲`proceed()`返回的`response`给返回去。
那我们就来看看`RealInterceptorChain`的`proceed()`方法：

```kotlin
override fun proceed(request: Request): Response {
    return proceed(request, transmitter, exchange)
}
  
@Throws(IOException::class)
fun proceed(request: Request, transmitter: Transmitter, exchange: Exchange?): Response {
    if (index >= interceptors.size) throw AssertionError()

    calls++

    // If we already have a stream, confirm that the incoming request will use it.
    check(this.exchange == null || this.exchange.connection()!!.supportsUrl(request.url)) {
        "network interceptor ${interceptors[index - 1]} must retain the same host and port"
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    check(this.exchange == null || calls <= 1) {
        "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
    }

    // Call the next interceptor in the chain.
    val next = RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout)
    val interceptor = interceptors[index]

    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")

    // Confirm that the next interceptor made its required call to chain.proceed().
    check(exchange == null || index + 1 >= interceptors.size || next.calls == 1) {
        "network interceptor $interceptor must call proceed() exactly once"
    }

    check(response.body != null) { "interceptor $interceptor returned a response with no body" }

    return response
  }
```
首先就是一个`index`，`index`是`RealInterceptorChain`构造器中传入的参数，她是第四个参数，所以我们看`getResponseWithInterceptorChain()`方法中创建`RealInterceptorChain`对象时构造器的第四个传入的值为0。然后判断`index`的值是不是大于`interceptors`的大小，如果大于就抛异常，否则就继续一顿检查！！！然后再创建`RealInterceptorChain`对象，此时创建的对象传入的`index`为此时的`index+1`，然后再调用`interceptor`的`intercept()`方法，并返回`response`。
`interceptor`的`intercept()`作用是当存在多个拦截器时都会在上面代码注释1处阻塞，并等待下一个拦截器的调用返回。

#### 2.1.3.2 Interceptor源码
那现在我们再来讲几个重要的拦截器吧。
`OkHttp`中`Interceptor`的实现类有：
1. `ConnectInterceptor`：连接拦截器。 
2. `CallServerInterceptor`：请求服务器拦截器 
3. `CacheInterceptor`：缓存拦截器 
4. `BridgeInterceptor`：桥梁拦截器。 

其中较为重要的就是`ConnectInterceptor`和`CallServerInterceptor`，那我们来看看这两个。

##### 2.1.3.2.1 ConnectInterceptor
这个类主要用来实现网络请求连接。我们来看下他的`intercept`方法：
```kotlin
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val request = realChain.request()
    val transmitter = realChain.transmitter()

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    val doExtensiveHealthChecks = request.method != "GET"
    val exchange = transmitter.newExchange(chain, doExtensiveHealthChecks)

    return realChain.proceed(request, transmitter, exchange)
}
```
这个方法先将传入的`chain`对象造型成了`RealInterceptorChain`的对象，这个类我们在上面提到过，然后调用他的`response()`和`transmitter()`方法，分别得到当前`chain`的`response`和`transmitter`。然后执行了`request`的`method()`方法，判断`request`的类型是不是`GET`，如果是`doExtensiveHealthChecks`就为`false`，否则为`true`，接着把`doExtensiveHealthChecks`传入`transmitter的newExchange()`方法中去，这个方法我们等会再说，然后再调用了`proceed()`方法，这个方法我们在上面说过。

##### 2.1.3.2.2 CallServerInterceptor
这个类是网络请求的本质。它的`intercept`方法源码如下：
```kotlin
@Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange()
    val request = realChain.request()
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()

    // 写入请求头
    exchange.writeRequestHeaders(request)

    var responseHeadersStarted = false
    var responseBuilder: Response.Builder? = null
    // 写入请求体信息
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
        // 如果请求上有“expect:100 continue”头
        // 请等待“http/1.1 100 continue”响应，然后再传输请求主体.
        // 如果我们没有得到，返回我们得到的（例如4xx响应），而不传输请求体。
        if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
            exchange.flushRequest()
            responseHeadersStarted = true
            exchange.responseHeadersStart()
            responseBuilder = exchange.readResponseHeaders(true)
        }
        if (responseBuilder == null) {
            if (requestBody.isDuplex()) {
                // 准备一个双工主体，以便应用程序稍后可以发送请求主体。
                exchange.flushRequest()
                val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
                requestBody.writeTo(bufferedRequestBody)
            } else {
                // 如果满足“expect:100 continue”预期，则编写请求正文。
                val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
                requestBody.writeTo(bufferedRequestBody)
                bufferedRequestBody.close()
            }
        } else {
            exchange.noRequestBody()
            if (!exchange.connection()!!.isMultiplexed) {
                // 如果不满足“expect:100 continue”的要求，请防止重用HTTP/1连接。
                // 否则，我们仍然有义务传输请求主体以使连接保持一致状态。
                exchange.noNewExchangesOnConnection()
            }
        }
    } else {
        exchange.noRequestBody()
    }

    // 结束请求
    if (requestBody == null || !requestBody.isDuplex()) {
        exchange.finishRequest()
    }
    if (!responseHeadersStarted) {
        exchange.responseHeadersStart()
    }
    if (responseBuilder == null) {
        responseBuilder = exchange.readResponseHeaders(false)!!
    }
    // 读取响应头信息
    var response = responseBuilder
        .request(request)
        .handshake(exchange.connection()!!.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
    var code = response.code
    if (code == 100) {
        // 服务器发送了一个100继续，即使我们没有请求。
        // 再次尝试读取实际响应
        response = exchange.readResponseHeaders(false)!!
            .request(request)
            .handshake(exchange.connection()!!.handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build()
        code = response.code
    }

    exchange.responseHeadersEnd(response)

    // openResponseBody 获取响应体信息
    response = if (forWebSocket && code == 101) {
        // 连接正在升级，但我们需要确保拦截器看到非空的响应主体。
        response.newBuilder()
            .body(EMPTY_RESPONSE)
            .build()
        } else {
            response.newBuilder()
                .body(exchange.openResponseBody(response))
                .build()
        }
        if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
        "close".equals(response.header("Connection"), ignoreCase = true)) {
            exchange.noNewExchangesOnConnection()
        }
        if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
            throw ProtocolException(
                "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
        }
    
    //返回一个响应
    return response
}
```
具体过程可以看代码中的注释，它主要是向服务器发送请求数据和接受服务器返回的数据。

### 2.1.4 Transmiter
`Transmitter`类是`OkHttp`的应用层和网络层的一个桥梁类。

我们先来看看该类的初始化：
```kotlin
class Transmitter(
    private val client: OkHttpClient,
    private val call: Call
) {
    private val connectionPool: RealConnectionPool = client.connectionPool.delegate
    private val eventListener: EventListener = client.eventListenerFactory.create(call)
    private val timeout = object : AsyncTimeout() {
        override fun timedOut() {
            cancel()
        }
    }.apply {
        timeout(client.callTimeoutMillis.toLong(), MILLISECONDS)
    }
```
`Transmitter`主要的一些成员变量就这些，首先构造器中传入了两个参数，一个`OkHttpClient`，一个`Call`。然后又创建了一个连接池`connectionPool`，还有一个监听器，我们可以通过扩展这个类来监听程序的`HTTP`的调用数量、大小和持续时间。

### 2.1.5 RealConnection
我们先看看他的一些属性：
```kotlin
class RealConnection(
    val connectionPool: RealConnectionPool,
    private val route: Route
) : Http2Connection.Listener(), Connection {

    // 以下字段由connect（）初始化，从不重新分配。

    // 底层socket
    private var rawSocket: Socket? = null

    /**
     * 应用层套接字。如果此连接不使用SSL，则可以是[sslsocket]分层在[rawsocket]上，也可以是[rawsocket]本身。
    */
    // 应用层socket
    private var socket: Socket? = null
    // 握手
    private var handshake: Handshake? = null
    //  协议
    private var protocol: Protocol? = null
    // http2的连接
    private var http2Connection: Http2Connection? = null
    // 与服务器交互的输入输出流
    private var source: BufferedSource? = null
    private var sink: BufferedSink? = null

    // 跟踪连接状态下的字段由连接池保护。

    /**
     * 如果为true，则不能在此连接上创建新的交换。一旦是true的，这总是true的。
     * 由[ConnectionPool]监视。
     */
    var noNewExchanges = false

    /**
     * 建立可能由于所选路由而导致的流时出现问题的次数。由[ConnectionPool]保护。
     */
    internal var routeFailureCount = 0

    internal var successCount = 0
    private var refusedStreamCount = 0

    /**
     * 此连接可以承载的最大并发流数。
     * 如果“allocations.size（）<allocationlimit”，则可以在此连接上创建新流。     
     */ 
    private var allocationLimit = 1
```
接下来我们看看他的`connect()`方法：
```kotlin
fun connect(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    pingIntervalMillis: Int,
    connectionRetryEnabled: Boolean,
    call: Call,
    eventListener: EventListener
) {
    check(protocol == null) { "already connected" }

    // 线路的选择
    var routeException: RouteException? = null
    val connectionSpecs = route.address.connectionSpecs
    val connectionSpecSelector = ConnectionSpecSelector(connectionSpecs)

    if (route.address.sslSocketFactory == null) {
        if (ConnectionSpec.CLEARTEXT !in connectionSpecs) {
            throw RouteException(UnknownServiceException(
                "CLEARTEXT communication not enabled for client"))
        }
        val host = route.address.url.host
        if (!Platform.get().isCleartextTrafficPermitted(host)) {
            throw RouteException(UnknownServiceException(
                "CLEARTEXT communication to $host not permitted by network security policy"))
        }
    } else {
        if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
            throw RouteException(UnknownServiceException(
                "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"))
        }
    }

    // 连接开始
    while (true) {
        try {
            // 如果要求隧道模式，建立通道连接，通常不是这种
            if (route.requiresTunnel()) {
                connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener)
                if (rawSocket == null) {
                    // 我们无法连接隧道，但适当关闭了我们的资源。
                    break
                }
            } else {
                // 一般都走这条逻辑了，实际上很简单就是socket的连接
                connectSocket(connectTimeout, readTimeout, call, eventListener)
            }
            // https的建立
            establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener)
            eventListener.connectEnd(call, route.socketAddress, route.proxy, protocol)
            break
        } catch (e: IOException) {
            socket?.closeQuietly()
            rawSocket?.closeQuietly()
            socket = null
            rawSocket = null
            source = null
            sink = null
            handshake = null
            protocol = null
            http2Connection = null

            eventListener.connectFailed(call, route.socketAddress, route.proxy, null, e)

            if (routeException == null) {
                routeException = RouteException(e)
            } else {
                routeException.addConnectException(e)
            }

            if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
                throw routeException
            }
        }
    }

    if (route.requiresTunnel() && rawSocket == null) {
        throw RouteException(ProtocolException(
            "Too many tunnel connections attempted: $MAX_TUNNEL_ATTEMPTS"))
    }

    val http2Connection = this.http2Connection
    if (http2Connection != null) {
        synchronized(connectionPool) {
            allocationLimit = http2Connection.maxConcurrentStreams()
        }
    }
}
```
首先检查是否已经建立连接，如果已经建立就抛异常，没有的话就继续。接着就得到了`ConnectionSpecs`，然后根据他建立了一个`connectionSpecSelector`集合。接着判断是不是安全连接，也就是ssl连接，如果不是的话就判断了一些属性，先确定是不是明文然后再确定主机能不能接受明文操作。接着就开始连接，判断是不是要进行隧道通信，如果是就调用`connectTunnel()`建立隧道通信，如果不是就调用`connectSocket()`建立普通的通信。然后通过`establishProtocol()`建立协议。如果是`HTTP/2`就设置相关属性。

然后我们就来看看他具体如何实现的，先看看`connectSocket()`方法：
```kotlin
@Throws(IOException::class)
private fun connectSocket(
    connectTimeout: Int,
    readTimeout: Int,
    call: Call,
    eventListener: EventListener
) {
    val proxy = route.proxy
    val address = route.address

    // 根据代理类型选择socket类型是代理还是直连
    val rawSocket = when (proxy.type()) {
        Proxy.Type.DIRECT, Proxy.Type.HTTP -> address.socketFactory.createSocket()!!
        else -> Socket(proxy)
    }
    this.rawSocket = rawSocket

    eventListener.connectStart(call, route.socketAddress, proxy)
    rawSocket.soTimeout = readTimeout
    try {
        // 连接socket，之所以这样写是因为支持不同的平台
        // 里面实际上是  socket.connect(address, connectTimeout);
        Platform.get().connectSocket(rawSocket, route.socketAddress, connectTimeout)
    } catch (e: ConnectException) {
        throw ConnectException("Failed to connect to ${route.socketAddress}").apply {
            initCause(e)
        }
    }

    // 下面的Try/Catch块是一种避免Android 7.0崩溃的伪黑客方法
    // More details:
    // https://github.com/square/okhttp/issues/3245
    // https://android-review.googlesource.com/#/c/271775/
    try {
        // 得到输入／输出流
        source = rawSocket.source().buffer()
        sink = rawSocket.sink().buffer()
    } catch (npe: NullPointerException) {
        if (npe.message == NPE_THROW_WITH_NULL) {
            throw IOException(npe)
        }
    }
}
```
首先先判断连接类型，如果是直连或者`HTTP`连接就直连，否则的话走`Socket`代理，然后通过`eventListener.connectStart()`方法创建连接，再设定超时->完成连接->创建用于I/O的`source`和`sink`。

我们接着再来看`connectTunnel()`方法：
```kotlin
@Throws(IOException::class)
private fun connectTunnel(
    connectTimeout: Int,
    readTimeout: Int,
    writeTimeout: Int,
    call: Call,
    eventListener: EventListener
) {
    var tunnelRequest: Request = createTunnelRequest()
    val url = tunnelRequest.url
    for (i in 0 until MAX_TUNNEL_ATTEMPTS) {
        connectSocket(connectTimeout, readTimeout, call, eventListener)
        tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url)
            ?: break // 已成功创建隧道。

        // 代理决定在身份验证质询后关闭连接。
        // 我们需要创建一个新的连接，但这次需要使用身份验证凭据。
        rawSocket?.closeQuietly()
        rawSocket = null
        sink = null
        source = null
        eventListener.connectEnd(call, route.socketAddress, route.proxy, null)
    }
}
```
大体就是先创建隧道请求，然后建立`socket`连接，再发送请求建立隧道。

# 3. 请求流程图
那我们最后来总结下
## 3.1 同步请求是如何操作的？
![OkHttpSyn](https://cdn.littlecorgi.top/mweb/2019-07-26/OkHttpSync.jpg)

## 3.2 异步请求是如何操作的？
![OkHttpAsyn](https://cdn.littlecorgi.top/mweb/2019-07-26/OkHttpAsync.jpg)
