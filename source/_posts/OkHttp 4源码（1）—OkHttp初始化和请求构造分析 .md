---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144805.png
tags: 
- OkHttp
categories:
- [Android, 框架]
---

>本文基于OkHttp 4.3.1源码分析
>[OkHttp - 官方地址](https://square.github.io/okhttp/)
[OkHttp - GitHub代码地址](https://github.com/square/okhttp)


## 概括
本篇主要从OkHttp的两个请求示例开始，对Okhttp的初始化工作，和请求从构造、分发到执行的流程进行源码分析介绍

OkHttp整体流程（本文覆盖红色部分）

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144743.png)

本文覆盖代码流程图

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144805.png)

## 示例
使用OkHttp一般流程，初始化一个共享OkHttpClient，构建Request，然后OkHttpClient根据Request构建Call，接着执行call，最后进行Response处理

### 同步请求

```
public class GetExample {
  OkHttpClient client = new OkHttpClient(); // 构建共享的Client

  String run(String url) throws IOException {
    Request request = new Request.Builder() // 构建request
        .url(url)
        .build();
    // 构建Call，执行
    try (Response response = client.newCall(request).execute()) {
      return response.body().string();
    }
  }

  public static void main(String[] args) throws IOException {
    GetExample example = new GetExample();
    String response = example.run("https://raw.github.com/square/okhttp/master/README.md");
    System.out.println(response);
  }
}
```
### 异步请求
```
public final class AsynchronousGet {
  private final OkHttpClient client = new OkHttpClient();// 构建共享Client

  public void run() throws Exception {
    // 构建Request
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();
    // 构建Call，执行，回调接受处理
    client.newCall(request).enqueue(new Callback() {
      @Override public void onFailure(Call call, IOException e) {
        e.printStackTrace();
      }

      @Override public void onResponse(Call call, Response response) throws IOException {
        try (ResponseBody responseBody = response.body()) {
          if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

          Headers responseHeaders = response.headers();
          for (int i = 0, size = responseHeaders.size(); i < size; i++) {
            System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
          }

          System.out.println(responseBody.string());
        }
      }
    });
  }

  public static void main(String... args) throws Exception {
    new AsynchronousGet().run();
  }
}
```

## 源码分析

### 构建OkHttpClient 

* OkHttpClient是Call的一个工厂类，OkHttpClient应该是共享的，或者说是单例
* 可以通过newBuilder来自定义Client
* 没有必要关心 关闭和资源释放


```
/*
 * Factory for [calls][Call], which can be used to send HTTP requests and read their responses.
 * ## OkHttpClients Should Be Shared
 * ## Customize Your Client With newBuilder()
 * ## Shutdown Isn't Necessary
 * /
open class OkHttpClient internal constructor(
  builder: Builder
) : Cloneable, Call.Factory, WebSocket.Factory {
    // 3. 成员变量初始化
    ... 
    
    // 1. 内部无参数构造函数
    constructor() : this(Builder())
    
    // 4. OkHttpClient函数初始化 
    init {
        // 初始化证书和拦截器等判断逻辑
        ...
    }
    // 2. Builder构造函数
    class Builder constructor() {
        ...
    } 
}
```
#### OkHttpClient.Builder
Builder模式，提供自定义配置化能力，同时有一份无需关心的默认配置

```
  class Builder constructor() {
        internal var dispatcher: Dispatcher = Dispatcher()
        internal var connectionPool: ConnectionPool = ConnectionPool()
        internal val interceptors: MutableList<Interceptor> = mutableListOf()
        internal val networkInterceptors: MutableList<Interceptor> = mutableListOf()
        internal var eventListenerFactory: EventListener.Factory = EventListener.NONE.asFactory()
        internal var retryOnConnectionFailure = true
        internal var authenticator: Authenticator = Authenticator.NONE
        internal var followRedirects = true
        internal var followSslRedirects = true
        internal var cookieJar: CookieJar = CookieJar.NO_COOKIES
        internal var cache: Cache? = null
        internal var dns: Dns = Dns.SYSTEM
        internal var proxy: Proxy? = null
        internal var proxySelector: ProxySelector? = null
        internal var proxyAuthenticator: Authenticator = Authenticator.NONE
        internal var socketFactory: SocketFactory = SocketFactory.getDefault()
        internal var sslSocketFactoryOrNull: SSLSocketFactory? = null
        internal var x509TrustManagerOrNull: X509TrustManager? = null
        internal var connectionSpecs: List<ConnectionSpec> = DEFAULT_CONNECTION_SPECS
        internal var protocols: List<Protocol> = DEFAULT_PROTOCOLS
        internal var hostnameVerifier: HostnameVerifier = OkHostnameVerifier
        internal var certificatePinner: CertificatePinner = CertificatePinner.DEFAULT
        internal var certificateChainCleaner: CertificateChainCleaner? = null
        internal var callTimeout = 0
        internal var connectTimeout = 10_000
        internal var readTimeout = 10_000
        internal var writeTimeout = 10_000
        internal var pingInterval = 0
    }   

```
#### OkHttpClient成员变量初始化
* 初始化成员变量
* JvmName是为了支持兼容3.x

