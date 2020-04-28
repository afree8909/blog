---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143225.png
tags: 
- OkHttp
categories:
- [Android, 框架]
---

>本文基于OkHttp 4.3.1源码分析
>[OkHttp - 官方地址](https://square.github.io/okhttp/)
[OkHttp - GitHub代码地址](https://github.com/square/okhttp)


## 预备知识

[HTTP缓存原理](https://www.jianshu.com/p/6ec0d13d85bb)

![HTTP缓存流程图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143220.png)

## 概述
OkHttp整体流程（本文覆盖红色部分）

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144516.png)

缓存处理流程
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143225.png)

缓存文件夹
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144602.png)

缓存日志格式
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428144613.png)


## 源码分析

### 测试代码
如果需要缓存机制，那么在构造OkHttpClient的时候需要传入一个Cache实例。
下面是OkHttp提供的一个CacheResponse的用例，除了传入一个Cache构造OkHttpClient，其它完全一样。
OkHttp实现缓存的切入点依旧是拦截器，之前的文章我们知道拦截器处理对象可以接受命令对象并根据自身情况选择处理还是不处理，所以接下来就是直接从CacheInterceptor开始分析

```
public final class CacheResponse {
  private final OkHttpClient client;

  public CacheResponse(File cacheDirectory) throws Exception {
    int cacheSize = 10 * 1024 * 1024; // 10 MiB 大小
    Cache cache = new Cache(cacheDirectory, cacheSize); // 路径
    // 要使用缓存功能，需要在构造OkHttpClient的时候传入 cache （缓存大小，缓存路径地址）
    client = new OkHttpClient.Builder()
        .cache(cache)
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();
    
    String response1Body;
    try (Response response1 = client.newCall(request).execute()) {
        ...
    }

    String response2Body;
    try (Response response2 = client.newCall(request).execute()) {
        ...    
    }
    // 两次请求结果一致
    System.out.println("Response 2 equals Response 1? " + response1Body.equals(response2Body));
  }

  public static void main(String... args) throws Exception {
    new CacheResponse(new File("CacheResponse.tmp")).run();
  }
}

```

### 缓存拦截器逻辑

#### CacheInterceptor.intercept

```
 override fun intercept(chain: Interceptor.Chain): Response {
    // 
    val cacheCandidate = cache?.get(chain.request())

    val now = System.currentTimeMillis()

    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    val networkRequest = strategy.networkRequest
    val cacheResponse = strategy.cacheResponse

    // 缓存追踪，网络请求数、命中缓存数，两者比值可以查看缓存命中率
    cache?.trackResponse(strategy)

    if (cacheCandidate != null && cacheResponse == null) {
      // The cache candidate wasn't applicable. Close it.
      cacheCandidate.body?.closeQuietly()
    }

    // 异常情况
    if (networkRequest == null && cacheResponse == null) {
      return Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(HTTP_GATEWAY_TIMEOUT)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
    }

    // 如果networkRequest为null，则直接使用缓存数据，拦截器处理至此终结，开始响应阶段
    if (networkRequest == null) {
      return cacheResponse!!.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build()
    }

    var networkResponse: Response? = null
    try {
      // 继续执行 拦截器链处理方法，最终发送网络请求，到读取网络数据
      networkResponse = chain.proceed(networkRequest)
    } finally {
      if (networkResponse == null && cacheCandidate != null) {
        cacheCandidate.body?.closeQuietly()
      }
    }

    
    if (cacheResponse != null) {
      if (networkResponse?.code == HTTP_NOT_MODIFIED) { // 304
      // 如果网络数据标志性没有改变，开始返回数据
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build()

        networkResponse.body!!.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        cache.update(cacheResponse, response)
        return response
      } else {
        cacheResponse.body?.closeQuietly()
      }
    }

    val response = networkResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build()

    
    if (cache != null) {
      if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // 对响应数据进行缓存
        val cacheRequest = cache.put(response)
        // 返回一个带有new Source的body读取流 的 Response
        return cacheWritingResponse(cacheRequest, response)
      }
        
      if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
          cache.remove(networkRequest)
        } catch (_: IOException) {
          // The cache cannot be written.
        }
      }
    }

    return response
  }


```

### 缓存获取逻辑


#### Cache.get

* 生成请求对应的唯一标志key
* 通过DiskLruCache获取对应key的缓存快照数据
* 如果有快照，则将快照对应的缓存资源数据通过Okio中流读写转换成内存数据
* 最后返回构造好的Response

