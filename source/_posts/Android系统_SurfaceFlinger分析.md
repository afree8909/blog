
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145918.png
tags: 
- 源码
- SurfaceFlinger
categories:
- [Android, 系统]
---

>基于API 23

SurfaceFlinger，合成抛射机，它在Android系统是一个独立的服务进程
它的作用是接受多个来源的图形显示数据，将他们合成，然后发送到显示设备。
它的工作内容主要包括合成的创建和管理、Vsync信号的处理
本文分析SurfaceFlinger的启动流程，和Vsync信号的处理流程

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145918.png)


### 启动流程
SurfaceFlinger 进程是由 init 进程创建的，运行在独立的 SurfaceFlinger 进程中。init 进程读取 init.rc 文件启动 SurfaceFlinger。首先来看main方法

#### main方法
* 设定线程池上线4，并启动binder线程池
* 创建SF对象
* 初始化SF
* 执行SF的run方法

```
int main(int, char**) {
    //设定surfaceflinger进程的binder线程池个数上限为4，并启动binder线程池
    ProcessState::self()->setThreadPoolMaxThreadCount(4);
    sp<ProcessState> ps(ProcessState::self());
    ps->startThreadPool();

    //实例化surfaceflinger
    sp<SurfaceFlinger> flinger =  new SurfaceFlinger();

    setpriority(PRIO_PROCESS, 0, PRIORITY_URGENT_DISPLAY);
    set_sched_policy(0, SP_FOREGROUND);

    //初始化
    flinger->init();

    //将服务注册到Service Manager
    sp<IServiceManager> sm(defaultServiceManager());
    sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false);

    // 运行在当前线程
    flinger->run();

    return 0;
}

```
#### 构造方法
继承BnSurfaceCompose
构造过程仅仅初始化了SurfaceFlinger的成员变量，同时调用了父类BnSurfaceComposer的构造函数。最后执行onFirstRef 走init方法来作一些初始化工作

```
SurfaceFlinger::SurfaceFlinger()
    :   BnSurfaceComposer(),
        mTransactionFlags(0),
        mTransactionPending(false),
        mAnimTransactionPending(false),
        mLayersRemoved(false),
        mRepaintEverything(0),
        mRenderEngine(NULL),
        mBootTime(systemTime()),
        mVisibleRegionsDirty(false),
        mHwWorkListDirty(false),
        mAnimCompositionPending(false),
        mDebugRegion(0),
        mDebugDDMS(0),
        mDebugDisableHWC(0),
        mDebugDisableTransformHint(0),
        mDebugInSwapBuffers(0),
        mLastSwapBufferTime(0),
        mDebugInTransaction(0),
        mLastTransactionTime(0),
        mBootFinished(false),
        mForceFullDamage(false),
        mPrimaryHWVsyncEnabled(false),
        mHWVsyncAvailable(false),
        mDaltonize(false),
        mHasColorMatrix(false),
        mHasPoweredOff(false),
        mFrameBuckets(),
        mTotalTime(0),
        mLastSwapTime(0)
{
    ALOGI("SurfaceFlinger is starting");
    char value[PROPERTY_VALUE_MAX];

    property_get("ro.bq.gpu_to_cpu_unsupported", value, "0");
    mGpuToCpuSupported = !atoi(value);

    property_get("debug.sf.showupdates", value, "0");
    mDebugRegion = atoi(value);

    property_get("debug.sf.ddms", value, "0");
    mDebugDDMS = atoi(value);       
}
```

#### SF.onFirstRef
由于SurfaceFlinger继承于RefBase类，同时实现了RefBase的onFirstRef()方法，因此在第一次引用SurfaceFlinger对象时，onFirstRef()函数自动被调用。初始化MessageQueue

```
void SurfaceFlinger::onFirstRef()
{
    mEventQueue.init(this);
}

```

#### MessageQueue
messageQueue对象的创建


```
void MessageQueue::init(const sp<SurfaceFlinger>& flinger)
{
    mFlinger = flinger;
    mLooper = new Looper(true);
    mHandler = new Handler(*this); 
}
class MessageQueue {
    class Handler : public MessageHandler {
        enum {
            eventMaskInvalidate     = 0x1,
            eventMaskRefresh        = 0x2,
            eventMaskTransaction    = 0x4
        };
        MessageQueue& mQueue;
        int32_t mEventMask;
    public:
        Handler(MessageQueue& queue) : mQueue(queue), mEventMask(0) { }
        virtual void handleMessage(const Message& message);
        void dispatchRefresh();
        void dispatchInvalidate();
        void dispatchTransaction();
    };
    ...
}
```

