---
tags: 
- 源码
- Handler
categories:
- [Android, 系统]
---

### 前言
Android应用程序有主线程和子线程之分，其中主线程由AMS请求Zygote进程创建；而子线程则由主线程或者其它子线程创建。我们知道Android规定只能在主线程中执行与界面相关工作（严格意义是界面创建元素对应的线程），一旦我们在主线程执行的任务过重，就可能导致UI绘制无法及时完成，产生掉帧现象，或者更严重直接ANR。所以为了避霾类似问题，我们需要多线程技术方案，把那些复杂或者非重要的任务移到其它线程执行，来提升体验。
我们知道对于不定期的后台任务，一般有两种处理方式。第一种方式是每当条件满足的时，就创建一个子线程来执行一个不定期的后台任务，当这个不定期的任务执行完成之后，这个新创建子线程就随之推出。第二种方式是创建一个具有消息循环的子线程，每当条件满足时，就将一个不定期后台任务封装成一个消息发送到子线程的消息队列中去执行，而当条件不满足时，这个子线程就会因问它的消息队列为空而进入睡眠等待状态。虽然第一种方式创建的子线程不需要消息循环机制，但是频繁的创建和销毁子线程是有代价的，因此更倾向于第二种方式来执行那些不定期的后台任务。Android应用程序主线程和子线程都是具有消息循环机制的。
下面我们将从Android消息机制原理、Android线程消息循环模型和Android线程和线程池进行全面理解。

### Android消息机制
Android的消息机制主要指Handler的运行机制及它附带的MessageQueue和Looper的工作过程。

#### Handler

**主要用途**

1. 在未来某个时间点处理 Messages 或者执行 Runnables
2. 将任务切换到另一个线程执行

**源码分析**
大致流程：构造Handler -> 发送Runnable -> 组合Message -> Message入队 -> Looper轮训Message执行任务 -> 取Message对应Handler执行消息回调处理

```
    // 字段
    /*
     * Set this flag to true to detect anonymous, local or member classes
     * that extend this Handler class and that are not static. These kind
     * of classes can potentially create leaks.
     * 非static的匿名内部类、局部变量或成员变量都将可能造成内存泄漏
     */
    private static final boolean FIND_POTENTIAL_LEAKS = false;
    final Looper mLooper; // 持有对应Looper，获取对应queue
    final MessageQueue mQueue; // 持有对应的消息队列，进行消息入队操作
    final Callback mCallback; // hook，非null 优先回调处理消息
    final boolean mAsynchronous; // 是否异步消息标识
    IMessenger mMessenger; // 作用进程间通信
    
    // 构造函数 Handler持有对应线程的Looper，同时持有对应Looper的MessageQueue
    public Handler() {  ...  }
    public Handler(Callback) {  ...  }
    public Handler(Looper) {  ...  }
    public Handler(Looper,Callback) {  ...  }
    public Handler(boolean) {  ...  }
    public Handler(Callback,boolean) {  ...  }
    public Handler(Looper,Callback,boolean) {  ...  }


    // post方法，针对不同执行时间的Runnable的方法，最终Runnable转为Message，调用发消息方法插入队列
    public final boolean post(Runnable r){ ... }
    public final boolean postAtFrontOfQueuepo(Runnable r){ ... }
    public final boolean postAtTime(Runnable r,long uptimeMillis){ ... }
    public final boolean postAtTime(Runnable r,Object token,long uptimeMillis){ ... }
    public final boolean postDelayed(Runnable r,long delayMillis){ ... }
    
    // 将runnable转为Message
    private static Message getPostMessage(Runnable r) {...}
    private static Message getPostMessage(Runnable r, Object token) {...}
    
    //  sendMessage方法，内部实现就是将Message入MessageQueue
    public final boolean sendMessage(Message msg){ ... }
    public final boolean sendEmptyMessage(int what){ ... }
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis){ ... }
    public final boolean sendMessageDelayed(Message msg, long delayMillis){ ... }
    ...
    
    // Message插入MessageQueue，
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this; // 指定Msg处理对象为当前Handler
        if (mAsynchronous) { // 是否异步消息，即跳过屏障执行。
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis); // 执行等待时间
    }
    
    
    // Looper轮训到执行任务消息时，便调用Message的target即此Handler的这个发送消息方法进行处理
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) { // 优先尝试msg的callback回调
            handleCallback(msg);
        } else {
            if (mCallback != null) { // Handler构造callBack
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg); // 子类实现处理
        }
    }

```