```
  internal fun get(request: Request): Response? {
    // 根据请求url字符串进行md5作为缓存对应的key标识
    val key = key(request.url)
    // Cache成员变量DiskLruCache，缓存逻辑实现类
    // snapshot
    val snapshot: DiskLruCache.Snapshot = try {
      // 从DiskLruCache中取缓存
      cache[key] ?: return null
    } catch (_: IOException) {
      return null // Give up because the cache cannot be read.
    }
    // 读取快照中的缓存资源流 ，构造数据Entry
    val entry: Entry = try {
      Entry(snapshot.getSource(ENTRY_METADATA))
    } catch (_: IOException) {
      snapshot.closeQuietly()
      return null
    }
    // 取 response
    val response = entry.response(snapshot)
    if (!entry.matches(request, response)) { 
      response.body?.closeQuietly() // 响应和请求 不匹配则关闭
      return null
    }

    return response // 返回响应数据
  }
```

#### Cache.Entry 构造
缓存数据实例，DiskLruCache中lruEntries存储了请求key对应的快照，快照里有key对应的本地文件读取流，根据读取流读取本地数据，转换输出构造Entry实例，最终应用构造出一个缓存的Response


```
internal constructor(rawSource: Source) {
      try {
        val source = rawSource.buffer()
        // 从缓存文件流中 读取数据 ，写入到Entry中
        ... // 一堆 读取流 写入内存操作
     } finally {
        rawSource.close()
      }
    }
```


#### DiskLruCache.get

* 初始化或者确认初始化（主要目的是将key和对应缓存本地文件标识存储到内存Map中，便于查询）
* 如果找到了就返回一个快照对象，没有则为null
 

```
  operator fun get(key: String): Snapshot? {
    initialize() // 初始化 || 确认已经初始化

    checkNotClosed() // check cache是否关闭
    validateKey(key) // 确认key 符合 写入的规则
    // 找key对应的缓存entry
    val entry = lruEntries[key] ?: return null 
    if (!entry.readable) return null
    // 有缓存entry 取对应的快照
    val snapshot = entry.snapshot() ?: return null
    // 标识一次操作
    redundantOpCount++ 
    // 写入 标记READ 和对应 key 一行
    journalWriter!!.writeUtf8(READ)
        .writeByte(' '.toInt())
        .writeUtf8(key)
        .writeByte('\n'.toInt())
    // 如果操作>2000次了，则需要重新清理 日志 
    if (journalRebuildRequired()) {
      // 执行清理任务
      cleanupQueue.schedule(cleanupTask)
    }
    // 返回快照
    return snapshot
  }
```

#### DiskLruCache.initialize


* 整理日志文件，删除backup或者重命名bakcup日志文件
* 读取日志文件，生成lruEntries内存缓存
* 处理日志文件
* 按需是否重新构建日志文件

```
 // 初始化方法
 fun initialize() {
    this.assertThreadHoldsLock()
    // 是否已经初始化 ，内存缓存
    if (initialized) {
      return // Already initialized.
    }
    // 开始初始化工作
    // 如果存在backup的日志文件，
    // 则看是否存在日志文件，如果存在日志文件则删除backup的，
    // 如果没有则将backup文件转化为日志文件
    if (fileSystem.exists(journalFileBackup)) {
      if (fileSystem.exists(journalFile)) {
        fileSystem.delete(journalFileBackup)
      } else {
        fileSystem.rename(journalFileBackup, journalFile)
      }
    }

    // journal文件存在 开始读取工作
    if (fileSystem.exists(journalFile)) {
      try {
        readJournal()
        processJournal()
        initialized = true
        return
      } catch (journalIsCorrupt: IOException) {
        Platform.get().log(
            "DiskLruCache $directory is corrupt: ${journalIsCorrupt.message}, removing",
            WARN,
            journalIsCorrupt)
      }

      // The cache is corrupted, attempt to delete the contents of the directory. This can throw and
      // we'll let that propagate out as it likely means there is a severe filesystem problem.
      try {
        delete()
      } finally {
        closed = false
      }
    }

    rebuildJournal()

    initialized = true
  }
```

#### DiskLruCache.readJournal

* 读取日志，判断日志是否可用
* 日志可用，则读取每行日志，构建Key、Entry的map到内存缓存
* 如果读取过程中有问题，则根据tmp日志重新构建日志