#### SurfaceFlinger.init
* 初始化 EGL
* 创建 HWComposer
* 初始化非虚拟显示屏
* 启动 EventThread 线程
* 启动开机动画

```
void SurfaceFlinger::init() {
    Mutex::Autolock _l(mStateLock);

    //初始化EGL，作为默认的显示
    mEGLDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    eglInitialize(mEGLDisplay, NULL, NULL);

    // 初始化硬件composer对象
    mHwc = new HWComposer(this, *static_cast<HWComposer::EventHandler *>(this));

    //获取RenderEngine引擎
    mRenderEngine = RenderEngine::create(mEGLDisplay, mHwc->getVisualID());

    //创建的EGL上下文
    mEGLContext = mRenderEngine->getEGLContext();

    //初始化非虚拟显示屏
    for (size_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
        DisplayDevice::DisplayType type((DisplayDevice::DisplayType)i);
        //建立已连接的显示设备
        if (mHwc->isConnected(i) || type==DisplayDevice::DISPLAY_PRIMARY) {
            bool isSecure = true;
            createBuiltinDisplayLocked(type);
            wp<IBinder> token = mBuiltinDisplays[i];

            sp<IGraphicBufferProducer> producer;
            sp<IGraphicBufferConsumer> consumer;
            //创建BufferQueue的生产者和消费者
            BufferQueue::createBufferQueue(&producer, &consumer,
                    new GraphicBufferAlloc());

            sp<FramebufferSurface> fbs = new FramebufferSurface(*mHwc, i, consumer);
            int32_t hwcId = allocateHwcDisplayId(type);
            //创建显示设备
            sp<DisplayDevice> hw = new DisplayDevice(this,
                    type, hwcId, mHwc->getFormat(hwcId), isSecure, token,
                    fbs, producer,
                    mRenderEngine->getEGLConfig());
            if (i > DisplayDevice::DISPLAY_PRIMARY) {
                hw->setPowerMode(HWC_POWER_MODE_NORMAL);
            }
            mDisplays.add(token, hw);
        }
    }

    getDefaultDisplayDevice()->makeCurrent(mEGLDisplay, mEGLContext);

    //当应用和sf的vsync偏移量一致时，则只创建一个EventThread线程
    if (vsyncPhaseOffsetNs != sfVsyncPhaseOffsetNs) {
        sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
                vsyncPhaseOffsetNs, true, "app");
        mEventThread = new EventThread(vsyncSrc);
        sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
                sfVsyncPhaseOffsetNs, true, "sf");
        mSFEventThread = new EventThread(sfVsyncSrc);
        mEventQueue.setEventThread(mSFEventThread);
    } else {
        //创建DispSyncSource对象
        sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
                vsyncPhaseOffsetNs, true, "sf-app");
        //创建线程EventThread 
        mEventThread = new EventThread(vsyncSrc);
        //设置EventThread 
        mEventQueue.setEventThread(mEventThread);
    }

    // 创建EventControl
    mEventControlThread = new EventControlThread(this);
    mEventControlThread->run("EventControl", PRIORITY_URGENT_DISPLAY);

    //当不存在HWComposer时，则设置软件vsync
    if (mHwc->initCheck() != NO_ERROR) {
        mPrimaryDispSync.setPeriod(16666667);
    }

    //初始化绘图状态
    mDrawingState = mCurrentState;

    //初始化显示设备
    initializeDisplays();

    //启动开机动画
    startBootAnim();
}
```

#### HWComposer构建
HWComposer代表着硬件显示设备，注册了VSYNC信号的回调。VSYNC信号本身是由显示驱动产生的， 在不支持硬件的VSYNC，则会创建“VSyncThread”线程来模拟定时VSYNC信号

