---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143543.png
tags: 
- OkHttp
categories:
- [Android, 框架]
---

>本文基于OkHttp 4.3.1源码分析
>[OkHttp - 官方地址](https://square.github.io/okhttp/)
[OkHttp - GitHub代码地址](https://github.com/square/okhttp)

## 概述

OkHttp整体流程（本文覆盖红色部分）

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144715.png)

Interceptor流程

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143543.png)



## 责任链模式

概念
责任链模式（Chain of Responsibility Pattern），包含命令对象和一系列实现了相同接口的处理对象，这些处理对象相互连接成为一条责任链。每一个处理对象能决定它能处理哪些命令对象，对于它不能处理的命令对象，将会传递给该链中的下一个处理对象。负责链模式就是对这种顺序处理事件的行为的抽象,通过接口来定义处理事件的方法，顺序分发/处理事件。

主要解决
职责链上的处理者负责处理请求，客户只需要将请求发送到职责链上即可，无须关心请求的处理细节和请求的传递，所以职责链将请求的发送者和请求的处理者解耦了。

优点

1. 降低耦合度。它将请求的发送者和接收者解耦。 
2. 简化了对象。使得对象不需要知道链的结构。 
3. 增强给对象指派职责的灵活性。通过改变链内的成员或者调动它们的次序，允许动态地新增或者删除责任。
4. 增加新的请求处理类很方便。

特点

1. 每个责任人实现相同的接口,处理一个事件对象
2. 让事件对象责任人之间顺序传递
3. 事件的处理结果的返回是逆序的
4. 责任链中的每个责任人都可以有权不继续传递事件,以自身为终点处理事件返回结果

OkHttp中应用
在 OkHttp 中，命令对象就是 Request 对象，处理对象就是每一个 Interceptor 对象。每个 interceptor 对 request 进行一些步骤的处理，而将其余的工作交给下一个 interceptor。另外，责任链中的处理对象如果可以全权处理命令对象，则不需要交给下一个处理对象。OkHttp 中的 CacheInterceptor 也是具有全权处理的能力。如果请求的结果已经缓存，则不需要再交给 ConnectInterceptor 等进行连接服务器、发送请求的处理，直接返回已缓存的 response 即可。

## 源码分析

### Interceptor
拦截器，即处理对象类，Chain和Interceptor共同实现顺序调用，最终反响依次返回Response
拦截器一般可以概括三个步骤

1. 请求前处理逻辑
2. 执行 chain.proceed，顺序调用
3. 获取响应结果后逻辑处理

默认拦截器依次包括

1. RetryAndFollowUpInterceptor （重试机制）
2. BridgeInterceptor    （Http连接转换桥）
3. CacheInterceptor     （缓存策略）
4. ConnectInterceptor   （连接）
5. CallServerInterceptor  （响应读取）

```
// 拦截器，处理对象
interface Interceptor {
  // 拦截方法，接受拦截器链（包括命令对象Request及Call、和一系列辅助对象），返回处理结果
  fun intercept(chain: Chain): Response

  // 链接口
  interface Chain {
    // 核心方法， 顺序执行 处理方法
    fun proceed(request: Request): Response
    ... 
  }
}

```

### RealInterceptorChain 拦截器链
完成责任链模式核心类，拦截器顺序执行、反向返回及中途中断核心实现
前一篇文章知道，命令对象RealCall，执行RealInterceptorChain的proceed方法，该方法最后会按照拦截器的加入顺序，循环构建Chain对象，然后执行拦截器的拦截方法，执导最后一个拦截器或者是中途拦截器直接返回了。

```
class RealInterceptorChain(
  private val interceptors: List<Interceptor>, // 拦截器，处理对象
  private val transmitter: Transmitter, // 发射机
  private val exchange: Exchange?, // 交换器 请求和响应 
  private val index: Int,
  private val request: Request, // 请求
  private val call: Call,  // 一次请求和返回封装
  private val connectTimeout: Int,
  private val readTimeout: Int,
  private val writeTimeout: Int
) : Interceptor.Chain {
  ...

  override fun proceed(request: Request): Response {
    return proceed(request, transmitter, exchange)
  }

  // 顺序调用方法 
  fun proceed(request: Request, transmitter: Transmitter, exchange: Exchange?): Response {
    ...
    calls++
    ...
    // 构造一个 RealInterceptorChain
    val next = RealInterceptorChain(interceptors, transmitter, exchange,
        index + 1, request, call, connectTimeout, readTimeout, writeTimeout)
    // 根据index 依次取拦截器
    val interceptor = interceptors[index]

    // 所有拦截器顺序执行，拦截器方法中执行 chain.proceed完成reponse返回，将命令对象交给下一个拦截器进行处理，如果不执行chain.proceed则在该拦截器进行终结。
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")
    ...
    return response
  }

}
```

