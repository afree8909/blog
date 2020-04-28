---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143252.png
tags: 
- OkHttp
categories:
- [Android, 框架]
---


> 本文基于Okio 2.4.3源码分析
> [Okio - 官方地址](https://square.github.io/okio/)
> [Okio - GitHub代码地址](https://github.com/square/okio)

## Okio 介绍

### Okio是什么

Okio来源Square公司，它是对java.io和java.nio的进一步封装实现，使得更容易处理、访问、缓存数据。它最初是作为OkHttp网络库的组件

### Okio 特点

#### Buffer and ByteString

目标：更好的CPU和Memory综合表现

*   Buffer：通过双向链表的Segment缓存结构，当从一个Buffer转移数据到另一个Buffer的时候，提供重新分配拥有权达到无需拷贝，相比一次深拷贝，效率大大增加。
*   ByteString：Encode UTF-8 String到byteString过程会缓存原string，Decode过程中则可以直接使用

#### Source and Sink

*   支持超时机制
*   非常轻便，便于实现、使用、测试

### Okio 图文概括
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143246.png)
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143252.png)




## 源码分析

### 测试代码示例

```
public final class OkioTest {
  @Rule public TemporaryFolder temporaryFolder = new TemporaryFolder();

  @Test public void readWriteFile() throws Exception {
    File file = temporaryFolder.newFile();
    // 写
    BufferedSink sink = Okio.buffer(Okio.sink(file));
    sink.writeUtf8("Hello, java.io file!");
    sink.close();
    assertTrue(file.exists());
    assertEquals(20, file.length());
    // 读
    BufferedSource source = Okio.buffer(Okio.source(file));
    assertEquals("Hello, java.io file!", source.readUtf8());
    source.close();
  }
}

```

### 数据写入 Sink

*   实现类：OutputStreamSink
*   实现原理：依旧是借助OutputStream进行写入操作
*   写入流程
    1.  超时判断
    2.  根据入参的所需写入数据大小，循环写入数据
    3.  取Buffer中第一个Segment，计算可读取的数据，写入到目标输出流中
    4.  每次循环写入的数据大小=min（剩余要写入数据大小，此次循环取到的Segment中的可读取数据大小），直至完成目标数据大小的写入
    5.  每次写入完毕，对Segment进行移除和回收

```
// 返回一个Sink
fun File.sink(append: Boolean = false): Sink = FileOutputStream(this, append).sink()

fun OutputStream.sink(): Sink = OutputStreamSink(this, Timeout())

// Sink实现类
private class OutputStreamSink(
  private val out: OutputStream, // java底层输出流
  private val timeout: Timeout // 超时机制
) : Sink {
  //  buffer写入到输出流
  override fun write(source: Buffer, byteCount: Long) {
    checkOffsetAndCount(source.size, 0, byteCount)
    var remaining = byteCount
    // 循环读取 所需读取数据大小
    while (remaining > 0) {
      timeout.throwIfReached() // 是否超时
      // Buffer的缓存数据是由 Segment双向链表数据结构缓存
      // 取Buffer的第一个Segment（缓存片段）
      val head = source.head!!
      // segment中limit-pos即数据大小，这里取两者最小的那个数据大小
      val toCopy = minOf(remaining, head.limit - head.pos).toInt()
      // 将目标读取数据 写入到 OutputStream中
      out.write(head.data, head.pos, toCopy)
      // segment 读指针迁移 写入数据大小
      head.pos += toCopy
      // 读取数据大小减少 写入数据大小
      remaining -= toCopy
      // source即buffer中数据大小减少 写入数据大小
      source.size -= toCopy
      // 如果读数据指针等于写数据指针，证明segment无有效数据，则脸表移除该Segment，并执行Segment回收方法
      if (head.pos == head.limit) {
        source.head = head.pop()
        SegmentPool.recycle(head)
      }
    }
  }
  // flush 执行OutputStream的flush方法
  override fun flush() = out.flush()
  // close 执行OutputStream的close方法
  override fun close() = out.close()
  // 超时机制
  override fun timeout() = timeout

  override fun toString() = "sink($out)"
}

```

### 数据读取 Source