```
HWComposer::HWComposer(
        const sp<SurfaceFlinger>& flinger,
        EventHandler& handler)
    : mFlinger(flinger),
      mFbDev(0), mHwc(0), mNumDisplays(1),
      mCBContext(new cb_context),
      mEventHandler(handler),
      mDebugForceFakeVSync(false)
{
    ...
    bool needVSyncThread = true;
    int fberr = loadFbHalModule(); //加载framebuffer的HAL层模块
    loadHwcModule(); //加载HWComposer模块

    //标记已分配的display ID
    for (size_t i=0 ; i<NUM_BUILTIN_DISPLAYS ; i++) {
        mAllocatedDisplayIDs.markBit(i);
    }

    if (mHwc) {
        if (mHwc->registerProcs) {
            mCBContext->hwc = this;
            mCBContext->procs.invalidate = &hook_invalidate;
            //VSYNC信号的回调方法
            mCBContext->procs.vsync = &hook_vsync;
            if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1))
                mCBContext->procs.hotplug = &hook_hotplug;
            else
                mCBContext->procs.hotplug = NULL;
            memset(mCBContext->procs.zero, 0, sizeof(mCBContext->procs.zero));
            //注册回调函数
            mHwc->registerProcs(mHwc, &mCBContext->procs);
        }

        //进入此处，说明已成功打开硬件composer设备，则不再需要vsync线程
        needVSyncThread = false;
        eventControl(HWC_DISPLAY_PRIMARY, HWC_EVENT_VSYNC, 0);
        ...
    }
    ...

    if (needVSyncThread) {
        //不支持硬件的VSYNC，则会创建线程来模拟定时VSYNC信号
        mVSyncThread = new VSyncThread(*this);
    }
}
```

#### 初始化显示设备
创建IGraphicBufferProducer和IGraphicBufferConsumer，以及FramebufferSurface，DisplayDevice对象。另外， 显示设备有3类：主设备，扩展设备，虚拟设备。其中前两个都是内置显示设备，故NUM_BUILTIN_DISPLAY_TYPES=2，

```
void SurfaceFlinger::init() {
    ...
    for (size_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
        DisplayDevice::DisplayType type((DisplayDevice::DisplayType)i);
        //建立已连接的显示设备
        if (mHwc->isConnected(i) || type==DisplayDevice::DISPLAY_PRIMARY) {
            bool isSecure = true;
            createBuiltinDisplayLocked(type);
            wp<IBinder> token = mBuiltinDisplays[i];

            sp<IGraphicBufferProducer> producer;
            sp<IGraphicBufferConsumer> consumer;
            //创建BufferQueue的生产者和消费者
            BufferQueue::createBufferQueue(&producer, &consumer,
                    new GraphicBufferAlloc());

            sp<FramebufferSurface> fbs = new FramebufferSurface(*mHwc, i, consumer);
            int32_t hwcId = allocateHwcDisplayId(type);
            //创建显示设备
            sp<DisplayDevice> hw = new DisplayDevice(this,
                    type, hwcId, mHwc->getFormat(hwcId), isSecure, token,
                    fbs, producer,
                    mRenderEngine->getEGLConfig());
            if (i > DisplayDevice::DISPLAY_PRIMARY) {
                hw->setPowerMode(HWC_POWER_MODE_NORMAL);
            }
            mDisplays.add(token, hw);
        }
    }
    ...
}
```

#### EventThread构造方法

```
EventThread::EventThread(const sp<VSyncSource>& src)
    : mVSyncSource(src),
      mUseSoftwareVSync(false),
      mVsyncEnabled(false),
      mDebugVsyncEnabled(false),
      mVsyncHintSent(false) {

    for (int32_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
        mVSyncEvent[i].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
        mVSyncEvent[i].header.id = 0;
        mVSyncEvent[i].header.timestamp = 0;
        mVSyncEvent[i].vsync.count =  0;
    }
    struct sigevent se;
    se.sigev_notify = SIGEV_THREAD;
    se.sigev_value.sival_ptr = this;
    se.sigev_notify_function = vsyncOffCallback;
    se.sigev_notify_attributes = NULL;
    timer_create(CLOCK_MONOTONIC, &se, &mTimerId);
}

void EventThread::onFirstRef() {
    //运行EventThread线程
    run("EventThread", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);
}
```

#### ET.threadLoop

```
bool EventThread::threadLoop() {
    DisplayEventReceiver::Event event;
    Vector< sp<EventThread::Connection> > signalConnections;
    // 等待事件
    signalConnections = waitForEvent(&event);

    //分发事件给所有的监听者
    const size_t count = signalConnections.size();
    for (size_t i=0 ; i<count ; i++) {
        const sp<Connection>& conn(signalConnections[i]);
        //传递事件【见小节3.10】
        status_t err = conn->postEvent(event);
        if (err == -EAGAIN || err == -EWOULDBLOCK) {
            //可能此时connection已满，则直接抛弃事件
            ALOGW("EventThread: dropping event (%08x) for connection %p",
                    event.header.type, conn.get());
        } else if (err < 0) {
            //发生致命错误，则清理该连接
            removeDisplayEventConnection(signalConnections[i]);
        }
    }
    return true;
}
```

