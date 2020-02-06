
---
tags: 
- 源码
categories:
- [Java, 系统]
---

### SoftReference和WeakReference
我们知道这两个类基本功能相似，区别在于当引用对象为空的时候，WeakRefreence对象会在GC回收的时候被回收，而SoftReference则需要更苛刻的条件

#### SoftReference实现原理
总结：GC回收时计算 SoftReference存活时间与剩余内存换算得出的时间进行大小比较（内存剩余空间*设定系数仅过换算规则计算得出），如果大于测认为需要回收，反之亦然

```
/// hotspot/src/share/vm/memory/referenceProcessor.cpp
if (rt == REF_SOFT) {
    // 是否需要清除软引用
    if (!_current_soft_ref_policy->should_clear_reference(obj, _soft_ref_timestamp_clock)) {
      return false;
    }    
  }
```
```
// should_clear_reference 实现如下
bool LRUMaxHeapPolicy::should_clear_reference(oop p,
                                             jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) { // 与_max_interval比较
    return false;
  }

  return true;
}
```
```
// Capture state (of-the-VM) information needed to evaluate the policy
void LRUMaxHeapPolicy::setup() {
  size_t max_heap = MaxHeapSize;
  max_heap -= Universe::get_heap_used_at_last_gc();
  max_heap /= M;

  _max_interval = max_heap * SoftRefLRUPolicyMSPerMB;
  // _max_interval 计算
  assert(_max_interval >= 0,"Sanity check"); 
}

```

```
// SoftReference对象记录两个时间，目的就是为了GC判断是否需要清除
public class SoftReference<T> extends Reference<T> {
    // 由JVM负责更新的，记录了上一次GC发生的时间。
    static private long clock;

    // 每次调用 get 方法都会更新，记录了当前Reference最后一次被访问的时间。
    private long timestamp;

    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }

    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }

    // 和super.get的逻辑最大的不同，就在于每次调用get都会把上次发生GC的时间，也就是
    // clock 更新到 timestamp 中去。
    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }
}

```

### PhantomReference和Cleaner
* 不能访问到referent
* Cleaner继承PhantomReference，它们本质流程都是Reference流程，有GC标记，ReferenceHandler处理
* Cleaner的clean方法由ReferenceHandler调用，最终到thunk线程进行自定义资源回收处理，以及自己的释放

```
public class PhantomReference<T> extends Reference<T> {
    // get方法永远是null，所以无法获得referent
    public T get() {
        return null;
    }
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```
```
public class Cleaner extends PhantomReference<Object>{
    // Reference需要Queue，但Cleaner自己管理ref，所以虚构个无用Queue
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue<>();

    // 所有的cleaner都会被加到一个双向链表中去，确保回收前这些Cleaner都是存活的。
    static private Cleaner first = null;

    private Cleaner
        next = null,
        prev = null;

    // 构造的时候把自己加到双向链表中去
    private static synchronized Cleaner add(Cleaner cl) {
        if (first != null) {
            cl.next = first;
            first.prev = cl;
        }
        first = cl;
        return cl;
    }

    // clean方法会调用remove把当前的cleaner从链表中删除。
    private static synchronized boolean remove(Cleaner cl) {
        // If already removed, do nothing
        if (cl.next == cl)
            return false;

        // Update list
        if (first == cl) {
            if (cl.next != null)
                first = cl.next;
            else
                first = cl.prev;
        }
        if (cl.next != null)
            cl.next.prev = cl.prev;
        if (cl.prev != null)
            cl.prev.next = cl.next;

        // Indicate removal by pointing the cleaner to itself
        cl.next = cl;
        cl.prev = cl;
        return true;
    }

    // 用户自定义的一个Runnable对象，
    private final Runnable thunk;

    // 私有有构造函数，保证了用户无法单独地使用new来创建Cleaner。
    private Cleaner(Object referent, Runnable thunk) {
        super(referent, dummyQueue);
        this.thunk = thunk;
    }

    /**
     * 所有的Cleaner都必须通过create方法进行创建。
     */
    public static Cleaner create(Object ob, Runnable thunk) {
        if (thunk == null)
            return null;
        return add(new Cleaner(ob, thunk));
    }

    /**
     * Reference Handler线程调用，来清理资源。
     */
    public void clean() {
        if (!remove(this))
            return;
        try {
            thunk.run();
        } catch (final Throwable x) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        if (System.err != null)
                            new Error("Cleaner terminated abnormally", x)
                                .printStackTrace();
                        System.exit(1);
                        return null;
                    }});
        }
    }
}

```

### FinalReference和Finalizer
* 它本质流程依然与Reference类似
* Finalizer的构造是通过标志由JVM构造（JVM构造的时候会判断finalize方法是否非空，最终会调用到register进行构造）
* Finalizer加入的queue会被Finalizer线程进行处理
* finalize方法只会调用一次，通过hasBeenFinalized这个tag保证
* ⚠️ finalize可以通过获取referent复活对象，其中存在很多安全隐患
* ⚠️ FinalizerThread为守护线程，优先级很低，很有可能抢占不到资源而导致资源无法回收