```
  @get:JvmName("dispatcher") val dispatcher: Dispatcher = builder.dispatcher

  @get:JvmName("connectionPool") val connectionPool: ConnectionPool = builder.connectionPool

  @get:JvmName("interceptors") val interceptors: List<Interceptor> =
      builder.interceptors.toImmutableList()

  @get:JvmName("networkInterceptors") val networkInterceptors: List<Interceptor> =
      builder.networkInterceptors.toImmutableList()

  @get:JvmName("eventListenerFactory") val eventListenerFactory: EventListener.Factory =
      builder.eventListenerFactory

  @get:JvmName("retryOnConnectionFailure") val retryOnConnectionFailure: Boolean =
      builder.retryOnConnectionFailure

  @get:JvmName("cookieJar") val cookieJar: CookieJar = builder.cookieJar

  @get:JvmName("cache") val cache: Cache? = builder.cache

  ....
```

### 构建Request
Request 对应HTTP请求中的Request，OkHttp依旧是Builder构建模式构建Request

#### Request.Builder
支持url、method、headers、body的配置

```
 open class Builder {
    internal var url: HttpUrl? = null 
    internal var method: String
    internal var headers: Headers.Builder
    internal var body: RequestBody? = null
    // 构造Request
    open fun build(): Request {
      return Request(
          checkNotNull(url) { "url == null" },
          method,
          headers.build(),
          body,
          tags.toImmutableMap()
      )
    }
}
```

#### Request
通过Request.Builder构造Request实例

```
class Request internal constructor(
  @get:JvmName("url") val url: HttpUrl,
  @get:JvmName("method") val method: String,
  @get:JvmName("headers") val headers: Headers,
  @get:JvmName("body") val body: RequestBody?,
  internal val tags: Map<Class<*>, Any>
) {
    ...
}
```

### 构建Call

#### OkHttpClient.newCall
OkHttpClient 实现了Call.Factory，作为Call的构造工厂类

```
  override fun newCall(request: Request): Call {
    // 执行 RealCall的构造call方法
    return RealCall.newRealCall(this, request, forWebSocket = false)
  }
```

#### RealCall.newRealCall
构造Call真正方法，另外创建了一个 发射器，接下来先了解下Call

```
companion object {
    fun newRealCall(
      client: OkHttpClient,
      originalRequest: Request,
      forWebSocket: Boolean
    ): RealCall {
      
      return RealCall(client, originalRequest, forWebSocket).apply {
       // 构造了一个 发射器，它是应用层和网络层交互的桥梁，后面会着重介绍
        transmitter = Transmitter(client, this) 
      }
    }
  }
```

#### RealCall
Call定义为一个准备好执行的请求，它是能被取消的，且它只能被执行一次（http请求也是一次执行）
**包括一个核心成员变量 Transmitter ，两个重要方法 execute（同步） 和 enqueue（异步）**


```

internal class RealCall private constructor(
  val client: OkHttpClient,
  /** The application's original request unadulterated by redirects or auth headers. */
  val originalRequest: Request,
  val forWebSocket: Boolean
) : Call {
 // 发射机
 private lateinit var transmitter: Transmitter
 
 // 同步请求执行方法
 override fun execute(): Response {
  ...
 }
  // 异步请求执行方法 
  override fun enqueue(responseCallback: Callback) {
    ...
  }}
  
  // 取消请求
  override fun cancel() {
    transmitter.cancel()
  }
```

### 同步请求

#### RealCall.execute
* 请求前校验逻辑，仅能执行一次，和过期时间逻辑判断逻辑
* 通知请求start事件，便于metrics 指标数据收集
* 将call加入分发器的同步请求队列中
* 通过拦截器责任链模式进行请求和返回一系列逻辑处理

```
  override fun execute(): Response {
    // 检查是否已经执行
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    // 过期时间逻辑，如果配置了会有WatchDog线程进行Watch然后执行退出逻辑
    transmitter.timeoutEnter()
    // 通知start，最后会通过 EventListener 发出时间，主要目的是收集 metrics events
    transmitter.callStart()
    try {
      // 调用client的dispatcher分发器执行call（将call加入同步call队列）
      client.dispatcher.executed(this)
      // 通过拦截器责任链模式进行请求和返回处理等一系类逻辑
      return getResponseWithInterceptorChain()
    } finally {
      client.dispatcher.finished(this)
    }
  }
```

