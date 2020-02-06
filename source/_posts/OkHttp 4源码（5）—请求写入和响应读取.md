---
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
![](https://upload-images.jianshu.io/upload_images/9696036-45a52add159e06b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

IO流程图
![](https://upload-images.jianshu.io/upload_images/9696036-4f71931e4b7a5416.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 源码分析
### 拦截器
#### CallServerInterceptor.intercept
整体可以划分6个步骤，根据不同协议执行实现逻辑区分Http1.x和Http2

1. 写请求头
2. 创建请求体
3. 写请求体
4. 完成请求写入
5. 读取响应头
6. 返回响应结果


```
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    val exchange = realChain.exchange()
    val request = realChain.request()
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()
    // 写入请求头
    exchange.writeRequestHeaders(request)

    var invokeStartEvent = true
    var responseBuilder: Response.Builder? = null
    // 针对支持body的请求，且body不为空的情况进行请求体写入
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
      // 对 “100-continue” 进行处理
      if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
        exchange.flushRequest()
        responseBuilder = exchange.readResponseHeaders(expectContinue = true)
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
      if (responseBuilder == null) {
        if (requestBody.isDuplex()) {
          // 写入请求体
          exchange.flushRequest()
          val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
          requestBody.writeTo(bufferedRequestBody)
        } else {
          val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
          requestBody.writeTo(bufferedRequestBody)
          bufferedRequestBody.close()
        }
      } else {
        exchange.noRequestBody()
        if (!exchange.connection()!!.isMultiplexed) {
          exchange.noNewExchangesOnConnection()
        }
      }
    } else {
      exchange.noRequestBody()
    }

    if (requestBody == null || !requestBody.isDuplex()) {
      exchange.finishRequest() // 写入请求
    }
    if (responseBuilder == null) {
      // 读取响应头，写入到创建的ResponseBuilder
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
    }
    // 创建响应
    var response = responseBuilder
        .request(request)
        .handshake(exchange.connection()!!.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
    var code = response.code
    if (code == 100) {
      // 响应码为100的情况下，再次读取响应头，重新构建response
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
      }
      response = responseBuilder
          .request(request)
          .handshake(exchange.connection()!!.handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
      code = response.code
    }
    // 读取响应头结束
    exchange.responseHeadersEnd(response)

    response = if (forWebSocket && code == 101) {
      // webSocket & 101情况下，返回一个empty的reponse
      response.newBuilder()
          .body(EMPTY_RESPONSE)
          .build()
    } else {
      // 输入响应体，创建响应
      response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build()
    }
    
    ... // check
    return response
  }
}


```


### 写请求头
#### Exchange.writeRequestHeaders

```
  fun writeRequestHeaders(request: Request) {
    try {
      eventListener.requestHeadersStart(call)
      codec.writeRequestHeaders(request) // 调用codec执行写headers
      eventListener.requestHeadersEnd(call, request)
    } catch (e: IOException) {
      ...
    }
  }
```
### 创建请求体
#### Exchange.createRequestBody

```
  fun createRequestBody(request: Request, duplex: Boolean): Sink {
    this.isDuplex = duplex
    val contentLength = request.body!!.contentLength()
    eventListener.requestBodyStart(call)
    // 执行编码器创建请求体
    val rawRequestBody = codec.createRequestBody(request, contentLength)
    return RequestBodySink(rawRequestBody, contentLength)
  }
```

### 写请求体
#### RequestBodySink.write
根据创建的RequestBodySink进行写入操作
Http/1.x ，newKnownLengthSink或者newChunkedSink

```
    override fun write(source: Buffer, byteCount: Long) {
      check(!closed) { "closed" }
      if (contentLength != -1L && bytesReceived + byteCount > contentLength) {
        throw ProtocolException(
            "expected $contentLength bytes but received ${bytesReceived + byteCount}")
      }
      try {
        // 执行写入操作 
        super.write(source, byteCount) 
        this.bytesReceived += byteCount
      } catch (e: IOException) {
        throw complete(e)
      }
    }
```

### 完成请求写入
#### Exchange.finishRequest


```
  fun finishRequest() {
    try {
      codec.finishRequest() // 执行请求写入
    } catch (e: IOException) {
      eventListener.requestFailed(call, e)
      trackFailure(e)
      throw e
    }
  }

```

### 读取响应头
#### Exchange.readResponseHeaders


```
  fun readResponseHeaders(expectContinue: Boolean): Response.Builder? {
    try {
      val result = codec.readResponseHeaders(expectContinue)
      result?.initExchange(this)
      return result
    } catch (e: IOException) {
      eventListener.responseFailed(call, e)
      trackFailure(e)
      throw e
    }
  }
```
### 读取响应体
#### Exchange.openResponseBody 
创建响应体封装实例，含有响应体的读取流

```
  fun openResponseBody(response: Response): ResponseBody {
    try {
      // 数据类型
      val contentType = response.header("Content-Type")
      // 数据大小
      val contentLength = codec.reportedContentLength(response)
      // 编码器创建读取原生流
      val rawSource = codec.openResponseBodySource(response)
      // 创建响应体的Source
      val source = ResponseBodySource(rawSource, contentLength)
      return RealResponseBody(contentType, contentLength, source.buffer())
    } catch (e: IOException) {
      eventListener.responseFailed(call, e)
      trackFailure(e)
      throw e
    }
  }
```

### HTTP1.x对应实现
#### Http1ExchangeCodec.writeRequestHeaders
创建RequestLine，执行写入headers方法

```
  override fun writeRequestHeaders(request: Request) {
    // 获取RequestLine实例，进行request输入
    val requestLine = RequestLine.get(
        request, realConnection!!.route().proxy.type())
    // 写入headers
    writeRequest(request.headers, requestLine)
  }

```

#### Http1ExchangeCodec.writeRequest
通过Okio的Sink实例 写入headers

```
  fun writeRequest(headers: Headers, requestLine: String) {
    check(state == STATE_IDLE) { "state: $state" }
    sink.writeUtf8(requestLine).writeUtf8("\r\n")
    // 写入headers
    for (i in 0 until headers.size) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n")
    }
    sink.writeUtf8("\r\n")
    state = STATE_OPEN_REQUEST_BODY
  }
```

#### Http1ExchangeCodec.createRequestBody

```
  override fun createRequestBody(request: Request, contentLength: Long): Sink {
    return when {
      request.body != null && request.body.isDuplex() -> throw ProtocolException(
          "Duplex connections are not supported for HTTP/1")
      request.isChunked() -> newChunkedSink() //  创建未知大小的SInk
      contentLength != -1L -> newKnownLengthSink() // 创建已知大小的Sink
      else -> // Stream a request body of a known length.
        throw IllegalStateException(
            "Cannot stream a request body without chunked encoding or a known content length!")
    }
  }
```

#### Http1ExchangeCodec.finishRequest
请求写入

```
  override fun finishRequest() {
    sink.flush()
  }
```
#### Http1ExchangeCodec.readResponseHeaders
* 读取响应头信息，根据StatusLine解析协议类型、响应码、Message
* 构建ResponseBuilder接受响应头信息

```
override fun readResponseHeaders(expectContinue: Boolean): Response.Builder? {
    check(state == STATE_OPEN_REQUEST_BODY || state == STATE_READ_RESPONSE_HEADERS) {
      "state: $state"
    }

    try {
      // 读取statusLine，响应状态信息
      val statusLine = StatusLine.parse(readHeaderLine())
      // 创建 Response，赋值status信息和header信息
      val responseBuilder = Response.Builder()
          .protocol(statusLine.protocol)
          .code(statusLine.code)
          .message(statusLine.message)
          .headers(readHeaders())
      // 返回responseBuilder
      return when {
        expectContinue && statusLine.code == HTTP_CONTINUE -> {
          null
        }
        statusLine.code == HTTP_CONTINUE -> {
          state = STATE_READ_RESPONSE_HEADERS
          responseBuilder
        }
        else -> {
          state = STATE_OPEN_RESPONSE_BODY
          responseBuilder
        }
      }
    } catch (e: EOFException) {
        ...
   }
  }

```

#### Http1ExchangeCodec.openResponseBodySource
根据响应体chunked特性和大小是否已知创建不同的Source流

```
  override fun openResponseBodySource(response: Response): Source {
    return when {
      !response.promisesBody() -> newFixedLengthSource(0) // 固定长度Source
      response.isChunked() -> newChunkedSource(response.request.url) // ChunkedSource
      else -> {
        val contentLength = response.headersContentLength()
        if (contentLength != -1L) {
          newFixedLengthSource(contentLength)
        } else {
          newUnknownLengthSource() // 未知长度Source
        }
      }
    }
  }
```

### HTTP2对应实现
#### Http2ExchangeCodec.writeRequestHeaders
创建RequestLine，执行写入headers方法

```
  override fun writeRequestHeaders(request: Request) {
    if (stream != null) return

    val hasRequestBody = request.body != null
    // 将request中的Headers 转换为一个含有Header的List集合
    val requestHeaders = http2HeadersList(request) 
    // 创建 本地发起的双向流 （过程会写入header）
    stream = connection.newStream(requestHeaders, hasRequestBody)
    
    ... 
  }

```
#### Http2ExchangeCodec.newStream

```
 private fun newStream(
    associatedStreamId: Int,
    requestHeaders: List<Header>,
    out: Boolean
  ): Http2Stream {
    val outFinished = !out
    val inFinished = false
    val flushHeaders: Boolean
    val stream: Http2Stream
    val streamId: Int

    synchronized(writer) {
      synchronized(this) {
        if (nextStreamId > Int.MAX_VALUE / 2) {
          shutdown(REFUSED_STREAM)
        }
        if (isShutdown) {
          throw ConnectionShutdownException()
        }
        streamId = nextStreamId
        nextStreamId += 2
        // 创建Http2Stream
        stream = Http2Stream(streamId, this, outFinished, inFinished, null)
        
        flushHeaders = !out ||
            writeBytesTotal >= writeBytesMaximum ||
            stream.writeBytesTotal >= stream.writeBytesMaximum
        if (stream.isOpen) {
          streams[streamId] = stream
        }
      }
      if (associatedStreamId == 0) { 
        // 写入headers
        writer.headers(outFinished, streamId, requestHeaders)
      } else {
        require(!client) { "client streams shouldn't have associated stream IDs" }
        // HTTP/2 has a PUSH_PROMISE frame.
        writer.pushPromise(associatedStreamId, streamId, requestHeaders)
      }
    }

    if (flushHeaders) {
      writer.flush()
    }

    return stream
  }
```

#### Http2Writer.headers
* HPACK加密数据
* 写入帧头数据
* 写入加密过的Header数据

```
  fun headers(
    outFinished: Boolean,
    streamId: Int,
    headerBlock: List<Header>
  ) {
    if (closed) throw IOException("closed")
    // 通过Hpack进行headers 的HPACK加密
    hpackWriter.writeHeaders(headerBlock)

    val byteCount = hpackBuffer.size
    val length = minOf(maxFrameSize.toLong(), byteCount)
    var flags = if (byteCount == length) FLAG_END_HEADERS else 0
    if (outFinished) flags = flags or FLAG_END_STREAM
    // 写入帧头
    frameHeader(
        streamId = streamId,
        length = length.toInt(),
        type = TYPE_HEADERS,
        flags = flags
    )
    // 写入加密过的header数据
    sink.write(hpackBuffer, length)

    if (byteCount > length) writeContinuationFrames(streamId, byteCount - length)
  }

```


#### Http2ExchangeCodec.createRequestBody
返回一个 FramingSink 对象 ，RequestBodySink.writeTo(bufferedRequestBody)最后会调用其成员变量delegate即sink对象的write方法

```
  override fun createRequestBody(request: Request, contentLength: Long): Sink {
    return stream!!.getSink() // FramingSink
  }
```

#### FramingSink.write
写入帧数据

```
    override fun write(source: Buffer, byteCount: Long) {
      this@Http2Stream.assertThreadDoesntHoldLock()
      // buffer写入source数据
      sendBuffer.write(source, byteCount)
      while (sendBuffer.size >= EMIT_BUFFER_SIZE) {
        emitFrame(false) // 当buffer数据大于EMIT_BUFFER_SIZE 则执行发送frame数据
      }
    }
```

#### FramingSink.emitFrame
> Emit a single data frame to the connection. The frame's size be limited by this stream's write window. This method will block until the write window is nonempty.

往连接中写入Frame数据，当写入的数据超过了写入的最大值就阻塞直到唤起
```
    private fun emitFrame(outFinishedOnLastFrame: Boolean) {
      val toWrite: Long
      val outFinished: Boolean
      synchronized(this@Http2Stream) {
        writeTimeout.enter()
        try {
          // 当写入的总数据大小 大于 最大写入值的时候 阻塞
          while (writeBytesTotal >= writeBytesMaximum &&
              !finished &&
              !closed &&
              errorCode == null) {
            waitForIo() // 阻塞直到 接受  WINDOW_UPDATE 
          }
        } finally {
          writeTimeout.exitAndThrowIfTimedOut()
        }

        checkOutNotClosed() // 
        toWrite = minOf(writeBytesMaximum - writeBytesTotal, sendBuffer.size)
        writeBytesTotal += toWrite
        outFinished = outFinishedOnLastFrame && toWrite == sendBuffer.size && errorCode == null
      }

      writeTimeout.enter()
      try {
        // 写入数据
        connection.writeData(id, outFinished, sendBuffer, toWrite)
      } finally {
        writeTimeout.exitAndThrowIfTimedOut()
      }
    }
```

#### Http2Connection.writeData
连接写入数据，当写入的数据超过了写入的最大值就阻塞直到唤起
writeBytesMaximum = 16384（每个帧限制大小）

```
fun writeData(
    streamId: Int,
    outFinished: Boolean,
    buffer: Buffer?,
    byteCount: Long
  ) {
    var byteCount = byteCount
    while (byteCount > 0L) {
      var toWrite: Int
      synchronized(this@Http2Connection) {
        try {
        
          while (writeBytesTotal >= writeBytesMaximum) { // 待写入数据超了
            if (!streams.containsKey(streamId)) {
              throw IOException("stream closed")
            }
            this@Http2Connection.wait() // 阻塞
          }
        } catch (e: InterruptedException) {
          Thread.currentThread().interrupt() // Retain interrupted status.
          throw InterruptedIOException()
        }

        toWrite = minOf(byteCount, writeBytesMaximum - writeBytesTotal).toInt()
        toWrite = minOf(toWrite, writer.maxDataLength())
        writeBytesTotal += toWrite.toLong()
      }

      byteCount -= toWrite.toLong()
      //写入数据 
      writer.data(outFinished && byteCount == 0L, streamId, buffer, toWrite)
    }
  }

```
#### Http2Write.data
数据帧的写入执行方法

```
  fun data(outFinished: Boolean, streamId: Int, source: Buffer?, byteCount: Int) {
    if (closed) throw IOException("closed")
    var flags = FLAG_NONE
    if (outFinished) flags = flags or FLAG_END_STREAM
    // 帧数据写入
    dataFrame(streamId, flags, source, byteCount)
  }

  @Throws(IOException::class)
  fun dataFrame(streamId: Int, flags: Int, buffer: Buffer?, byteCount: Int) {
    // 写入帧头
    frameHeader(
        streamId = streamId,
        length = byteCount,
        type = TYPE_DATA,
        flags = flags
    )
    // 写入数据
    if (byteCount > 0) {
      sink.write(buffer!!, byteCount.toLong())
    }
  }
```



#### Http2ExchangeCodec.readResponseHeaders
* 阻塞等待获取Headers数据 
* 获取Headers数据，转换为Response中headers，以及其它响应头信息，创建ResponseBuilder

```
  override fun readResponseHeaders(expectContinue: Boolean): Response.Builder? {
    val headers = stream!!.takeHeaders() // 取headers
    val responseBuilder = readHttp2HeadersList(headers, protocol) // 读取到headers集合并创建建响应Buidler
    return if (expectContinue && responseBuilder.code == HTTP_CONTINUE) {
      null
    } else {
      responseBuilder
    }
  }
```

#### Http2Stream.takeHeaders
阻塞等待headers数据，在拿到headers后，返回数据
headers数据在receiveHeaders方法中会添加

```
  fun takeHeaders(): Headers {
    readTimeout.enter()
    try {
      while (headersQueue.isEmpty() && errorCode == null) {
        waitForIo() // 如果为空，则阻塞等待 
      }
    } finally {
      readTimeout.exitAndThrowIfTimedOut()
    }
    if (headersQueue.isNotEmpty()) {
      return headersQueue.removeFirst() //取headers 返回
    }
    throw errorException ?: StreamResetException(errorCode!!)
  }

```

#### Http2Stream.receiveHeaders
从数据源接受header后，存储到headers队列中


```
  fun receiveHeaders(headers: Headers, inFinished: Boolean) {
    this@Http2Stream.assertThreadDoesntHoldLock()

    val open: Boolean
    synchronized(this) {
      if (!hasResponseHeaders || !inFinished) {
        hasResponseHeaders = true
        headersQueue += headers // 加入headers
      } else {
        this.source.trailers = headers
      }
      if (inFinished) {
        this.source.finished = true
      }
      open = isOpen
      notifyAll() 
    }
    if (!open) {
      connection.removeStream(id)
    }
  }

```

#### ReaderRunnable.run
在Http2Connection.start的时候，最后会执行一个此线程方法
当读取完preface之后，就一直循环读取下一个帧数据

```
    override fun run() {
      var connectionErrorCode = ErrorCode.INTERNAL_ERROR
      var streamErrorCode = ErrorCode.INTERNAL_ERROR
      var errorException: IOException? = null
      try {
        reader.readConnectionPreface(this) 
        // 循环读取帧数据
        while (reader.nextFrame(false, this)) {
        }
        connectionErrorCode = ErrorCode.NO_ERROR
        streamErrorCode = ErrorCode.CANCEL
      } 
      
    }
```

#### Http2Reader.nextFrame
读取响应数据

```
  fun nextFrame(requireSettings: Boolean, handler: Handler): Boolean {
    try {
      source.require(9) // Frame header size.
    } catch (e: EOFException) {
      return false // This might be a normal socket close.
    }

    val length = source.readMedium()
    if (length > INITIAL_MAX_FRAME_SIZE) {
      throw IOException("FRAME_SIZE_ERROR: $length")
    }
    val type = source.readByte() and 0xff
    if (requireSettings && type != TYPE_SETTINGS) {
      throw IOException("Expected a SETTINGS frame but was $type")
    }
    val flags = source.readByte() and 0xff
    val streamId = source.readInt() and 0x7fffffff // Ignore reserved bit.
    if (logger.isLoggable(FINE)) logger.fine(frameLog(true, streamId, length, type, flags))
    // 读取不同类型数据
    when (type) {
      TYPE_DATA -> readData(handler, length, flags, streamId)
      TYPE_HEADERS -> readHeaders(handler, length, flags, streamId)
      TYPE_PRIORITY -> readPriority(handler, length, flags, streamId)
      TYPE_RST_STREAM -> readRstStream(handler, length, flags, streamId)
      TYPE_SETTINGS -> readSettings(handler, length, flags, streamId)
      TYPE_PUSH_PROMISE -> readPushPromise(handler, length, flags, streamId)
      TYPE_PING -> readPing(handler, length, flags, streamId)
      TYPE_GOAWAY -> readGoAway(handler, length, flags, streamId)
      TYPE_WINDOW_UPDATE -> readWindowUpdate(handler, length, flags, streamId)
      else -> source.skip(length.toLong()) // Implementations MUST discard frames of unknown types.
    }

    return true
  }
```

```
```


#### FramingSource.read

```
override fun read(sink: Buffer, byteCount: Long): Long {
      require(byteCount >= 0L) { "byteCount < 0: $byteCount" }

      while (true) {
        var tryAgain = false
        var readBytesDelivered = -1L
        var errorExceptionToDeliver: IOException? = null

        // 1. Decide what to do in a synchronized block.

        synchronized(this@Http2Stream) {
          readTimeout.enter()
          try {
            if (errorCode != null) {
              // Prepare to deliver an error.
              errorExceptionToDeliver = errorException ?: StreamResetException(errorCode!!)
            }

            if (closed) {
              throw IOException("stream closed")
            } else if (readBuffer.size > 0L) {
              // Prepare to read bytes. Start by moving them to the caller's buffer.
              readBytesDelivered = readBuffer.read(sink, minOf(byteCount, readBuffer.size))
              readBytesTotal += readBytesDelivered

              val unacknowledgedBytesRead = readBytesTotal - readBytesAcknowledged
              if (errorExceptionToDeliver == null &&
                  unacknowledgedBytesRead >= connection.okHttpSettings.initialWindowSize / 2) {
                // Flow control: notify the peer that we're ready for more data! Only send a
                // WINDOW_UPDATE if the stream isn't in error.
                connection.writeWindowUpdateLater(id, unacknowledgedBytesRead)
                readBytesAcknowledged = readBytesTotal
              }
            } else if (!finished && errorExceptionToDeliver == null) {
              // Nothing to do. Wait until that changes then try again.
              waitForIo()
              tryAgain = true
            }
          } finally {
            readTimeout.exitAndThrowIfTimedOut()
          }
        }

        // 2. Do it outside of the synchronized block and timeout.

        if (tryAgain) {
          continue
        }

        if (readBytesDelivered != -1L) {
          // Update connection.unacknowledgedBytesRead outside the synchronized block.
          updateConnectionFlowControl(readBytesDelivered)
          return readBytesDelivered
        }

        ...

        return -1L // This source is exhausted.
      }
    }

```
[下一篇 OkHttp 4源码（6）— Okio源码解析](https://www.jianshu.com/p/7b7ba4333c5e)


