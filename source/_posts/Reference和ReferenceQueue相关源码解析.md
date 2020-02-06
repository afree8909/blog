---
tags: 
- 源码
categories:
- [Java, 系统]
---

### 为什么需要引用？
我们知道在最早的JVM实现里，是使用“跟踪回收”算法从GC ROOTS出发，按照BFS或者DFS遍历所有可达对象，针对不不可达对象进行回收。但随着Java演进，暴露出一些不能覆盖的场景，例如：某些场景下使用方希望在回收具体对象的同时还能辅助回收这个对象绑定的一些资源（如socket、堆外内存等）、某些场景下希望使用堆内缓存尽可能缓存更多更久的数据但不OOM。基于此，JDK在1.2引入了Refreence及其子类来支持一些新特性和功能


### Reference介绍
##### Reference类图
![](https://upload-images.jianshu.io/upload_images/9696036-fec84bdd662d2c10.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* **FinalReference**，一种保底策略，因为GC只能管理自动内存资源而无法管理其它资源（如堆外内存、file handle、socket等），这些需要使用方手动对资源进行管理
* **SoftReference**，软引用，只有在堆内存不足时，垃圾回收器会回收对应引用。所以比较适合用来实现不是特别重要的缓存
* **WeakReference**，弱引用，每次垃圾回收都会回收其引用，一般在需要控制内存但又又想要尽量用到内存的场景下使用
* **PhantomReference**，虚引用，对引用无影响，只用于获取对象被回收的通知。和软引用以及弱引用不同的是幻影引用指向的对象没有其他强引用、软引用指向时不会自动被GC清理。

⚠️ 因为默认的引用就是强引用，所以没有强引用的Reference实现类。
详情：[Reference子类源码解析](https://www.jianshu.com/p/7ff38dfbb5a8)


#### Reference生命周期
Referent被GC回收，则会根据是持有ReferenceQueue，而加入到对应到ReferenceQueue中，这样可以通过RQ来监听reference是否回收状态

![](https://upload-images.jianshu.io/upload_images/9696036-78514b107996e4a8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Reference运行结构
![](https://upload-images.jianshu.io/upload_images/9696036-0bd5881d52164f6e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Reference源码
###### 字段
* referent：  引用指向的对象
* queue： ReferenceQueue，Reference构造的时候传入，其内部封装了单向链表的添加，删除和遍历等操作。用于Reference状态监听及管理
* discovered：单向链表，由JVM维护
* next：指向ReferenceQueue中下一个元素，ReferenceQueue链表指针
* pending：discovered链表表头，在referent被回收后的reference
将有JVM标记，等待入队处理

###### 方法
* static代码块：构造ReferenceHandler线程，循环执行tryHandlePending方法
* tryHandlePending：循环处理pending链表头，维护discovered链表，如果pending不为空，则进行插入ReferenceQueue进行后续操作。（这里如果是cleaner，则先进行clean操作）

###### 内部类
* ReferenceHandler，线程实现类，run方法中循环调用Refreence的静态方法tryHandlePending

```
// 用于控制垃圾回收器操作与Pending状态的Reference入队操作不冲突执行的全局锁
// 垃圾回收器开始一轮垃圾回收前要获取此锁
// 所以所有占用这个锁的代码必须尽快完成，不能生成新对象，也不能调用用户代码
static private class Lock { };
private static Lock lock = new Lock();

private static class ReferenceHandler extends Thread {

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }

    public void run() {
        // 这个线程一直执行
        for (;;) {
            Reference<Object> r;
            // 获取锁，避免与垃圾回收器同时操作
            synchronized (lock) {
                // 判断pending-Reference链表是否有数据
                if (pending != null) {
                    // 如果有Pending Reference，从列表中取出
                    r = pending;
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // 如果没有Pending Reference，调用wait等待
                    // 
                    // wait等待锁，是可能抛出OOME的，
                    // 因为可能发生InterruptedException异常，然后就需要实例化这个异常对象，
                    // 如果此时内存不足，就可能抛出OOME，所以这里需要捕获OutOfMemoryError，
                    // 避免因为OOME而导致ReferenceHandler进程静默退出
                    try {
                        try {
                            lock.wait();
                        } catch (OutOfMemoryError x) { }
                    } catch (InterruptedException x) { }
                    continue;
                }
            }

            // 如果Reference是Cleaner，调用其clean方法
            // 这与Cleaner机制有关系，不在此文的讨论访问
            if (r instanceof Cleaner) {
                ((Cleaner)r).clean();
                continue;
            }

            // 把Reference添加到关联的ReferenceQueue中
            // 如果Reference构造时没有关联ReferenceQueue，会关联ReferenceQueue.NULL，这里就不会进行入队操作了
            ReferenceQueue<Object> q = r.queue;
            if (q != ReferenceQueue.NULL) q.enqueue(r);
        }
    }
```

#### ReferenceQueue源码
```
public class ReferenceQueue<T> {
    // 单向链表
    private volatile Reference<? extends T> head = null;
}

```
```
boolean enqueue(Reference<? extends T> r) { 
    /* Called only by Reference class */
    synchronized (lock) {
		// 判断Reference是否需要入队
        ReferenceQueue<?> queue = r.queue;
        if ((queue == NULL) || (queue == ENQUEUED)) {
            return false;
        }
        assert queue == this;
        
        // Reference入队后，其queue变量设置为ENQUEUED
        r.queue = ENQUEUED;
        // Reference的next变量指向ReferenceQueue中下一个元素
        r.next = (head == null) ? r : head;
        head = r;
        queueLength++;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(1);
        }
        lock.notifyAll();
        return true;
    }
}

public Reference<? extends T> poll() {
        // 出队
}

```


### Reference类解决的三个问题
* 实现特定的引用类型，支持不同功能
* 使用者可以在对象被回收后得到通知
* 使用者可以自定义回收，进行非GC回收的资源释放



参考
https://coldwalker.com/2019/02//gc_intro/
https://github.com/zxiaofan/JDK/blob/master/JDK1.8/src/java/lang/ref/ReferenceQueue.java


