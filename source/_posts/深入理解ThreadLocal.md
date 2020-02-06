---
tags: 
- 源码
categories:
- [Android, 系统]
---


### ThreadLocal是什么？
> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).

释义：
ThreadLocal提供线程本地实例。它与普通变量区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。
ThreadLocal 变量通常被private static修饰。当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。



### ThreadLocal解决什么问题？
并发编程中常见需求：每条线程都需要存取一个同名变量，但每条线程中该变量的值均不相同。
常规思路：使用一个线程共享的Map<Thread,T>,通过map来获取对应线程变量的值。带来的问题是需要同步，效率比较低。
而ThreadLocal从另一个角度解决多线程并发的问题。ThreadLocal会为每一个线程提供一个独立的变量副本，从而隔离了多个线程对数据的访问冲突。因为每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。
（空间换时间）



### ThreadLocal源码解析
##### 整体结构图
* Thread持有成员变量threadLocals（ThreadLocalMap）
* ThreadLocalMap是一个映射表，内部实现一个数组，每一个元素位Entry
* Entry为一个键值对，key为Thread，Value为任何对象

![](media/15527089073930.jpg)

##### ThreadLocal解析
set和get方法通过线程对应ThreadLocalMap来管理实现

```
   public void set(T value) {
        Thread t = Thread.currentThread(); // 获取当前线程
        ThreadLocalMap map = getMap(t); // 获取线程对应映射表
        if (map != null) // 设置KV
            map.set(this, value);
        else
            createMap(t, value);
    }

```
```
public T get() {
      Thread t = Thread.currentThread();
      ThreadLocalMap map = getMap(t);
      if (map != null) {
          // 获取映射表中当前ThreadLocal对应的Value
          ThreadLocalMap.Entry e = map.getEntry(this);
          if (e != null) {
              @SuppressWarnings("unchecked")
              T result = (T)e.value;
              return result;
          }
      }
      // 如果Map还未初始化或者Map中没有找到Key，则设置一个初始值
      return setInitialValue();
  }

```
##### ThreadLocalMap解析

**成员变量和内部类**

```
 // 为了处理非常大(指的是值)和长时间的用途，哈希表的Key使用了弱引用(WeakReferences)。
 // 引用的队列(弱引用)不再被使用的时候，对应的过期的条目就能通过主动删除移出哈希表。
 static class ThreadLocalMap {

        // Value为WeakReference持有
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * 初始化容量，必须是2的幂次方
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * 哈希表，长度为2的幂次方
         */
        private Entry[] table;

        /**
         * The number of entries in the table.（Entry数目）
         */
        private int size = 0;

        /**
         * 记录下一次扩容阀值
         */
        private int threshold; // Default to 0

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.（设置下一次阀值，len的三分之二）
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        /**
         * Increment i modulo len.（以len为模增加i）
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.（以len为模减少i）
         */
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }


```

**构造函数**

```
// 构造ThreadLocal时候使用，对应ThreadLocal的实例方法void createMap(Thread t, T firstValue)
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 哈希表默认容量为16
    table = new Entry[INITIAL_CAPACITY];
    // 计算第一个元素的哈希码 （黄金分割数 &（容量-1））
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}

// 构造InheritableThreadLocal时候使用，基于父线程的ThreadLocalMap里面的内容进行提取放入新的ThreadLocalMap的哈希表中
// 对应ThreadLocal的静态方法static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap)
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];
    // 基于父ThreadLocalMap的哈希表进行拷贝
    for (Entry e : parentTable) {
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}

```

**set部分**

```
 private void set(ThreadLocal<?> key, Object value) {
 
            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.
            
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1); //取index值
            // nextIndex方法实现全遍历
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) { 
                 
                ThreadLocal<?> k = e.get();

                if (k == key) { // 匹配则直接替换value
                    e.value = value;
                    return;
                }
                // key为null，则替换key并赋值value
                if (k == null) { 
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            // 追求效率的平衡，仅清理i到sz指引的回收value
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash(); // 重新hash
        }
        
          private void rehash() {
            // 清理一遍哈希表
            expungeStaleEntries();

            // 哈希表元素数目大雨 3/4阀值，则扩容
            if (size >= threshold - threshold / 4)
                resize();
        }


    private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

```
**get部分**

```
  private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else  // 注意这里，如果e为null或者Key对不上，会调用getEntryAfterMiss
                return getEntryAfterMiss(key, i, e);
        }

 private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            // 这里会通过nextIndex尝试遍历整个哈希表，如果找到匹配的Key则返回Entry
            // 如果哈希表中存在Key == null的情况，调用expungeStaleEntry进行清理
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }


```


### FAQ
* 什么情况下ThreadLocal的使用会导致内存泄漏？
    * 通过源码分析可以知道，ThreadLocalMap存放的Value是弱引用，会自动GC。但是对应的强引用则只在调用get、set或者remove才可能被回收
    * 例如：大量地(静态)初始化ThreadLocal实例，初始化之后不再调用get()、set()、remove()方法。



### ThreadLocal使用场景
* JavaWeb中Session的实现
* Android应用中的Looper创建管理




### 参考
[ThreadLocal源码分析-黄金分割数的使用](http://throwable.coding.me/2019/02/17/java-currency-threadlocal/#%E9%BB%84%E9%87%91%E5%88%86%E5%89%B2%E6%95%B0%E7%9A%84%E5%BA%94%E7%94%A8)