#### ET.waitForEvent
EventThread线程，进入mCondition的wait()方法，等待唤醒

```
Vector< sp<EventThread::Connection> > EventThread::waitForEvent(
        DisplayEventReceiver::Event* event)
{
    Mutex::Autolock _l(mLock);
    Vector< sp<EventThread::Connection> > signalConnections;

    do {
        bool eventPending = false;
        bool waitForVSync = false;

        size_t vsyncCount = 0;
        nsecs_t timestamp = 0;
        for (int32_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
            timestamp = mVSyncEvent[i].header.timestamp;
            if (timestamp) {
                *event = mVSyncEvent[i];
                mVSyncEvent[i].header.timestamp = 0;
                vsyncCount = mVSyncEvent[i].vsync.count;
                break;
            }
        }

        if (!timestamp) {
            //没有vsync事件，则查看其它事件
            eventPending = !mPendingEvents.isEmpty();
            if (eventPending) {
                //存在其它事件可用于分发
                *event = mPendingEvents[0];
                mPendingEvents.removeAt(0);
            }
        }

        //查找正在等待事件的连接
        size_t count = mDisplayEventConnections.size();
        for (size_t i=0 ; i<count ; i++) {
            sp<Connection> connection(mDisplayEventConnections[i].promote());
            if (connection != NULL) {
                bool added = false;
                if (connection->count >= 0) {
                    //需要vsync事件，由于至少存在一个连接正在等待vsync
                    waitForVSync = true;
                    if (timestamp) {
                        if (connection->count == 0) {
                            connection->count = -1;
                            signalConnections.add(connection);
                            added = true;
                        } else if (connection->count == 1 ||
                                (vsyncCount % connection->count) == 0) {
                            signalConnections.add(connection);
                            added = true;
                        }
                    }
                }

                if (eventPending && !timestamp && !added) {
                    //没有vsync事件需要处理(timestamp==0),但存在pending消息
                    signalConnections.add(connection);
                }
            } else {
                //该连接已死亡，则直接清理
                mDisplayEventConnections.removeAt(i);
                --i; --count;
            }
        }

        if (timestamp && !waitForVSync) {
            //接收到VSYNC，但没有client需要它，则直接关闭VSYNC
            disableVSyncLocked();
        } else if (!timestamp && waitForVSync) {
            //至少存在一个client，则需要使能VSYNC
            enableVSyncLocked();
        }

        if (!timestamp && !eventPending) {
            if (waitForVSync) {
                bool softwareSync = mUseSoftwareVSync;
                nsecs_t timeout = softwareSync ? ms2ns(16) : ms2ns(1000);
                if (mCondition.waitRelative(mLock, timeout) == TIMED_OUT) {
                    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
                    mVSyncEvent[0].header.id = DisplayDevice::DISPLAY_PRIMARY;
                    mVSyncEvent[0].header.timestamp = systemTime(SYSTEM_TIME_MONOTONIC);
                    mVSyncEvent[0].vsync.count++;
                }
            } else {
                //不存在对vsync感兴趣的连接，即将要进入休眠
                mCondition.wait(mLock);
            }
        }
    } while (signalConnections.isEmpty());

    //到此处，则保证存在timestamp以及连接
    return signalConnections;
}
```

#### MQ.setEvenetThread
设置EventThread，并监听BitTube
创建一个BitTube对象mEventTube
创建一个EventConnection

```
void MessageQueue::setEventThread(const sp<EventThread>& eventThread)
{
    mEventThread = eventThread;
    //创建连接
    mEvents = eventThread->createEventConnection();
    //获取BitTube对象
    mEventTube = mEvents->getDataChannel();
    //监听BitTube，一旦有数据到来则调用cb_eventReceiver()
    mLooper->addFd(mEventTube->getFd(), 0, Looper::EVENT_INPUT,
            MessageQueue::cb_eventReceiver, this);
}
```

#### SF.run
主线程进入waitMessage状态