```
  // 读取日志文件，构造[key,entry]内存缓存集合
  private fun readJournal() {
    fileSystem.source(journalFile).buffer().use { source ->
      // 基于Okio读写本地数据
      // libcore.io.DiskLruCache
      val magic = source.readUtf8LineStrict() 
      val version = source.readUtf8LineStrict() // 版本 1
      val appVersionString = source.readUtf8LineStrict() // app version
      val valueCountString = source.readUtf8LineStrict()// 
      val blank = source.readUtf8LineStrict()

      if (MAGIC != magic ||
          VERSION_1 != version ||
          appVersion.toString() != appVersionString ||
          valueCount.toString() != valueCountString ||
          blank.isNotEmpty()) {
        throw IOException(
            "unexpected journal header: [$magic, $version, $valueCountString, $blank]")
      }

      var lineCount = 0
      while (true) {
        try {
          // 见readJournalLine方法分析（读取key，构造entry）
          readJournalLine(source.readUtf8LineStrict())
          lineCount++
        } catch (_: EOFException) {
          break // End of journal.
        }
      }
      // 无用的line数目
      redundantOpCount = lineCount - lruEntries.size

      // 如果读取有无，则重构建日志
      if (!source.exhausted()) {
        // 通过Okio读取tmp日志文件重新写Journal文件
        rebuildJournal() 
      } else {
        journalWriter = newJournalWriter() // 创建日志writer
      }
    }
  }

```

#### DiskLruCache.readJournalLine

* 读取每一行日志
* 根据行日志属性，CLEAN、DIRTY、REMOVE、READ进行对应的处理
* 缓存到内存缓存LruEntries的Map中

```
  // 读取每一行日志，存储到LruEntries Map中
  private fun readJournalLine(line: String) {
    // 找 第一个空格的 指引
    val firstSpace = line.indexOf(' ')
    if (firstSpace == -1) throw IOException("unexpected journal line: $line")
    // 从第一个空格指引+1位置开始找第二个空格指引
    val keyBegin = firstSpace + 1
    val secondSpace = line.indexOf(' ', keyBegin)
    val key: String
    // 如果第二个空格指引没有找到
    if (secondSpace == -1) {
      // 截取第一个空格指引到字符串最后的字段，截取的字段赋值key
      key = line.substring(keyBegin)
      // 如果第一个空格指引大小为Remove长度或者line是以remove开头
      if (firstSpace == REMOVE.length && line.startsWith(REMOVE)) {
        // lruEntries 移除key
        lruEntries.remove(key)
        return
      }
    } else {
      // 如果第二个空格存在，则key赋值为第一个空格和第二个空格之间的字符串值
      key = line.substring(keyBegin, secondSpace)
    }
    // 取key对应的Entry
    var entry: Entry? = lruEntries[key]
    if (entry == null) {
      // 不存砸，则创建Entry并赋值
      entry = Entry(key)
      lruEntries[key] = entry
    }

    when {
      secondSpace != -1 && firstSpace == CLEAN.length && line.startsWith(CLEAN) -> {
        // 示例数据 CLEAN 4b217e04ba52215f3a6b64d28f6729c6 333 194
        // 读line间隔有多少个 方便存储对应大小数据
        val parts = line.substring(secondSpace + 1)
            .split(' ')
        entry.readable = true // 设置可读
        entry.currentEditor = null // 重置null
        entry.setLengths(parts) // 根据文件个数，设置每个文件大小
      }
        
      // 示例数据：DIRTY 4b217e04ba52215f3a6b64d28f6729c6
      secondSpace == -1 && firstSpace == DIRTY.length && line.startsWith(DIRTY) -> {
        // 赋值currentEditor为对应Entry的Editor
        entry.currentEditor = Editor(entry)
      }
      // READ，不做处理
      secondSpace == -1 && firstSpace == READ.length && line.startsWith(READ) -> {
        // This work was already done by calling lruEntries.get().
      }

      else -> throw IOException("unexpected journal line: $line")
    }
  }

```

#### DiskLruCache.processJournal

* 计算有效文件的总大小值
* 删除无效本地缓存文件和内存缓存key

```
  private fun processJournal() {
    // 删除journal.tmp临时文件
    fileSystem.delete(journalFileTmp)
    val i = lruEntries.values.iterator()
    // 遍历lruEntries ，计算currentEditor为null对应的文件总大小
    while (i.hasNext()) { 
      val entry = i.next()
      if (entry.currentEditor == null) {
        for (t in 0 until valueCount) {
          size += entry.lengths[t]
        }
      } else {
        entry.currentEditor = null
        // 删除currentEditor文件（对应Dirty文件）
        for (t in 0 until valueCount) {
          fileSystem.delete(entry.cleanFiles[t])
          fileSystem.delete(entry.dirtyFiles[t])
        }
        i.remove()
      }
    }
  }
```