*   实现类：InputStreamSource
*   实现原理：依旧是借助InputStream进行写入操作
*   读取流程
    1.  超时判断，目标读取数据大小合法判断
    2.  获取Buffer的尾部Segment，并计算该Segment可写入的数据最大值 ，与目标读取数据大小值进行比较，取其中小的一个
    3.  读取数据到Buffer中
    4.  返回读取数据大小

```
// 返回File的一个Source
fun File.source(): Source = inputStream().source()

// 创建InputStreamSource作为Source返回
fun InputStream.source(): Source = InputStreamSource(this, Timeout())

// InputStreamSource实现类
private class InputStreamSource(
  private val input: InputStream,
  private val timeout: Timeout
) : Source {
  // 核心read方法实现
  override fun read(sink: Buffer, byteCount: Long): Long {
    if (byteCount == 0L) return 0
    require(byteCount >= 0) { "byteCount < 0: $byteCount" }
    try {
      // 是否超时
      timeout.throwIfReached()
      // 获取一个可写入数据的 Segment
      val tail = sink.writableSegment(1)
      // 最大可写入到Buffer中的大小（一个Segment数据最大大小 减去 该Segment已经写入的数据大小）
      val maxToCopy = minOf(byteCount, Segment.SIZE - tail.limit).toInt()
      // 从目标读取流中 ，读取制定大小数据到Segment中
      val bytesRead = input.read(tail.data, tail.limit, maxToCopy)
      // 读取大小-1异常情况处理
      if (bytesRead == -1) {
        if (tail.pos == tail.limit) {
          // 没有数据，移除Segment并回收
          sink.head = tail.pop()
          SegmentPool.recycle(tail)
        }
        return -1
      }
      // Segment的limit增加读取的数据大小
      tail.limit += bytesRead
      // buffer数据大小增加 读取的数据大小
      sink.size += bytesRead
      // 返回本次读取数据的大小
      return bytesRead.toLong() 
    } catch (e: AssertionError) {
      if (e.isAndroidGetsocknameError) throw IOException(e)
      throw e
    }
  }

  override fun close() = input.close()

  override fun timeout() = timeout

  override fun toString() = "source($input)"
}

```

### 数据缓存 Buffer

#### 写入缓存

##### BufferedSink 接口

```
actual interface BufferedSink : Sink, WritableByteChannel {
  // Buffer
  actual val buffer: Buffer

  // 一系列 write方法
  actual fun writeXXX(...): BufferedSink

  // 将Buffer中数据写入到Sink中
  actual fun emitCompleteSegments(): BufferedSink

}

```

##### RealBufferedSink 实现类

*   BufferedSink实现类，含有两个重要成员Sink和Buffer
*   接口方法基本上都经过 internal/RealBufferedSink一层封装，方法实现皆借助Buffer进行写入操作

```
internal actual class RealBufferedSink actual constructor(
  @JvmField actual val sink: Sink // 目标写入Sink
) : BufferedSink {

  @JvmField val bufferField = Buffer() // 创建Buffer，buffer逻辑实现核心类

  @JvmField actual var closed: Boolean = false // 是否关闭

  @Suppress("OVERRIDE_BY_INLINE") // 重载父类buffer getter方法
   override val buffer: Buffer
    inline get() = bufferField

  // 一系列 write方法 。实现逻辑都是通过internal/RealBufferSink进行实现，里面都是通过Buffer进行写操作
  override fun writeAll(source: Source) = commonWriteAll(source)
  override fun write(source: Source, byteCount: Long): BufferedSink = commonWrite(source, byteCount)
  override fun writeByte(b: Int) = commonWriteByte(b)
  override fun writeShort(s: Int) = commonWriteShort(s)
  override fun writeShortLe(s: Int) = commonWriteShortLe(s)
  override fun writeInt(i: Int) = commonWriteInt(i)
  override fun writeIntLe(i: Int) = commonWriteIntLe(i)
  override fun writeLong(v: Long) = commonWriteLong(v)
  override fun writeLongLe(v: Long) = commonWriteLongLe(v)
  override fun writeDecimalLong(v: Long) = commonWriteDecimalLong(v)
  override fun writeHexadecimalUnsignedLong(v: Long) = commonWriteHexadecimalUnsignedLong(v)
  override fun emitCompleteSegments() = commonEmitCompleteSegments()
  override fun emit() = commonEmit()
  ...

}

```

