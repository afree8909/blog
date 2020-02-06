
---
tags: 
- 源码
- Surface
categories:
- [Android, 系统]
---


>基于API 23

![](https://upload-images.jianshu.io/upload_images/9696036-b7271e93c8490bde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### SurfaceComposerClient的创建过程

#### WMS.addWinodw
WMS添加window过程，最后会执行到session.windowAddedLocked

```
//WMS
public int addWindow(Session session, IWindow client, int seq, WindowManager.LayoutParams attrs, int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets, Rect outOutsets, InputChannel outInputChannel) {
    ...
    win.attach(); // WindowState
    ...
}

// WindowState.attch
void attach() {
    mSession.windowAddedLocked();
}
```

#### Session.windowAddedLocked
创建 SurfaceSession 对象，并将当前 Session 添加到 WMS.mSessions 成员变量

```
void windowAddedLocked() {
    if (mSurfaceSession == null) {
        mSurfaceSession = new SurfaceSession();
        mService.mSessions.add(this);
        if (mLastReportedAnimatorScale != mService.getCurrentAnimatorScale()) {
            mService.dispatchNewAnimatorScaleLocked(this);
        }
    }
    mNumWindow++;
}
```

#### android_view_SurfaceSession.cpp
SurfaceSession 的创建会调用 JNI，在 JNI 调用 nativeCreate()
构造了一个SurfaceComposerClient对象。并返回它的指针。这个对象一个应用程序就有一个，它是应用程序与SurfaceFlinger沟通的桥梁

```
// SurfaceSeesion
public SurfaceSession() {
   mNativeClient = nativeCreate();
}
// JNI
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    // 创建 SurfaceComposeClient 空实现，查看onFirstRef
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}

```

#### SurfaceComposerClient.onFirstRef
通过SurfaceFlinger创造了一个Client对象，每一个APP都有一个Client对象向对应，通过这个代理对象可以跟SurfaceFlinger通信

```
void SurfaceComposerClient::onFirstRef() {
    ....
    sp<ISurfaceComposerClient> conn;
    //sf 就是SurfaceFlinger
    conn = (rootProducer != nullptr) ? sf->createScopedConnection(rootProducer) :
            sf->createConnection();
    ...
}


```

#### SurfaceFlinger.cpp
构造了一个Client对象，Client实现了ISurfaceComposerClient接口。是一个可以跨进程通信的aidl对象。除此之外它还可以创建Surface，并且维护一个应用程序的所有Layer


```
sp<ISurfaceComposerClient> SurfaceFlinger::createConnection(){
  sp<ISurfaceComposerClient> bclient;
  sp<Client> client(new Client(this));
  return bclient;
}
```

#### Client.h
```
class Client : public BnSurfaceComposerClient
{
public:
    
    void attachLayer(const sp<IBinder>& handle, const sp<Layer>& layer);
    void detachLayer(const Layer* layer);
    
};

```



### Surface的创建
一个ViewRootImpl就对应一个Surface

```
final Surface mSurface = new Surface();

public Surface() {
    // 空构造函数，需要继续追看Surface赋值过程
}

```

#### ViewRootImpl.relayoutWindow
直接看ViewRootImpl的绘制流程

```
private void performTraversals() {
    finalView host = mView; //mView是一个Window的根View，对于Activity来说就是DecorView
    ...
    relayoutWindow(params, viewVisibility, insetsPending);
    ...
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...         
    performLayout(lp, mWidth, mHeight);
    ...
    performDraw();
    ...
}
// 重布window
private int relayoutWindow(WindowManager.LayoutParams params, ...) throws RemoteException {
    ...
    int relayoutResult = mWindowSession.relayout(mWindow,..., mSurface);
    ...
}

```

#### WindowManagerService.relayoutWindow
winAnimator.createSurfaceLocked实际上是创建了一个SurfaceControl。即上面是先构造SurfaceControl，然后在构造Surface

```
public int relayoutWindow(Session session, IWindow client....Surface outSurface){ 
    ...
    result = createSurfaceControl(outSurface, result, win, winAnimator);  
    ...
}

private int createSurfaceControl(Surface outSurface, int result, WindowState win,WindowStateAnimator winAnimator) {
    ...
    surfaceController = winAnimator.createSurfaceLocked(win.mAttrs.type, win.mOwnerUid);
    ...
    surfaceController.getSurface(outSurface);
}

```

#### SurfaceControl的创建
通过SurfaceControl的构造函数创建了一个SurfaceControl对象,这个对象的作用其实就是负责维护Surface,Surface其实也是由这个对象负责创建的

```
long mNativeObject; //成员指针变量，指向native创建的SurfaceControl

private SurfaceControl(...){
    ...
    mNativeObject = nativeCreate(session, name, w, h, format, flags,
        parent != null ? parent.mNativeObject : 0, windowType, ownerUid);
}

```

#### android_view_SurfaceControl.cpp


```
static jlong nativeCreate(JNIEnv* env, ...) {
    //这个client为前面创建的SurfaceComposerClinent
    sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj)); 
    //创建成功之后，这个指针会指向新创建的SurfaceControl
    sp<SurfaceControl> surface;
    status_t err = client->createSurface(String8(name.c_str()), w, h, format, &surface, flags, parent, windowType, ownerUid);
    ...
    return reinterpret_cast<jlong>(surface.get()); //返回这个SurfaceControl的地址
}

```

#### SurfaceComposerClient.createSurface
创建时传入了一个对象 sp<IGraphicBufferProducer> gbp, 后面会说吗应用所渲染的每一帧，实际上都会添加到IGraphicBufferProducer中，来等待SurfaceFlinger的渲染

```
//outSurface会指向新创建的SurfaceControl
status_t SurfaceComposerClient::createSurface(...sp<SurfaceControl>* outSurface..) 
{
    sp<IGraphicBufferProducer> gbp; 
    ...
    // 调用 之前缓存的client进行创建
    err = mClient->createSurface(name, w, h, format, flags, parentHandle, windowType, ownerUid, &handle, &gbp);
    if (err == NO_ERROR) {
        //SurfaceControl创建成功, 指针赋值
        sur = new SurfaceControl(this, handle, gbp, true);
    }
    return sur;
}

```

#### Client.createSurface
Client将应用程序创建Surface的请求转换为异步消息投递到SurfaceFlinger的消息队列中，将创建Surface的任务转交给SurfaceFlinger，因为同一时刻可以有多个应用程序请求SurfaceFlinger为其创建Surface，通过消息队列可以实现请求排队，然后SurfaceFlinger依次为应用程序创建Surface

```

status_t Client::createSurface(
        const String8& name,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp)
{
    ...

    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
            name, this, w, h, format, flags, handle, gbp);
    mFlinger->postMessageSync(msg);
    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
}
```

#### SurfaceFlinger.createLayer
该函数中根据flag创建不同的Layer，Layer用于标示一个图层。
除了SurfaceFlinger需要统一管理系统中创建的所有Layer对象外，专门为每个应用程序进程服务的Client也需要统一管理当前应用程序进程所创建的Layer，因此在addClientLayer函数里还会通过Client::attachLayer将创建的Layer和该类对应的handle以键值对的方式保存到Client的成员变量mLayers表中


```
status_t SurfaceFlinger::createLayer(const String8& name,const sp<Client>& client...)
{
    status_t result = NO_ERROR;
    //创建的layer
    sp<Layer> layer; 
    switch (flags & ISurfaceComposerClient::eFXSurfaceMask) {
        case ISurfaceComposerClient::eFXSurfaceNormal:
            result = createBufferLayer(client,
                    uniqueName, w, h, flags, format,
                    handle, gbp, &layer);
            break;
            ... //Layer 其它情况的创建
    }
    ...
    //这个layer和client相关联, 添加到Client的mLayers集合中
    result = addClientLayer(client, *handle, *gbp, layer, *parent);  
    ...
    return result;
}

```
#### SurfaceFlinger.createNormalLayer
SurfaceFlinger为应用程序创建好Layer后，需要统一管理这些Layer对象，因此通过函数addClientLayer将创建的Layer保存到当前State的Z秩序列表layersSortedByZ中，同时将这个Layer所对应的IGraphicBufferProducer本地Binder对象gbp保存到SurfaceFlinger的成员变量mGraphicBufferProducerList中

```
status_t SurfaceFlinger::createNormalLayer(const sp<Client>& client,
        const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer)
{
    // initialize the surfaces
    switch (format) {
    case PIXEL_FORMAT_TRANSPARENT:
    case PIXEL_FORMAT_TRANSLUCENT:
        format = PIXEL_FORMAT_RGBA_8888;
        break;
    case PIXEL_FORMAT_OPAQUE:
        format = PIXEL_FORMAT_RGBX_8888;
        break;
    }

    *outLayer = new Layer(this, client, name, w, h, flags);
    status_t err = (*outLayer)->setBuffers(w, h, format, flags);
    if (err == NO_ERROR) {
        *handle = (*outLayer)->getHandle();
        *gbp = (*outLayer)->getProducer();
    }

    ALOGE_IF(err, "createNormalLayer() failed (%s)", strerror(-err));
    return err;
}
```

#### Layer.cpp

```
void Layer::onFirstRef() {
    // Creates a custom BufferQueue for SurfaceFlingerConsumer to use
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferConsumer> consumer;
    BufferQueue::createBufferQueue(&producer, &consumer);
    mProducer = new MonitoredProducer(producer, mFlinger);
    mSurfaceFlingerConsumer = new SurfaceFlingerConsumer(consumer, mTextureName);
    mSurfaceFlingerConsumer->setConsumerUsageBits(getEffectiveUsage(0));
    mSurfaceFlingerConsumer->setContentsChangedListener(this);
    mSurfaceFlingerConsumer->setName(mName);

#ifdef TARGET_DISABLE_TRIPLE_BUFFERING
#warning "disabling triple buffering"
    mSurfaceFlingerConsumer->setDefaultMaxBufferCount(2);
#else
    mSurfaceFlingerConsumer->setDefaultMaxBufferCount(3);
#endif

    const sp<const DisplayDevice> hw(mFlinger->getDefaultDisplayDevice());
    updateTransformHint(hw);
}
```

#### BufferQueue.cpp
* BufferQueue是一个服务中心，IGraphicBufferProducer和IGraphicBufferConsumer
所需要使用的buffer必须要通过它来管理。比如说当IGraphicBufferProducer想要获取一个buffer时，它不能越过BufferQueue直接与IGraphicBufferConsumer进行联系，反之亦然。
* IGraphicBufferProducer就是“填充”buffer空间的人，通常情况下是应用程序。因为应用程序不断地刷新UI，从而将产生的显示数据源源不断地写到buffer中。当IGraphicBufferProducer需要使用一块buffer时，它首先会向中介BufferQueue发起dequeueBuffer申请，然后才能对指定的buffer进行操作。此时buffer就只属于IGraphicBufferProducer一个人的了，它可以对buffer进行任何必要的操作，而IGraphicBufferConsumer此刻绝不能操作这块buffer。当IGraphicBufferProducer认为一块buffer已经写入完成后，它进一步调用queueBuffer函数。从字面上看这个函数是“入列”的意思，形象地表达了buffer此时的操作，把buffer归还到BufferQueue的队列中。一旦queue成功后，buffer的owner也就随之改变为BufferQueue了
* IGraphicBufferConsumer是与IGraphicBufferProducer相对应的，它的操作同样受到BufferQueue的管控。当一块buffer已经就绪后，IGraphicBufferConsumer就可以开始工作了。



```
void BufferQueue::createBufferQueue(sp<IGraphicBufferProducer>* outProducer,
        sp<IGraphicBufferConsumer>* outConsumer,
        const sp<IGraphicBufferAlloc>& allocator) {

    sp<BufferQueueCore> core(new BufferQueueCore(allocator));
    *outProducer = producer;
    *outConsumer = consumer;
}

```



#### 从SurfaceControl中获取Surface
上面完成了 surfaceController的创建跟踪，下面分析从surfaceController获取surface过程

```
private int createSurfaceControl(Surface outSurface, int result, WindowState win,WindowStateAnimator winAnimator) {
    ...
    // 上面完成了 surfaceController的创建
    SurfaceControl surfaceController = winAnimator.createSurfaceLocked();
    // 下面分析 surfaceController获取surface
    outSurface.copyFrom(surfaceControl);
}

```

#### android_view_Surface.nativeCreateFromSurfaceControl
JNI构建方法获取到一个指针，

```
static jlong nativeCreateFromSurfaceControl(JNIEnv* env, jclass clazz, jlong surfaceControlNativeObj) {
    // 把java指针转化内native指针
    sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
    // 直接构造一个Surface，指向 ctrl->getSurface()
    sp<Surface> surface(ctrl->getSurface());
    if (surface != NULL) {
        surface->incStrong(&sRefBaseOwner); //强引用
    }
    return reinterpret_cast<jlong>(surface.get());
}

```

#### SurfaceControl.getSurface
创建一个native层的Surface对象，并将该对象指针返回给Java层的Surface，从而建立Java层的Surface和native层Surface的关联关系
另外Surface和SurfaceControl都持有mGraphicBufferProducer用于操作位于SurfaceFlinger中的图形buffer

```
sp<Surface> SurfaceControl::getSurface() const
{
    Mutex::Autolock _l(mLock);
    if (mSurfaceData == 0) {
         mSurfaceData = new Surface(mGraphicBufferProducer, false);
    }
    return mSurfaceData;
}



```

-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)