```
void SurfaceFlinger::run() {
    do {
        //不断循环地等待事件
        waitForEvent();
    } while (true);
}

void SurfaceFlinger::waitForEvent() {
    mEventQueue.waitMessage(); 
}

void MessageQueue::waitMessage() {
    do {
        IPCThreadState::self()->flushCommands();
        int32_t ret = mLooper->pollOnce(-1);
        ...
    } while (true);
}
```



### Vsync信号处理
HWComposer对象创建过程，会注册一些回调方法，当硬件产生VSYNC信号时，则会回调hook_vsync()方法。

#### HWComposer.hook_vsync
hook 监听Vysnc

```
void HWComposer::hook_vsync(const struct hwc_procs* procs, int disp,
        int64_t timestamp) {
    cb_context* ctx = reinterpret_cast<cb_context*>(
            const_cast<hwc_procs_t*>(procs));
    ctx->hwc->vsync(disp, timestamp); 】
}
```

#### HWComposer.vsync
Vsync信号回调，执行SF的onVSyncReceived方法

```
void HWComposer::vsync(int disp, int64_t timestamp) {
    if (uint32_t(disp) < HWC_NUM_PHYSICAL_DISPLAY_TYPES) {
        {
            Mutex::Autolock _l(mLock);
            if (timestamp == mLastHwVSync[disp]) {
                return; //忽略重复的VSYNC信号
            }
            mLastHwVSync[disp] = timestamp;
        }
        //
        mEventHandler.onVSyncReceived(disp, timestamp);
    }
}
```

#### SF.onVSyncReceived

```
void SurfaceFlinger::onVSyncReceived(int type, nsecs_t timestamp) {
    bool needsHwVsync = false;

    {
        Mutex::Autolock _l(mHWVsyncLock);
        if (type == 0 && mPrimaryHWVsyncEnabled) {
            // 此处mPrimaryDispSync为DispSync类 kai是分析DispSync
            needsHwVsync = mPrimaryDispSync.addResyncSample(timestamp);
        }
    }

    if (needsHwVsync) {
        enableHardwareVsync();
    } else {
        disableHardwareVsync(false);
    }
}
```

#### DispSync构建

```
DispSync::DispSync() :
        mRefreshSkipCount(0),
        mThread(new DispSyncThread()) {
    // 运行在DispSync线程
    mThread->run("DispSync", PRIORITY_URGENT_DISPLAY + PRIORITY_MORE_FAVORABLE);

    reset();
    beginResync();
    ...
}
```

#### DispSyncThread线程
线程”DispSync”停留在mCond的wait()过程，等待被唤醒
当收集到Vsync信号后开始回调onDispSyncEvent方法

```
virtual bool threadLoop() {
     status_t err;
     nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
     nsecs_t nextEventTime = 0;

     while (true) {
         Vector<CallbackInvocation> callbackInvocations;

         nsecs_t targetTime = 0;
         { // Scope for lock
             Mutex::Autolock lock(mMutex);
             if (mStop) {
                 return false;
             }

             if (mPeriod == 0) {
                 err = mCond.wait(mMutex);
                 continue;
             }

             nextEventTime = computeNextEventTimeLocked(now);
             targetTime = nextEventTime;
             bool isWakeup = false;

             if (now < targetTime) {
                 err = mCond.waitRelative(mMutex, targetTime - now);
                 if (err == TIMED_OUT) {
                     isWakeup = true;
                 } else if (err != NO_ERROR) {
                     return false;
                 }
             }

             now = systemTime(SYSTEM_TIME_MONOTONIC);

             if (isWakeup) {
                 mWakeupLatency = ((mWakeupLatency * 63) +
                         (now - targetTime)) / 64;
                 if (mWakeupLatency > 500000) {
                     mWakeupLatency = 500000;
                 }
             }
             //收集vsync信号的所有回调方法
             callbackInvocations = gatherCallbackInvocationsLocked(now);
         }

         if (callbackInvocations.size() > 0) {
             //回调所有对象的onDispSyncEvent方法
             fireCallbackInvocations(callbackInvocations);
         }
     }

     return false;
 }   
 
void fireCallbackInvocations(const Vector<CallbackInvocation>& callbacks) {
    for (size_t i = 0; i < callbacks.size(); i++) {
        //执行DSP 的接受方法
        callbacks[i].mCallback->onDispSyncEvent(callbacks[i].mEventTime);
    }
}
```
####  DispSyncSource.onDispSyncEvent