#### 读取缓存

##### BufferedSource 接口

```
actual interface BufferedSource : Source, ReadableByteChannel {
  // Buffer
  actual val buffer: Buffer

  // 一系列 read方法
  actual fun readXXX(...): XXX

}

```

##### RealBufferedSource 实现类

*   BufferedSource实现类，含有两个重要成员Source和Buffer
*   接口方法基本上都经过 internal/RealBufferedSource一层封装，方法实现皆借助Buffer进行读取操作

```
internal actual class RealBufferedSource actual constructor(
  @JvmField actual val source: Source
) : BufferedSource {
  @JvmField val bufferField = Buffer() // 创建一个Buffer
  @JvmField actual var closed: Boolean = false

  @Suppress("OVERRIDE_BY_INLINE") // 重载父类buffer getter方法
  override val buffer: Buffer
    inline get() = bufferField

  // 一系列 read方法 。实现逻辑都是通过internal/RealBufferSource进行实现，里面都是通过Buffer进行写操作
  override fun readByte(): Byte = commonReadByte()
  override fun readByteString(): ByteString = commonReadByteString()
  override fun readByteString(byteCount: Long): ByteString = commonReadByteString(byteCount)
  override fun readFully(sink: Buffer, byteCount: Long): Unit = commonReadFully(sink, byteCount)
  override fun readAll(sink: Sink): Long = commonReadAll(sink)
  override fun readUtf8(): String = commonReadUtf8()
  override fun readUtf8(byteCount: Long): String = commonReadUtf8(byteCount)
  ...

}

```

#### 缓存实现类 Buffer

缓存逻辑实现核心类，对RealBufferedSource和RealBufferedSink中暴露的Api进行了实现，重点看下Buffer的读写方法，其核心设计原则就是兼顾CPU（时间）和Memory（空间）

*   每一个Buffer中，包含Segment双向链表结构，进行缓存数据存储，多个固定的Segment有助于避免内存的申请和回收，减少内存片段
*   Buffer既充当了缓存写入角色，又充当了缓存读取角色，更有利于实现buffer平滑实现一次拷贝
*   读Buffer数据转移到写Buffer，1\. 如果写Buffer可以容纳，则直接拷贝存储；2.如果写Buffer不可以容纳，则通过split方法在来一个Segment进行存储

Buffer、Segment、SegmentPool巧妙的设计，兼顾了CPU和Memory的平衡

```
internal inline fun Buffer.commonRead(sink: Buffer, byteCount: Long): Long {
  var byteCount = byteCount
  require(byteCount >= 0) { "byteCount < 0: $byteCount" }

  if (size == 0L) return -1L
  if (byteCount > size) byteCount = size
  // 执行 Buffer的write方法
  sink.write(this, byteCount)
  return byteCount
}

internal inline fun Buffer.commonWrite(source: Buffer, byteCount: Long) {
  var byteCount = byteCount // 写入大小
  // 逻辑判断
  require(source !== this) { "source == this" }
  checkOffsetAndCount(source.size, 0, byteCount)

  while (byteCount > 0L) {
    // source对应的Segment数据大小是否 大于 目标写入大小，大于则只写入一部分数据
    if (byteCount < source.head!!.limit - source.head!!.pos) {
      // 取尾部Segment
      val tail = if (head != null) head!!.prev else null
      // 尾部不为空，且拥有数据写入权 且 拥有足够写入的空间
      if (tail != null && tail.owner &&
        byteCount + tail.limit - (if (tail.shared) 0 else tail.pos) <= Segment.SIZE) {
        // 将读取Buffer的数据写入到 本Buffer的尾部Segment中
        // writeTo方法 可能是深拷贝或者是浅拷贝，后面有分析
        source.head!!.writeTo(tail, byteCount.toInt())
        // 重置source的size 和本Buffer的size
        source.size -= byteCount
        size += byteCount
        return
      } else {
        // 装不下读取的数据，则需要新的Segment来写入 ，新的segement可能通过shareCopy或者SegemntPool来（见Segment分析）
        source.head = source.head!!.split(byteCount.toInt())
      }
    }

    // Remove the source's head segment and append it to our tail.
    // source对应Buffer中的head Segment数据读取完毕，对Buffer中的Segment进行移除
    val segmentToMove = source.head
    val movedByteCount = (segmentToMove!!.limit - segmentToMove.pos).toLong()
    source.head = segmentToMove.pop()
    // 移除后，因为它有可能含有共享的数据（见Segment），所以将其加入本Buffer的tail
    // 从而巧妙实现 Buffer 读写的复用
    if (head == null) { // head为空，直接赋值head
      head = segmentToMove
      segmentToMove.prev = segmentToMove
      segmentToMove.next = segmentToMove.prev
    } else {
      // head不为空，赋值到尾部
      var tail = head!!.prev
      tail = tail!!.push(segmentToMove)
      // 新的tail加入，确认是否需要压缩
      tail.compact()
    }
    // 重置source大小，和本Buffer的size大小
    source.size -= movedByteCount
    size += movedByteCount
    byteCount -= movedByteCount
  }
}

```