### RetryAndFollowUpInterceptor
重试拦截器

#### RetryAndFollowUpInterceptor.intercept
* 请求连接准备工作
* 执行chain.proceed顺序执行，获取response
* 若出现异常，则判断是否需要进行重试机制
* 根据服务端返回的状态码，进行重定向请求生成
* 进行重试，直到不需要重试或者成功，最多20次

```
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    var request = chain.request()
    val realChain = chain as RealInterceptorChain
    val transmitter = realChain.transmitter()
    var followUpCount = 0
    var priorResponse: Response? = null
    while (true) { // 循环
      // 发射机 执行连接准备工作
      transmitter.prepareToConnect(request)
    
      if (transmitter.isCanceled) { // 处理取消的请求
        throw IOException("Canceled")
      }

      var response: Response
      var success = false
      try {
        // 执行 proceed顺序执行流程，最终获取response
        response = realChain.proceed(request, transmitter, null)
        success = true
      } catch (e: RouteException) {
        // 若不满足重试条件，则抛出异常
        if (!recover(e.lastConnectException, transmitter, false, request)) {
          throw e.firstConnectException
        }
        continue    // 满足，则重试
      } catch (e: IOException) {
        // 若不满足重试条件，则抛出异常
        val requestSendStarted = e !is ConnectionShutdownException
        if (!recover(e, transmitter, requestSendStarted, request)) throw e
        continue    // 满足，则重试
      } finally {
        // 异常抛出，则success=false，执行释放资源
        if (!success) {
          transmitter.exchangeDoneDueToException()
        }
      }

      if (priorResponse != null) { // 赋值上一次的response
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }

      val exchange = response.exchange
      val route = exchange?.connection()?.route()
      // 获取进一步重试的请求
      val followUp = followUpRequest(response, route)

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          transmitter.timeoutEarlyExit() // 无重试，停止timeout并退出
        }
        return response
      }

      val followUpBody = followUp.body
      // 如果期望执行一次，则结束
      if (followUpBody != null && followUpBody.isOneShot()) { 
        return response
      }

      response.body?.closeQuietly()
      if (transmitter.hasExchange()) {
        exchange?.detachWithViolence()
      }
      // 重试不超过20此
      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }
      // 修改下次的请求
      request = followUp
      // 记录上一次的response
      priorResponse = response
    }
  }
  
```

#### RetryAndFollowUpInterceptor.followUpRequest
主要针对重定向的几个状态码进行特殊里，从Header中取Location字段，构造重定向request