```
virtual void onDispSyncEvent(nsecs_t when) {
    sp<VSyncSource::Callback> callback;
    {
       Mutex::Autolock lock(mCallbackMutex);
       callback = mCallback;
    }

    if (callback != NULL) {
        // 执行 EventThread接受Vsync方法
      callback->onVSyncEvent(when);
    }
}
```

#### EventThread.onVsyncEvent

```
void EventThread::onVSyncEvent(nsecs_t timestamp) {
    Mutex::Autolock _l(mLock);
    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
    mVSyncEvent[0].header.id = 0;
    mVSyncEvent[0].header.timestamp = timestamp;
    mVSyncEvent[0].vsync.count++;
    mCondition.broadcast(); //唤醒EventThread线程
}

```
#### EventThread.postEvent

```
status_t EventThread::Connection::postEvent(
        const DisplayEventReceiver::Event& event) {
    ssize_t size = DisplayEventReceiver::sendEvents(mChannel, &event, 1);
    return size < 0 ? status_t(size) : status_t(NO_ERROR);
}
```

####  DisplayEventReceiver.sendEvents
MQ.setEvenetThread监听BitTube，此处调用BitTube来sendObjects。一旦收到数据，则调用MQ.cb_eventReceiver()方法

```
ssize_t DisplayEventReceiver::sendEvents(const sp<BitTube>& dataChannel,
        Event const* events, size_t count)
{
    return BitTube::sendObjects(dataChannel, events, count);
}
```
#### MQ.cb_eventReceiver

```
int MessageQueue::cb_eventReceiver(int fd, int events, void* data) {
    MessageQueue* queue = reinterpret_cast<MessageQueue *>(data);
    return queue->eventReceiver(fd, events);
}
```
#### MQ.eventReceiver
接受事件，分发dispatchInvalidate、dispatchRefresh
消息接收，执行SF的onMessageReceived

```
int MessageQueue::eventReceiver(int /*fd*/, int /*events*/) {
    ssize_t n;
    DisplayEventReceiver::Event buffer[8];
    while ((n = DisplayEventReceiver::getEvents(mEventTube, buffer, 8)) > 0) {
        for (int i=0 ; i<n ; i++) {
            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
#if INVALIDATE_ON_VSYNC
                mHandler->dispatchInvalidate();
#else
                mHandler->dispatchRefresh(); 
#endif
                break;
            }
        }
    }
    return 1;
}
// 发送refresh
void MessageQueue::Handler::dispatchRefresh() {
    if ((android_atomic_or(eventMaskRefresh, &mEventMask) & eventMaskRefresh) == 0) {
        //发送消息，则进入handleMessage过程【
        mQueue.mLooper->sendMessage(this, Message(MessageQueue::REFRESH));
    }
}

// MQ的消息接受处理
void MessageQueue::Handler::handleMessage(const Message& message) {
    switch (message.what) {
        case INVALIDATE:
            android_atomic_and(~eventMaskInvalidate, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
        case REFRESH:
            android_atomic_and(~eventMaskRefresh, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
        case TRANSACTION:
            android_atomic_and(~eventMaskTransaction, &mEventMask);
            mQueue.mFlinger->onMessageReceived(message.what);
            break;
    }
}


```
#### SF.onMessageReceived

```
void SurfaceFlinger::onMessageReceived(int32_t what) {
    ATRACE_CALL();
    switch (what) {
        case MessageQueue::TRANSACTION: {
            handleMessageTransaction();
            break;
        }
        case MessageQueue::INVALIDATE: {
            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate(); 
            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded) {
                signalRefresh();
            }
            break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh(); // 执行refresh流程
            break;
        }
    }
}

```

#### SF.handleMessageRefresh

```
void SurfaceFlinger::handleMessageRefresh() {
    ATRACE_CALL();
    // 处理显示设备与 layers 的改变
    preComposition(); 
    // 重建所有layer，根据z轴排序
    rebuildLayerStacks();
    // 更新 HWComposer涂层
    setUpHWComposer();
    doDebugFlashRegions();
    // 生成OpenGL 纹理图像
    doComposition();
    // 将图像传递到物理屏幕
    postComposition();
}
```


-------
推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)
参考：
[一篇文章看明白 Android 图形系统 Surface 与 SurfaceFlinger 之间的关系](https://blog.csdn.net/freekiteyu/article/details/79483406)
[SurfaceFlinger启动篇](http://gityuan.com/2017/02/11/surface_flinger/)