#### Message
数据结构主要包含一个int标识，一个long执行时间标识，一个Object数据传输对象，一个Bundle类型data存储对象，记录对应的Hanlder对象。
另外维护了一个默认50大小的单链表，用于Message创建，后续系统会回收Message

```
// 定义一个可以发送给 Handler 的消息，包含描述和任意数据对象。消息对象有两个额外的 int 字段和一个 object 字段，这可以满足大部分场景的需求了。
// 推荐通过Message.obtain()构建Message而不是直接new，里面维护了默认50大小的链表Message的sPool 
public final class Message implements Parcelable {
    public int what; // 消息标识
    public Object obj; // 消息数据存储，用于非bundle传输
    // Flag标识（是否使用、是否异步消息）
    /*package*/ static final int FLAG_IN_USE = 1 << 0;
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;
    /*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;
    /*package*/ int flags;
    
    /*package*/ long when; // 执行时间
    /*package*/ Bundle data; // 非obj传输情况
    /*package*/ Handler target; // 发送的Handler
    /*package*/ Runnable callback; /
    // sometimes we store linked lists of these things
    /*package*/ Message next; // Message单链表指向 

    private static final Object sPoolSync = new Object();
    private static Message sPool; // Message 链表池
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50;
    private static boolean gCheckRecycle = true; // 回收标识

    // 各种Message构建方法 
    public static Message obtain(Handler h, int what, int arg1, int arg2) { ... }

}
```

#### Looper
通过ThreadLocal实现各线程持有自己的Looper
loop方法进行消息轮训，获取消息，调用目标Handler分发任务

```
// 扮演消息循环角色，从MessageQueue取消息，有就执行，无则阻塞等待
public final class Looper {
    // 重要字段
    // ThreadLocal缓存，实现各线程持有各自Looper对象。参考https://www.jianshu.com/p/8a7fe7d592f8
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;  // 持有主线程Looper，便于其它线程交互

    final MessageQueue mQueue; // Looper对应的MessageQueue
    final Thread mThread; // Looper对应当前线程

    // 构造函数私有，必须通过prepare方法来构建
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    
    // 构建Looper，通过ThreadLocal维护Looper（各线程对应一个Looper）
    private static void prepare(boolean quitAllowed) {
        // 主线程消息轮训不允许退出，一直循环处理
        // 子线程消息轮训可以退出
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    // 主线程构建Looper方法（ActivityThread调用）
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper(); // 此处缓存主线程Looper，便于后续与主线程的交互
        }
    }

    // 最重要的 loop 方法，消息轮训实现 （部分关键代码）
    public static void loop() {
        final Looper me = myLooper(); // 获取当前线程对应的loop
        final MessageQueue queue = me.mQueue;
        
        for (;;) { // 循环去消息 （MessageQueue取过程可能阻塞）
            Message msg = queue.next(); // might block （参考MessageQueue next方法）
            if (msg == null) {
                return;
            }
            
            msg.target.dispatchMessage(msg);// 派发消息到对应Handler
             
            msg.recycleUnchecked(); // 释放message
        }
    }    

}
```

#### MessageQueue
重点关注 next取message 和 enqueueMessage插入message方法
>主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，详情见Android消息机制1-Handler(Java层)，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。


```
// 基于Native JNI实现，重点看next和enqueueMessage方法
public final class MessageQueue {

    private long mPtr; // 保存Native层的MessageQueue的对象

    Message mMessages; // 即将执行的message（链表头部）
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>(); // 空闲handler列表（用于添加空闲任务）
    private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;
    private IdleHandler[] mPendingIdleHandlers; // IdleHandler数组
    private boolean mQuitting; // 是否终止

    // Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
    private boolean mBlocked; // 表明next方法是否block 当调用JNI pollOnce方法

    // The next barrier token.
    // Barriers are indicated by messages with a null target whose arg1 field carries the token.
    private int mNextBarrierToken;
    // JNI 方法
    private native static long nativeInit();
    ......

    // 构造函数
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit(); // 进行native层初始化
    }
    
    // 重点方法 ，获取下一个待处理任务 （部分重点代码）
    Message next() {
        int pendingIdleHandlerCount = -1; // 用于IdleHandler任务处理
        int nextPollTimeoutMillis = 0;
        
        for (;;) { // 无限循环
            nativePollOnce(ptr, nextPollTimeoutMillis); //Native Looper的epoll

            synchronized (this) {
             
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // 处理无target的异步消息 （暂时忽略）
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                
                if (msg != null) {
                    if (now < msg.when) {
                        // 标记下一次轮训时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next; // 记录下一个头部message
                        }
                        msg.next = null;
                        
                        msg.markInUse(); // 标记message状态
                        return msg; // 返回message
                    }
                } else {
                    nextPollTimeoutMillis = -1;
                }

                
                if (mQuitting) { // 终止
                    dispose();
                    return null;
                }

                // 后面是 IdleHandler相关处理逻辑（暂时忽略）
                ......
                
        }
    }
    
    
    // 重点方法 ，Message入队 （部分重点代码）
    boolean enqueueMessage(Message msg, long when) {
        synchronized (this) {
            msg.markInUse(); // 
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // 立即执行任务，标记为head message。
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked; // 唤醒，如果处于blocked状态
            } else {
                // 考虑是否异步任务
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                // 根据执行时间先后，插入message链表队列
                for (;;) { 
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // 唤醒. 参考JNI方法
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
    
}
```