### 缓存使用策略逻辑

缓存使用策略：给定请求和缓存响应，决策出用网络响应还是缓存响应，或者两者都有

#### CacheStrategy.init

* 读取请求时间戳、响应时间戳
* 读取缓存策略决策的关键Header，Date、Expires、Last-Modified、ETag

```
init {
      if (cacheResponse != null) {
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis
        val headers = cacheResponse.headers
        for (i in 0 until headers.size) {
          val fieldName = headers.name(i)
          val value = headers.value(i)
          when {
            // 读取缓存策略决策的关键Header
            // Date、Expires、Last-Modified、ETag
            fieldName.equals("Date", ignoreCase = true) -> {
              servedDate = value.toHttpDateOrNull()
              servedDateString = value
            }
            fieldName.equals("Expires", ignoreCase = true) -> {
              expires = value.toHttpDateOrNull()
            }
            fieldName.equals("Last-Modified", ignoreCase = true) -> {
              lastModified = value.toHttpDateOrNull()
              lastModifiedString = value
            }
            fieldName.equals("ETag", ignoreCase = true) -> {
              etag = value
            }
            fieldName.equals("Age", ignoreCase = true) -> {
              ageSeconds = value.toNonNegativeInt(-1)
            }
          }
        }
      }
    }

```

#### CacheStrategy.compute
执行CacheStrategy.computeCandidate策略，最后返回四种策略结果

1. CacheStrategy(request, null)； 无缓存，需要取网络数据
2. CacheStrategy(null, cacheResponse)；直接使用缓存
3. CacheStrategy(conditionalRequestHeaders,cacheResponse)；构建新的网络请求（加Header）需要新的网络请求返回后判断是否使用哪一个
4. CacheStrategy(null, null)； 错误场景

```
    fun compute(): CacheStrategy {
      val candidate = computeCandidate()

      // 禁止构建新的请求 且 缓存数据有没有，则返回null，这一般是个错误场景
      if (candidate.networkRequest != null && request.cacheControl.onlyIfCached) {
        return CacheStrategy(null, null)
      }

      return candidate
    }
```

#### CacheStrategy.computeCandidate
缓存策略一系列判断，最后有三种策略返回结果

1. CacheStrategy(request, null)； 无缓存，需要取网络数据
2. CacheStrategy(null, cacheResponse)；直接使用缓存
3. CacheStrategy(conditionalRequestHeaders,cacheResponse)；构建新的网络请求（加Header）需要新的网络请求返回后判断是否使用哪一个


```
  private fun computeCandidate(): CacheStrategy {
      // 无缓存
      if (cacheResponse == null) {
        return CacheStrategy(request, null)
      }

      // https请求 且无handshake数据
      if (request.isHttps && cacheResponse.handshake == null) {
        return CacheStrategy(request, null)
      }

      // 不支持缓存
      if (!isCacheable(cacheResponse, request)) {
        return CacheStrategy(request, null)
      }
        
      // 配置了cacheControl字段 说明不需要cache
      val requestCaching = request.cacheControl
      if (requestCaching.noCache || hasConditions(request)) {
        return CacheStrategy(request, null)
      }
      // 计算!noCache情况Header下，是否过期，是否可以直接使用Cache数据
      val responseCaching = cacheResponse.cacheControl

      val ageMillis = cacheResponseAge()
      var freshMillis = computeFreshnessLifetime()

      if (requestCaching.maxAgeSeconds != -1) {
        freshMillis = minOf(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds.toLong()))
      }

      var minFreshMillis: Long = 0
      if (requestCaching.minFreshSeconds != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds.toLong())
      }

      var maxStaleMillis: Long = 0
      if (!responseCaching.mustRevalidate && requestCaching.maxStaleSeconds != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds.toLong())
      }
      // !noCache Header标志，且缓存数据未过期，则直接使用
      if (!responseCaching.noCache && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        val builder = cacheResponse.newBuilder()
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"")
        }
        val oneDayMillis = 24 * 60 * 60 * 1000L
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"")
        }
        return CacheStrategy(null, builder.build())
      }

      // 将header数据取出，带入新的请求header中
      val conditionName: String
      val conditionValue: String?
      when {
        etag != null -> {
          conditionName = "If-None-Match"
          conditionValue = etag
        }

        lastModified != null -> {
          conditionName = "If-Modified-Since"
          conditionValue = lastModifiedString
        }

        servedDate != null -> {
          conditionName = "If-Modified-Since"
          conditionValue = servedDateString
        }

        else -> return CacheStrategy(request, null) // No condition! Make a regular request.
      }
      // 构建新的网络请求，返回带有新的网络请求+cacheReponse数据
      val conditionalRequestHeaders = request.headers.newBuilder()
      conditionalRequestHeaders.addLenient(conditionName, conditionValue!!)

      val conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build()
      return CacheStrategy(conditionalRequest, cacheResponse)
    }
```

