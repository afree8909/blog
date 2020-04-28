
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428150233.png
tags: 
- 源码
- Choreogprapher
categories:
- [Android, 系统]
---


>基于API 23w

### 图文概述
Choreographer 编舞者。统一动画、输入和绘制时机
Choreographer 的作用，主要是配合 Vsync ，给上层 App 的渲染提供一个稳定的 Message 处理的时机，即 Vsync 到来的时候 ，系统通过对 Vsync 信号周期的调整，来控制每一帧绘制操作的时机


![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428150233.png)


![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428150244.png)


### Choreographer启动流程
Window添加流程中，当Activity启动执行onResume后，会执行到创建ViewRootImpt [Android系统_Window的创建和添加流程分析](https://www.jianshu.com/p/6571fbdd1bcb)

 ```
 public ViewRootImpl(Context context, Display display) {
    ...
    // VoewRootImpl 构造方法 会获取Choreographer实例
    mChoreographer = Choreographer.getInstance();
    ...
}
 ```

#### Choreographer构造
通过单例模式构建，这里是每一个线程有一个Choreographer对象，而这里的线程为应用进程的主线程

 ```
 public static Choreographer getInstance() {
    return sThreadInstance.get(); //单例模式
}

private static final ThreadLocal<Choreographer> sThreadInstance =
    new ThreadLocal<Choreographer>() {

    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper(); //获取当前线程的Looper
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        return new Choreographer(looper); 创建
    }
};
 ```
 
####  Choreographer构造方法
 初始化一个Looper和一个FrameHandler变量用来处理消息，另外创建了一个 FrameDisplayEventReceiver用来请求和接受Vysnc事件
 
 ```
 private Choreographer(Looper looper) {
    mLooper = looper;
    //创建Handler对象
    mHandler = new FrameHandler(looper);
    //创建用于接收VSync信号的对象 （使用Vsync同步机制情况下创建）
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    //是指上一次帧绘制时间点
    mLastFrameTimeNanos = Long.MIN_VALUE;
    //帧间时长，默认16.7ms.
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    //创建回调对象
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
 ```
 
####  创建FrameHandler
 Choreographer 处理绘制的逻辑核心在 Choreographer.doFrame 函数中，doFrame 函数主要做三件事情，计算掉帧逻辑、记录帧绘制信息、
执行 CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_INSETS_ANIMATION、CALLBACK_TRAVERSAL、CALLBACK_COMMIT
 
 
 ```
 private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME:
                doFrame(System.nanoTime(), 0);
                break;
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK:
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}
 ```
 
#### 创建FrameDisplayEventReceiver
它继承DisPlayEvetReceiver，调用父类构造方法
另外FrameDisplayEventReceiver中三个比较重要的方法

* onVsync – Vsync 信号回调
* run – 执行 doFrame
* scheduleVsync – 请求 Vsync 信号

 
 ```
private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
            
        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
         super(looper, vsyncSource);
        }
            
        @Override
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
           ......
           mTimestampNanos = timestampNanos;
           mFrame = frame;
           Message msg = Message.obtain(mHandler, this);
           msg.setAsynchronous(true);
           mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }
        @Override
        public void run() {
           mHavePendingVsync = false;
           doFrame(mTimestampNanos, mFrame);
        }
            
        public void scheduleVsync() {
           ......  
           nativeScheduleVsync(mReceiverPtr);
           
        }
}
 ```
 
####  DisplayEventReceiver构造

 ```
public DisplayEventReceiver(Looper looper) {
    mMessageQueue = looper.getQueue(); //获取主线程的消息队列
    // 通过JNI执行native层初始化
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue);
}
 ```
 
####  android_view_DisplayEventReceiver.cpp 初始化
 创建NativeDisplayEventReceiver， 监听mReceiver的所获取的文件句柄，一旦有数据到来，则回调this(此处NativeDisplayEventReceiver)中所复写LooperCallback对象的 handleEvent
 
 ```
 static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak, jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    ...
    //创建NativeDisplayEventReceiver
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue);
    //初始化
    status_t status = receiver->initialize();
    ...

    //获取DisplayEventReceiver对象的引用
    receiver->incStrong(gDisplayEventReceiverClassInfo.clazz);
    return reinterpret_cast<jlong>(receiver.get());
}

// NativeDisplayEventReceiver继承于LooperCallback对象，此处mReceiverWeakGlobal记录的是Java层 DisplayEventReceiver对象的全局引用
NativeDisplayEventReceiver::NativeDisplayEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<MessageQueue>& messageQueue) :
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mMessageQueue(messageQueue), mWaitingForVsync(false) {
    ALOGV("receiver %p ~ Initializing display event receiver.", this);
}

status_t NativeDisplayEventReceiver::initialize() {
    //mReceiver为DisplayEventReceiver类型
    status_t result = mReceiver.initCheck();
    ...
    //监听mReceiver的所获取的文件句柄。
    int rc = mMessageQueue->getLooper()->addFd(mReceiver.getFd(), 0, Looper::EVENT_INPUT,
            this, NULL);
    ...

    return OK;
}
 ```
 
### Callback添加流程

Choreographer提供postCallback和postFrameCallback两种回调方式及对应的delay两种，其最终执行的内部postCallbackDelayedInternal方法

#### Choreographer.postCallbackDelayedInternal
创建Callback，然后添加到mCallbackQueues队列上，
然后根据是否立即执行，决定是发消息还是直接调用
最终执行scheduleFrameLocked方法

```
private void postCallbackDelayedInternal(int callbackType,
       Object action, Object token, long delayMillis) {
   synchronized (mLock) {
       //当前时间
       final long now = SystemClock.uptimeMillis();
       //回调执行时间，=当前时间+delay时间
       final long dueTime = now + delayMillis;
       //添加到callback队列
mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

       if (dueTime <= now) {
           scheduleFrameLocked(now); // 立即执行
       } else {
           // 发送消息执行
           Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
           msg.arg1 = callbackType;
           msg.setAsynchronous(true);
           mHandler.sendMessageAtTime(msg, dueTime);
       }
   }
}
```
#### Choreographer.scheduleFrameLocked
使用Vsync，按时间执行到scheduleVsyncLocked方法

```
private void scheduleFrameLocked(long now) {
   if (!mFrameScheduled) {
       mFrameScheduled = true;
       if (USE_VSYNC) {
           // If running on the Looper thread, then schedule the vsync immediately,
           // otherwise post a message to schedule the vsync from the UI thread
           // as soon as possible.
           if (isRunningOnLooperThreadLocked()) {
               // 请求Vsync信号，最终会调用到native层，natie层处理完成后出发onVsync信号接收回调流程
               scheduleVsyncLocked();
           } else {
               // 发送消息，delay后执行此方法
               Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
               msg.setAsynchronous(true);
               mHandler.sendMessageAtFrontOfQueue(msg);
           }
       } else {
            // 没有使用Vsync 则直接发送msg，然后调用doFrame
           final long nextFrameTime = Math.max(
                   mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
          
           Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
           msg.setAsynchronous(true);
           mHandler.sendMessageAtTime(msg, nextFrameTime);
       }
   }
}
```

#### Choreographer.scheduleVsyncLocked
执行FrameDisplayEventReceiver.scheduleVsync
di层向SurfaceFlinger服务注册，即下一次Vsync事件会调用DisplayEventReceiver的disptachVsync方法

```

private void scheduleVsyncLocked() {
   mDisplayEventReceiver.scheduleVsync();
}
    
public void scheduleVsync() {
    nativeScheduleVsync(mReceiverPtr);
}
```

 
###  Vsync回调流程
 当vysnc信号由底层HWC触发后会执行android_view_DisplayEventReceiver的handleEvent方法。
 JNI层接受HWC的Vysnc信号，过滤处理，分发到Java层DisplayEventReceiver
 
####  android_view_DisplayEventReceiver.handleEvent
 ```
int NativeDisplayEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    ...
    nsecs_t vsyncTimestamp;
    int32_t vsyncDisplayId;
    uint32_t vsyncCount;
    //清除所有的pending事件，只保留最后一次vsync
    if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount)) {
        mWaitingForVsync = false;
        //分发Vsync
        dispatchVsync(vsyncTimestamp, vsyncDisplayId, vsyncCount);
    }
    return 1;
}

// 遍历所有的事件，当有多个VSync事件到来，则只关注最近一次的事件
bool NativeDisplayEventReceiver::processPendingEvents(
        nsecs_t* outTimestamp, int32_t* outId, uint32_t* outCount) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
            case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                gotVsync = true; //获取VSync信号
                *outTimestamp = ev.header.timestamp;
                *outId = ev.header.id;
                *outCount = ev.vsync.count;
                break;
            case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                dispatchHotplug(ev.header.timestamp, ev.header.id, ev.hotplug.connected);
                break;
            default:
                break;
            }
        }
    }
    return gotVsync;
}

// 分发Vsync ，DisplayEventReceiver进行接受
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();

    ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
    if (receiverObj.get()) {
        // 此处调用到Java层的DisplayEventReceiver对象的dispatchVsync()方法，执行进入Java层
        env->CallVoidMethod(receiverObj.get(),
                gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
    }

    mMessageQueue->raiseAndClearException(env, "dispatchVsync");
}
 
 ```
 
####  FrameDisplayEventReceiver.onVsync
Java层接受Vsync事件，通过Handler通信机制发送消息，后续执行到它自身的run方法，然后执行doFrame操作

```
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
    
    @Override
    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ...
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        //该消息的callback为当前对象FrameDisplayEventReceiver
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        //此处mHandler为FrameHandler
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }
    ...
}


```
 
####  Choreographer.doFrame
* 先判断Vsync信号间隔时间和刷新时间是否符合
* 顺序执行4种时间对应的CallbackQueue队列中注册的回调函数
    1. CALLBACK_INPUT : 处理输入事件处理有关
    2. CALLBACK_ANIMATION ： 处理 Animation 的处理有关
    3. CALLBACK_INSETS_ANIMATION ： 处理 Insets Animation 的相关回调
    4. CALLBACK_TRAVERSAL : 处理和 UI 等控件绘制有关
    5. CALLBACK_COMMIT ： 处理 Commit 相关回调


```
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        // 每调用一次scheduleFrameLocked()，则mFrameScheduled=true，能执行一次doFrame()操作，执行完doFrame()并设置mFrameScheduled=false；
        if (!mFrameScheduled) {
            return; // mFrameScheduled=false，则直接返回。
        }
        //原本计划的绘帧时间点
        long intendedFrameTimeNanos = frameTimeNanos;        
        //起始时间
        startNanos = System.nanoTime();
        //计算消息发送与函数调用开始之间所花费的时间
        final long jitterNanos = startNanos - frameTimeNanos;
        //如果线程处理该消息的时间超过了屏幕刷新周期
        if (jitterNanos >= mFrameIntervalNanos) {
            // 计算函数调用期间所错过的帧数
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            //当掉帧个数超过30，则输出相应log
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            //对齐帧的时间间隔
            frameTimeNanos = startNanos - lastFrameOffset; 
        }
        //此处frameTimeNanos是底层VSYNC信号到达的时间戳，如果frameTimeNanos小于一个屏幕刷新周期，则重新请求VSync信号
        if (frameTimeNanos < mLastFrameTimeNanos) {
            scheduleVsyncLocked();
            return;
        }

        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
        mFrameScheduled = false;
        mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");

        mFrameInfo.markInputHandlingStart();
        //执行4种事件对应的回调方法
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        //标记动画开始时间
        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

####  Choreographer.doCallbacks
* 从队列头mHead查找CallbackRecord对象，当队列头部的callbacks对象为空或者执行时间还没到达，则直接返回；
* 开始执行相应回调的run()方法；
* 回收callbacks，加入对象池mCallbackPool，就是说callback一旦执行完成，则会被回收。

```
void doCallbacks(int callbackType, long frameTimeNanos) {
    CallbackRecord callbacks;
    synchronized (mLock) {
        final long now = System.nanoTime();
        // 从队列查找相应类型的CallbackRecord对象
        callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                now / TimeUtils.NANOS_PER_MS);
        if (callbacks == null) {
            return;  //当队列为空，则直接返回
        }
        mCallbacksRunning = true;

        if (callbackType == Choreographer.CALLBACK_COMMIT) {
            final long jitterNanos = now - frameTimeNanos;
            //当commit类型回调执行的时间点超过2帧，则更新mLastFrameTimeNanos。
            if (jitterNanos >= 2 * mFrameIntervalNanos) {
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                        + mFrameIntervalNanos;
                frameTimeNanos = now - lastFrameOffset;
                mLastFrameTimeNanos = frameTimeNanos;
            }
        }
    }
    try {
        for (CallbackRecord c = callbacks; c != null; c = c.next) {
            c.run(frameTimeNanos); 
        }
    } finally {
      synchronized (mLock) {
          mCallbacksRunning = false;
          //回收callbacks，加入对象池mCallbackPool
          do {
              final CallbackRecord next = callbacks.next;
              recycleCallbackLocked(callbacks);
              callbacks = next;
          } while (callbacks != null);
      }
      Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}

```
#### CallbackRecord.run
* 当token的数据类型为FRAME_CALLBACK_TOKEN，则执行该对象的doFrame()方法;
* 当token为其他类型，则执行该对象的run()方法。

```
private static final class CallbackRecord {
    public CallbackRecord next;
    public long dueTime;
    public Object action; // Runnable或者 FrameCallback
    public Object token;

    public void run(long frameTimeNanos) {
        if (token == FRAME_CALLBACK_TOKEN) {
            ((FrameCallback)action).doFrame(frameTimeNanos);
        } else {
            ((Runnable)action).run();
        }
    }
}
```

#### Choreographer.CALLBACK_TRAVERSAL
这里主要分析Traversal类型，即View的绘制流程

* ViewRootImpl.scheduleTraversals , 添加callback
* 回调 TraversalRunnable.run 方法
* 开始View的绘制流程

```
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //为了提高优先级，先 postSyncBarrier
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        // 真正开始执行 measure、layout、draw
        doTraversal();
    }
}
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        // 这里把 SyncBarrier remove
mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        // 真正开始
        performTraversals();
    }
}
private void performTraversals() {
      // measure 操作
      if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight() || contentInsetsChanged || updatedConfiguration) {
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      }
      // layout 操作
      if (didLayout) {
          performLayout(lp, mWidth, mHeight);
      }
      // draw 操作
      if (!cancelDraw && !newSurface) {
          performDraw();
      }
}
```



-------
推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)
参考
[Android 基于 Choreographer 的渲染机制详解](https://www.androidperformance.com/2019/10/22/Android-Choreographer/)
[Choreographer原理](http://gityuan.com/2017/02/25/choreographer)
[源码分析](https://www.jianshu.com/p/996bca12eb1d)