JNI方法（参考）[Native层](http://gityuan.com/2015/12/27/handler-message-native/)

MessageQueue通过mPtr变量保存NativeMessageQueue对象，从而使得MessageQueue成为Java层和Native层的枢纽，既能处理上层消息，也能处理native层消息

```
//Looper.cpp

int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        ...
        result = pollInner(timeoutMillis);
    }
}

int Looper::pollInner(int timeoutMillis) {
	...
    // Poll.
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
    mIdling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
	//阻塞等待可以读取管道的通知
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mIdling = false;

    // Acquire lock.
    mLock.lock();
	...
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();// 关键代码方法
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
           ...
        }
    }
Done: ;

  ...
    return result;
}

void Looper::awoken() { // 唤醒方法 （enqueueMessage方法调用）
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ awoken", this);
#endif

    char buffer[16];
    ssize_t nRead;
    do {
        nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));//可以看到读取了管道中的内容
    } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
}


void Looper::wake() { // 最终调用到 native wake方法
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    ssize_t nWrite;
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);//进行了写操作
    } while (nWrite == -1 && errno == EINTR);

    if (nWrite != 1) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}


```



### Android线程消息循环模型
#### 主线程消息循环模型
ActivityThread通过ApplicationThread和AMS进行进程间通讯，AMS以进程间通信的方式完成ActivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread中去执行，

system_server进程：
系统进程，包含了大量的系统服务。如图中ApplicationThreadProxy、ActivityManagerService，这两个服务都运行在system_server进程的不同线程中。另外ATP、AMS都是基于IBinder接口，属于binder线程，都是由binder底层驱动创建和销毁

App进程：
即应用程序，主线程主要负责Activity/Service等组件的生命周期以及UI相关操作；另外每个App进程至少都有两个Binder线程，ApplicationThread和ActivityManagerProxy。

Binder
用于不同进程之间通信


![主线程消息循环模型](https://upload-images.jianshu.io/upload_images/9696036-5f594a326e5a7d2e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 主线程与子线程交互的消息循环模型

![主线程与子线程交互的消息循环模型.jpg](https://upload-images.jianshu.io/upload_images/9696036-cb9d80fde5fadfb6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### Android线程和线程池相关概念

参考：[Java线程池理解]([https://www.jianshu.com/p/d9e46d5a4af9](https://www.jianshu.com/p/d9e46d5a4af9)

### FAQ
##### Looper死循环为什么不会导致应用卡死，会消耗大量资源吗？
见MessageQueue的next方法，涉及到Linux pipe/epoll机制，因为在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next ()中的nativePollOnce方法里，这是主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事物发生。所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。 

##### 主线程的消息循环机制是什么（死循环如何处理其它事务）？
参考前面的主线程消息模型图例

##### ActivityThread 的动力是什么？（ActivityThread执行Looper的线程是什么）
ActivityThread通过main方法执行，也就是咱们的Runtime线程。该线程默认是不可quit的


```
 //  程序入口，runtime线程执行
 public static void main(String[] args) {
     
        Looper.prepareMainLooper(); // 将该线程对应Looper标记为MainLooper

        ActivityThread thread = new ActivityThread(); 
        thread.attach(false, startSeq);  // 初始化ActivityThread
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Looper.loop(); // 循环
    }
```


##### Handler 是如何能够线程切换，发送Message的？（线程间通讯）
主线程到其它线程，只需要创建线程执行任务就可以。其它线程切回主线程，只需要拿到或者创建主线程Handler既可以发送消息切换回去