### 缓存数据片段 Segment

Buffer数据存储片段，目标数据都存储在Segment中的data字段中，Segment双向链表结构，提供pop、push进行Segment的增加和移除。split和compact对Segment进行拆分和合并，shareCopy为共享数据提供一次拷贝便利等

#### 共享机制

两个字段（shared、owner）和两个方法（sharedCopy、unsharedCopy）来实现
浅拷贝的应用会直接减少一次I/O操作，大大提高I/O效率

*   shared：代表该Segment的数据是否共享
*   owner：代表该Segment是否拥有对数据的写入权利
*   sharedCopy：data浅拷贝，共享复制的Segment中的data数据，其中owner为false（即拷贝的Segment只能读不能写）
*   unsharedCopy：data深拷贝，完全复制一个Segment

#### 拆分与合并

为了Segment的回收，以及更加合理化存储数据，提供两个方法

*   compact：合并，如果本Segment与前一个Segment两者数据可以合为一个，则可以通过compact方法合并为一个Segment，然后回收本Segment
*   split：分割，当存储的数据大于当前Segment的容量时，则需要一个新的Segment，此时会有两种情况，1.如果数据量大于1024则浅拷贝一个Segment来装数据，如果不是，则直接深拷贝数据创建一个新的Segment

```
internal class Segment {
  // 缓存的数据
  @JvmField val data: ByteArray

  /** 读取数据的指针  */
  @JvmField var pos: Int = 0

  /** 写入数据的的指针 */
  @JvmField var limit: Int = 0

  /** 数据是否共享  */
  @JvmField var shared: Boolean = false

  /** limit扩展字段，数据是否属于当前的Segment，true即可以写入 */
  @JvmField var owner: Boolean = false

  /** 链表数据结构的next字段 */
  @JvmField var next: Segment? = null

  /** 链表数据结构的prev字段  */
  @JvmField var prev: Segment? = null

  constructor() {
    this.data = ByteArray(SIZE)
    this.owner = true
    this.shared = false
  }

  constructor(data: ByteArray, pos: Int, limit: Int, shared: Boolean, owner: Boolean) {
    this.data = data
    this.pos = pos
    this.limit = limit
    this.shared = shared
    this.owner = owner
  }

  // data浅拷贝，创建一个共享的Segment，owner为false
  fun sharedCopy(): Segment {
    shared = true
    return Segment(data, pos, limit, true, false)
  }

  // data深拷贝，创建一个非共享的Segment，
  fun unsharedCopy() = Segment(data.copyOf(), pos, limit, false, true)

  // 移除本Segment对象
  fun pop(): Segment? {
    val result = if (next !== this) next else null
    prev!!.next = next
    next!!.prev = prev
    next = null
    prev = null
    return result
  }

  // 添加一个segment到链表中
  fun push(segment: Segment): Segment {
    segment.prev = this
    segment.next = next
    next!!.prev = segment
    next = segment
    return segment
  }

  // 分割为两个Segment，数据起始点分别为 `[pos..pos+byteCount)`、`[pos+byteCount..limit)`
  fun split(byteCount: Int): Segment {
    require(byteCount > 0 && byteCount <= limit - pos) { "byteCount out of range" }
    val prefix: Segment

    // 1\. 尽量减少二次拷贝；2\. 共享拷贝在一定大小才执行，避免过多的短小的Segment
    if (byteCount >= SHARE_MINIMUM) {
      prefix = sharedCopy() // 共享拷贝
    } else {
      prefix = SegmentPool.take() // 直接拷贝
      data.copyInto(prefix.data, startIndex = pos, endIndex = pos + byteCount)
    }
    // prefix赋值 limit pos
    prefix.limit = prefix.pos + byteCount
    pos += byteCount
    prev!!.push(prefix)
    return prefix
  }

  // 合并压缩
  fun compact() {
    check(prev !== this) { "cannot compact" }
    if (!prev!!.owner) return 
    // 本Segment数据大小
    val byteCount = limit - pos
    // 前一个Segment剩余可写入空间大小
    val availableByteCount = SIZE - prev!!.limit + if (prev!!.shared) 0 else prev!!.pos
    if (byteCount > availableByteCount) return     
    // 如果有足够的空间，则将本Segment数据写入到前一个Segment
    writeTo(prev!!, byteCount)
    // pop，然后执行回收
    pop()
    SegmentPool.recycle(this)
  }

  /** 将数据写入到buffer的segment中 */
  fun writeTo(sink: Segment, byteCount: Int) {
    // 只有owner为true才能写入数据
    check(sink.owner) { "only owner can write" }
    // 如果被写入的sink limit+byteCount大于了最大值，则需要先充值下pos和limit
    if (sink.limit + byteCount > SIZE) {
      if (sink.shared) throw IllegalArgumentException()
      // 如果所有空闲都容纳不下 ，则抛出异常
      if (sink.limit + byteCount - sink.pos > SIZE) throw IllegalArgumentException()
      // 将被写入的segment中的数据复制到 [ 0,limit-pos)
      sink.data.copyInto(sink.data, startIndex = sink.pos, endIndex = sink.limit)
      // 重置 pos和limit
      sink.limit -= sink.pos
      sink.pos = 0
    }
    // 将本Segment的data复制到sink的data中，偏移大小为limit，起点为pos
    data.copyInto(sink.data, destinationOffset = sink.limit, startIndex = pos,
        endIndex = pos + byteCount)
    sink.limit += byteCount
    pos += byteCount
  }

  companion object {
    /** Segment 大小 bytes  */
    const val SIZE = 8192

    /** 共享数据大小  */
    const val SHARE_MINIMUM = 1024
  }
}

```