### 缓存存储逻辑

#### Cache.put
⚠️ 对于非GET请求，不做缓存逻辑，原因：POST请求虽然可以做到缓存逻辑，但是实现复杂度和收益比非常低，所以没有处理非Get的请求缓存




```
 internal fun put(response: Response): CacheRequest? {
    // 请求方法
    val requestMethod = response.request.method
    // 如果是非Get则为true
    if (HttpMethod.invalidatesCache(response.request.method)) {
      try {
        remove(response.request) // 移除缓存
      } catch (_: IOException) {
        // The cache cannot be written.
      }
      return null
    }
    // 对于非GET请求，不做缓存逻辑，原因：POST请求虽然可以做到缓存逻辑，但是实现复杂度和收益比非常低，所以没有做处理
    if (requestMethod != "GET") {
      // Don't cache non-GET responses. We're technically allowed to cache HEAD requests and some
      // POST requests, but the complexity of doing so is high and the benefit is low.
      return null
    }

    if (response.hasVaryAll()) {
      return null
    }
    // 构造 Entry
    val entry = Entry(response)
    var editor: DiskLruCache.Editor? = null
    try {
      // Entry 编辑器，数据书写
      editor = cache.edit(key(response.request.url)) ?: return null
      entry.writeTo(editor)
      // 构造一个 RealCacheRequest返回 
      return RealCacheRequest(editor)
    } catch (_: IOException) {
      abortQuietly(editor)
      return null
    }
  }

```

#### DiskLruCache.edit
主要是获取编辑器，编辑器获取的同时，DiskLruCache内存缓存中会存储对应的key和Entry，entry会指定对应的Editor，且会写入一条数据到日志中，目前日志key对应的数据是DIRTY

```
 fun edit(key: String, expectedSequenceNumber: Long = ANY_SEQUENCE_NUMBER): Editor? {
    initialize()

    checkNotClosed()
    validateKey(key)
    // key对应Entry
    var entry: Entry? = lruEntries[key] 
    if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER &&
        (entry == null || entry.sequenceNumber != expectedSequenceNumber)) {
      return null // Snapshot is stale.
    }
    // 是否正在编辑
    if (entry?.currentEditor != null) {
      return null // Another edit is in progress.
    }
    // 是否需要执行clean
    if (mostRecentTrimFailed || mostRecentRebuildFailed) {
      cleanupQueue.schedule(cleanupTask)
      return null
    }

    // 写入 key 到日志中 ，目前对应是 DIRTY
    val journalWriter = this.journalWriter!!
    journalWriter.writeUtf8(DIRTY)
        .writeByte(' '.toInt())
        .writeUtf8(key)
        .writeByte('\n'.toInt())
    journalWriter.flush()

    if (hasJournalErrors) {
      return null // Don't edit; the journal can't be written.
    }
    // 构造Entry，存储到map中
    if (entry == null) {
      entry = Entry(key)
      lruEntries[key] = entry
    }
    // 构造编辑器，及Etry赋值当前编辑器
    val editor = Editor(entry)
    entry.currentEditor = editor
    return editor
  }


```

#### Cache.writeTo
通过Okio将响应数据写入到本地缓存

```
fun writeTo(editor: DiskLruCache.Editor) {
      val sink = editor.newSink(ENTRY_METADATA).buffer()
      sink.writeUtf8(url).writeByte('\n'.toInt())
      sink.writeUtf8(requestMethod).writeByte('\n'.toInt())
      sink.writeDecimalLong(varyHeaders.size.toLong()).writeByte('\n'.toInt())
      for (i in 0 until varyHeaders.size) {
        sink.writeUtf8(varyHeaders.name(i))
            .writeUtf8(": ")
            .writeUtf8(varyHeaders.value(i))
            .writeByte('\n'.toInt())
      }
      // 写入数据
      ...
      sink.close()
    }

```
[下一篇 OkHttp 4源码（4）— 连接机制分析](https://www.jianshu.com/p/be6d09f2656b)