```
private fun followUpRequest(userResponse: Response, route: Route?): Request? {
    val responseCode = userResponse.code

    val method = userResponse.request.method
    when (responseCode) {
      HTTP_PROXY_AUTH -> { // 407
        val selectedProxy = route!!.proxy
        if (selectedProxy.type() != Proxy.Type.HTTP) {
          throw ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy")
        }
        // 代理身份认证
        return client.proxyAuthenticator.authenticate(route, userResponse)
      }
      // 身份认证
      HTTP_UNAUTHORIZED -> return client.authenticator.authenticate(route, userResponse)
      // 针对Get、Head请求 执行构建重定向请求
      HTTP_PERM_REDIRECT, HTTP_TEMP_REDIRECT -> { // 307 、308
        if (method != "GET" && method != "HEAD") {
          return null
        }
        return buildRedirectRequest(userResponse, method)
      }
      // 针对300、301、302、303 执行构建重定向请求
      HTTP_MULT_CHOICE, HTTP_MOVED_PERM, HTTP_MOVED_TEMP, HTTP_SEE_OTHER -> {
        return buildRedirectRequest(userResponse, method)
      }
        
      HTTP_CLIENT_TIMEOUT -> {
        // 408 ，说明需要重新发送一次相同的请求
        ... // 判断是否有必要重新发送
        return userResponse.request
      }

      HTTP_UNAVAILABLE -> { // 503
        return null
      }
      else -> return null
    }
  }

private fun buildRedirectRequest(userResponse: Response, method: String): Request? {
    // 设置不需要执行重定向
    if (!client.followRedirects) return null
    // location header 是否支持
    val location = userResponse.header("Location") ?: return null
    // Don't follow redirects to unsupported protocols.
    val url = userResponse.request.url.resolve(location) ?: return null

    // 如果不允许SSL和Non-SSL之间的重定向，则返回null
    val sameScheme = url.scheme == userResponse.request.url.scheme
    if (!sameScheme && !client.followSslRedirects) return null

    // Most redirects don't include a request body.
    val requestBuilder = userResponse.request.newBuilder()
    if (HttpMethod.permitsRequestBody(method)) {
      val maintainBody = HttpMethod.redirectsWithBody(method)
      if (HttpMethod.redirectsToGet(method)) {
        requestBuilder.method("GET", null)
      } else {
        val requestBody = if (maintainBody) userResponse.request.body else null
        requestBuilder.method(method, requestBody)
      }
      if (!maintainBody) {
        requestBuilder.removeHeader("Transfer-Encoding")
        requestBuilder.removeHeader("Content-Length")
        requestBuilder.removeHeader("Content-Type")
      }
    }

    // When redirecting across hosts, drop all authentication headers. This
    // is potentially annoying to the application layer since they have no
    // way to retain them.
    if (!userResponse.request.url.canReuseConnectionFor(url)) {
      requestBuilder.removeHeader("Authorization")
    }
    // 构建 新的请求
    return requestBuilder.url(url).build()
  }
```


### BridgeInterceptor

>Bridges from application code to network code. 

连接应用层和网络层的桥梁

* 主要针对请求Header进行处理，如gzip、cookie及其他Header字段等
* 响应则进行gzip解压


```
 
 class BridgeInterceptor(private val cookieJar: CookieJar) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val userRequest = chain.request()
    val requestBuilder = userRequest.newBuilder()
    
    // 将userRequest中的属性设置到request中
    val body = userRequest.body
    if (body != null) {
      val contentType = body.contentType()
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString())
      }

      val contentLength = body.contentLength()
      if (contentLength != -1L) {
        requestBuilder.header("Content-Length", contentLength.toString())
        requestBuilder.removeHeader("Transfer-Encoding")
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked")
        requestBuilder.removeHeader("Content-Length")
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", userRequest.url.toHostHeader())
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive")
    }

    // 如果没有设置Accept-Encoding，则自动设置为gzip
    var transparentGzip = false
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true
      requestBuilder.header("Accept-Encoding", "gzip")
    }
    // 将userRequest中的cookie解析出来并设置到Request中的Cookie字段中
    val cookies = cookieJar.loadForRequest(userRequest.url)
    if (cookies.isNotEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies))
    }
    // 设置 User-Agent
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", userAgent)
    }
    // 顺序执行处理方法
    val networkResponse = chain.proceed(requestBuilder.build())

    // 处理响应的headers，并缓存cookie
    cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

    val responseBuilder = networkResponse.newBuilder()
        .request(userRequest)
    // 设置了Gzip，且响应结果支持gzip，则进行Gzip解析
    if (transparentGzip &&
        "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
        networkResponse.promisesBody()) {
        
      val responseBody = networkResponse.body
      if (responseBody != null) {
        val gzipSource = GzipSource(responseBody.source())
        val strippedHeaders = networkResponse.headers.newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build()
        responseBuilder.headers(strippedHeaders)
        val contentType = networkResponse.header("Content-Type")
        responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
      }
    }

    return responseBuilder.build()
  }

  /** Returns a 'Cookie' HTTP request header with all cookies, like `a=b; c=d`. */
  private fun cookieHeader(cookies: List<Cookie>): String = buildString {
    cookies.forEachIndexed { index, cookie ->
      if (index > 0) append("; ")
      append(cookie.name).append('=').append(cookie.value)
    }
  }
}
```

[下一篇 OkHttp 4源码（3）— 缓存机制分析](https://www.jianshu.com/p/2eafcd161dd9)