### 缓存片段池 SegmentPool

*   大小：64 * 1024 Kib，一个Segment大小8192，相当于8个Segment
*   结构：单链表结构，提供take（取）和recycle（存）Segment方法，且线程安全
*   作用：防止已申请的资源被回收，增加资源的重复利用，提高效率，减少GC,避免内存抖动

```
internal object SegmentPool {
  // 缓存Segment池总大小 
  const val MAX_SIZE = 64 * 1024L // 64 KiB.

  // 单链表next指针
  var next: Segment? = null

  // segment缓存池使用大小
  var byteCount = 0L

  // 取 segment方法  （线程安全） 
  fun take(): Segment {
    synchronized(this) {
      // 先从链表中获取
      next?.let { result ->
        // 将缓存池next指针后移到下一个next
        next = result.next 
        // 目标segment中next指引为null
        result.next = null
        // 总大小 减去
        byteCount -= Segment.SIZE
        return result
      }
    }
    return Segment() // 创建一个新的segment
  }

  // 存（回收） segment方法
  fun recycle(segment: Segment) {
    require(segment.next == null && segment.prev == null)
    if (segment.shared) return // 共用segment，不可回收使用

    synchronized(this) {
      // 目前segment池大小已经满了，不在回收
      if (byteCount + Segment.SIZE > MAX_SIZE) return // Pool is full.
      // 回收目标segment，增加大小、更换next指引、重置segment
      byteCount += Segment.SIZE
      segment.next = next
      segment.limit = 0
      segment.pos = segment.limit
      next = segment
    }
  }
}

```
[下一篇 OkHttp 4源码（7）— 总结](https://www.jianshu.com/p/c988d0416020)