#### RealCall.getResponseWithInterceptorChain
* 配置拦截器，所有请求和响应处理逻辑解耦到各个拦截器负责模块
* 构造拦截器链式调用处理类RealInterceptorChain实例
* chain.proceed 进行拦截器的链式调用


```
  fun getResponseWithInterceptorChain(): Response {
    // 构建所有的拦截器
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors // client配置的拦截器
    interceptors += RetryAndFollowUpInterceptor(client) // 重试机制拦截器
    interceptors += BridgeInterceptor(client.cookieJar) // 请求和返回桥（http信息配置和解析）拦截器
    interceptors += CacheInterceptor(client.cache) // 缓存拦截
    interceptors += ConnectInterceptor // 连接拦截器
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)  // 执行拦截器
    
    // 拦截器核心处理类 
    val chain = RealInterceptorChain(interceptors, transmitter, null, 0, originalRequest, this,
        client.connectTimeoutMillis, client.readTimeoutMillis, client.writeTimeoutMillis)

    var calledNoMoreExchanges = false
    try {
      // 链式调用
      val response = chain.proceed(originalRequest)
      if (transmitter.isCanceled) { // 处理取消
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response //返回请求结果
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

### 异步请求

#### RealCall.enqueue
* 构造一个异步call
* 调用dispatcher 入队AsyncCall

```
  override fun enqueue(responseCallback: Callback) {
    synchronized(this) { 
      check(!executed) { "Already Executed" }
      executed = true
    }
    transmitter.callStart()
    // 构造 AsyncCall ，接着分发器入队操作
    client.dispatcher.enqueue(AsyncCall(responseCallback)) 
  }
```

#### Dispatcher 分发器队Call进行分发执行
[参考：线程池理解](https://www.jianshu.com/p/d9e46d5a4af9)

* 初始化时，构造分发器的线程池，及对应执行参数
* 主要管理异步请求的处理（异步Call入队、并发执行）

```
class Dispatcher constructor() {
  // 默认最大请求数 64
  @get:Synchronized var maxRequests = 64
  // 默认最大并发Host 5
  @get:Synchronized var maxRequestsPerHost = 5
  // 线程池执行器 默认创建可缓存线程池 
  @get:JvmName("executorService") val executorService: ExecutorService
    get() {
      if (executorServiceOrNull == null) {
        executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
            SynchronousQueue(), threadFactory("OkHttp Dispatcher", false))
      }
      return executorServiceOrNull!!
    }
    
  /** 准备好的异步Call对列 */
  private val readyAsyncCalls = ArrayDeque<AsyncCall>()

  /** 执行中异步call对列 */
  private val runningAsyncCalls = ArrayDeque<AsyncCall>()

  /** 同步call对列 */
  private val runningSyncCalls = ArrayDeque<RealCall>()
  
  // 异步Call 入队操作  
  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      readyAsyncCalls.add(call)
      // same host情况下的复用逻辑处理
      if (!call.get().forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host())
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    promoteAndExecute() // 准备线程池执行器，及执行
  }

// 执行Call
private fun promoteAndExecute(): Boolean {
    this.assertThreadDoesntHoldLock()

    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      // 遍历所有的ready 异步 call
      val i = readyAsyncCalls.iterator()
      while (i.hasNext()) {
        val asyncCall = i.next()

        if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
        if (asyncCall.callsPerHost().get() >= this.maxRequestsPerHost) continue // Host max capacity.

        i.remove()
        asyncCall.callsPerHost().incrementAndGet()
        executableCalls.add(asyncCall) // 加入此次执行队列缓存
        runningAsyncCalls.add(asyncCall) // 加入正在执行队列
      }
      isRunning = runningCallsCount() > 0
    }

    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService) // 将线程执行器传入call，在Call中进行执行
    }

    return isRunning
  }
  
  /** 同步call，添加到队列 */
  @Synchronized internal fun executed(call: RealCall) {
    runningSyncCalls.add(call)
  }

}
```

#### AsyncCall执行
* executeOn ，线程池执行器执行Runnable
* run，通过拦截器进行请求和获取响应结果

```
fun executeOn(executorService: ExecutorService) {
    client.dispatcher.assertThreadDoesntHoldLock()
    
    var success = false
    try {
        // 线程池 执行器执行
        executorService.execute(this)
        success = true
    }
}
// 执行方法
override fun run() {
    threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        transmitter.timeoutEnter()
        try {
            // 同“同步请求” 最终执行拦截器链式调用
            val response = getResponseWithInterceptorChain()
            signalledCallback = true
            // 响应 回调
            responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
            ...
        } finally {
            client.dispatcher.finished(this)
        }
    }
}
```


[下一篇 OkHttp 4源码（2）— 拦截器机制分析](https://www.jianshu.com/p/0c830962c6e3)


