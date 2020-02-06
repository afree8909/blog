
---
tags: 
- 源码
categories:
- [Java, 系统]
---


### 前言
线程池是维护了一批线程来处理用户提交的任务，达到线程复用的目的，合理使用线程有3个好处。

1. 降低资源消耗。通过重用已创建的线程来降低线程创建和销毁造成的消耗
2. 提高响应速度。通过已创建线程立即执行任务，减少了线程的创建时间
3. 提高线程的可管理性。通过合理地使用线程池，从而实现统一分配、调优和监控等


### 线程池的工作流程

![15534290301842.jpg](https://upload-images.jianshu.io/upload_images/9696036-bf3d2e726a5dd6d0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/9696036-d52f7d01a39f8a44.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**ThreadPoolExecutor执行任务流程**

1. 如果当前运行的线程少于corePoolSize，则创建线程来执行任务
2. 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue
3. 如果BlockingQueue已满，则创建新的线程来处理任务
4. 如果当前运行线程总数大于maximumPoolSize，任务将被拒绝执行。并调用RejectedExecutionHandler.rejectedExecution方法



### 线程池包结构

线程池简要组成部分可以分三块，任务、任务执行者及工具类相关

* 任务: Callable、Runnable、FutureTask
* 任务执行者：ThreadPoolExecutor、ScheduledThreadPoolExecutor
* 工具类：Executors

![](https://upload-images.jianshu.io/upload_images/9696036-836ddd4f9b3c7f1f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### ThreadPoolExecutor解析
Java线程池最核心的类即ThreadPoolExecutor，它是线程池的实现类。
##### 类图

![](https://upload-images.jianshu.io/upload_images/9696036-ef17773e5dd66c5f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 构造函数

* WorkQueue: 任务阻塞队列，缓存将要执行的Runnable任务

    *  ArrayBlockingQueue：基于数组有界阻塞队列
    *  LinkedBlockingQueue：基于链表阻塞队列
    *  SynchronousQueue：不存储元素的阻塞队列（读写须等待一并进行）
    *  PriorityBlockingQueue：支持优先级的无界队列

* RejectedExecutionHandler：任务拒绝策略，默认AbortPolicy
    * AbortPolicy：直接抛出异常。
    * CallerRunsPolicy：只用调用者所在线程来运行任务。
    * DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
    * DiscardPolicy：不处理，丢弃掉。  
  

```
 public ThreadPoolExecutor(
     int corePoolSize,                   // 核心线程数
     int maximumPoolSize,                // 最大线程数
     long keepAliveTime,                 // 非核心线程闲置回收时间
     TimeUnit unit,                      // 时间单位
     BlockingQueue<Runnable> workQueue,  // 装载任务的阻塞队列                                              ThreadFactory threadFactory,             // 线程创建工厂                        
     RejectedExecutionHandler handler    // 任务拒绝状态时处理策略
     ) { 
        // ......                          
}
```

##### 状态变量

```
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS; // 运行中
    private static final int SHUTDOWN   =  0 << COUNT_BITS; // 拒绝新任务
    private static final int STOP       =  1 << COUNT_BITS; // 拒绝新任务且不处理剩余任务
    private static final int TIDYING    =  2 << COUNT_BITS; // 所有线程停止，准备执行终止方法 
    private static final int TERMINATED =  3 << COUNT_BITS; // 已执行终止方法

    // 线程状态 ctl值取低29位
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
     // 线程状态 ctl值取高3位
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }

```

##### execute()
```
 public void execute(Runnable command) {
        int c = ctl.get();
        // 首先，运行线程数是否小雨核心线程
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true)) // 创建核心线程
                return;
            c = ctl.get();
        }
        // 其次，往队列中插入任务
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 否则，创建非核心线程执行任务
        else if (!addWorker(command, false))
            reject(command); // 如果上面都失败，则拒绝执行任务，调用handler
    }
```

##### addWork()

```
private boolean addWorker(Runnable firstTask, boolean core) {
        
        // 使用CAS机制轮训线程池的状态，如果处于SHTUTDOWN及以上状态则拒绝执行任务
        ...

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask); // 构建Worker（worker会创建thread）
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w); // 新建woker线程加入集合保存
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start(); // 执行任务
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

##### Work类
TreadPoolExecutor内部类，Worker构造方法指定第一个要执行的任务，并通过线程工厂创建线程。
Worker为Runnable，可以执行run，即调用到外部类的runWorker方法
继承AbstractQueuedSynchronizer，执行每个任务前通过lock方法加锁，执行完后通过unlock释放锁，以防止运行中任务中断。

```
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
      
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        Runnable firstTask;
        
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask; 
            this.thread = getThreadFactory().newThread(this); // 构造的时候创建线程
        }

        
        public void run() {
            runWorker(this);
        }

    }

```

##### runWork（）
每一个Worker在getTask()成功之后都要获取Worker的锁之后运行，也就是说运行中的Worker不会中断。因为核心线程一般在空闲的时候会一直阻塞在获取Task上，也只有中断才可能导致其退出。这些阻塞着的Worker就是空闲的线程（当然，非核心线程阻塞之后也是空闲线程）。如果设置了keepAliveTime>0，那非核心线程会在空闲状态下等待keepAliveTime之后销毁，直到最终的线程数量等于corePoolSize

```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run(); // 任务执行
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //当指定任务执行完成，阻塞队列中也取不到可执行任务时，会进入这里，做一些善后工作
            //比如在corePoolSize跟maximumPoolSize之间的woker会进行回收
            processWorkerExit(w, completedAbruptly);
        }
    }


```

##### getTask（）
通过一个循环不断轮询任务队列有没有任务到来，首先判断线程池是否处于正常运行状态，根据超时配置有两种方法取出任务：
BlockingQueue.poll 阻塞指定的时间尝试获取任务，如果超过指定的时间还未获取到任务就返回null。
BlockingQueue.take 这种方法会在取到任务前一直阻塞。
keepAliveTime代表了线程池中的线程（即woker线程）的存活时间，如果到期则回收woker线程
FixedThreadPool使用的是take方法，所以会线程会一直阻塞等待任务。CachedThreadPool使用的是poll方法，也就是说CachedThreadPool中的线程如果在60秒内未获取到队列中的任务就会被终止。

```
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            ......
            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 超时配置时间，通过不同方法取任务
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();

                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

### Executors构建线程方法

##### newFixedThreadPool
定长线程池：
可控制线程最大并发数（同时执行的线程数）
超出的线程会在队列中等待

```
  public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
}
```

##### newCachedThreadPool
可缓存线程池：
线程数无限制
有空闲线程则复用空闲线程，若无空闲线程则新建线程
一定程序减少频繁创建/销毁线程，减少系统开销

```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

##### newSingleThreadExecutor
单线程化的线程池：

有且仅有一个工作线程执行任务
所有任务按照指定顺序执行，即遵循队列的入队出队规则

```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

##### newScheduledThreadPool
支持定时以指定周期循环执行任务:

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```

