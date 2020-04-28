
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145839.png
tags: 
- 源码
- Surface
categories:
- [Android, 系统]
---


>基于API 23

前篇：[Android系统_Surface创建流程分析](https://www.jianshu.com/p/956db9044cd8)

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145839.png)


### 申请Buffer

#### ViewRootImpl.draw
执行遍历 > 执行Draw > draw方法 > 软件绘制流程

```

private void performTraversals() {
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

private void performDraw() {
    draw(fullRedrawNeeded);
}

private void draw(boolean fullRedrawNeeded) {
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        // 硬件加速 ，绘制绘制流程
        if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
             ...
            mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
        } else {
          ...
           // 软件绘制 流程
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
    }
}
```

#### ViewRootImpl.drawSoftware
主要三个步骤 申请Buffer、绘制、绘制完成

```
private boolean drawSoftware(Surface surface, AttachInfo attachInfo,...) {
    // Draw with software renderer.
    final Canvas canvas;
    ...
    canvas = mSurface.lockCanvas(dirty);  //申请Buffer
    ...
    mView.draw(canvas);  //绘制
    ...
    mSurface.unlockCanvasAndPost(canvas);  //绘制完成
}
```

#### Surface.lockCanvas
执行JNI层的nativeLockCanvas，mNativeObject就是native层Surface指针

```
public Canvas lockCanvas(Rect inOutDirty)
       throws Surface.OutOfResourcesException, IllegalArgumentException {
   synchronized (mLock) {
       checkNotReleasedLocked();
       
       mLockedObject = nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
       return mCanvas;
   }
}
```
#### android_view_Surface.cpp
创建一个Rect对象，确定要重绘制的区域，执行Surface的lock方法来申请bugger，然后新建了一个SKBitmap，设置了内存地址，并把这个bitmap放入了Canvas中

```
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject)); //转换指针
    ...
    Rect dirtyRect;
    Rect* dirtyRectPtr = NULL;

    if (dirtyRectObj) {
        dirtyRect.left   = env->GetIntField(dirtyRectObj, gRectClassInfo.left);
        dirtyRect.top    = env->GetIntField(dirtyRectObj, gRectClassInfo.top);
        dirtyRect.right  = env->GetIntField(dirtyRectObj, gRectClassInfo.right);
        dirtyRect.bottom = env->GetIntField(dirtyRectObj, gRectClassInfo.bottom);
        dirtyRectPtr = &dirtyRect;
    }
    
    // 申请内存Buffer
    ANativeWindow_Buffer outBuffer;
    status_t err = surface->lock(&outBuffer, dirtyRectPtr);
    
    ...
    SkImageInfo info = SkImageInfo::Make(outBuffer.width, outBuffer.height,
                                     convertPixelFormat(outBuffer.format),
                                     kPremul_SkAlphaType);               
    
    // 新建一个SkBitmap 并进行一系列设置
    SkBitmap bitmap;
    ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
    bitmap.setInfo(info, bpr);
    if (outBuffer.width > 0 && outBuffer.height > 0) {
        bitmap.setPixels(outBuffer.bits);
    } else {
        // be safe with an empty bitmap.
        bitmap.setPixels(NULL);
    }
    // 把创建的bitmap 设置到Canvas中
    Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
    nativeCanvas->setBitmap(bitma
    if (dirtyRectPtr) {
        nativeCanvas->clipRect(dirtyRect.left, dirtyRect.top,
                dirtyRect.right, dirtyRect.bottom);
    
    if (dirtyRectObj) {
        env->SetIntField(dirtyRectObj, gRectClassInfo.left,   dirtyRect.left);
        env->SetIntField(dirtyRectObj, gRectClassInfo.top,    dirtyRect.top);
        env->SetIntField(dirtyRectObj, gRectClassInfo.right,  dirtyRect.right);
        env->SetIntField(dirtyRectObj, gRectClassInfo.bottom, dirtyRect.bottom);
    
    // Create another reference to the surface and return it.  This reference
    // should be passed to nativeUnlockCanvasAndPost in place of mNativeObject,
    // because the latter could be replaced while the surface is locked.
    sp<Surface> lockedSurface(surface);
    lockedSurface->incStrong(&sRefBaseOwner);
    return (jlong) lockedSurface.get();    
}
```

#### Surface.cpp
通过dequeueBuffer获取一个ANativeWindowBuffer，之后构造一个GraphicBuffer，这个bugger用来传递绘制的元数据
BufferQueueProducer中dequeueBuffer，将分配的内存放到mSlots中，outSlot就是给应用进程mSlots的序号
BufferQueueProducer中requestBuffer，根据序号，从mSlots拿到buffer

```
status_t Surface::lock(ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds){

    ANativeWindowBuffer* out; int fenceFd = -1;
    status_t err = dequeueBuffer(&out, &fenceFd);  //从 GraphicBufferProduce 中 拿出来一个 buffer 
    
    if (err == NO_ERROR) {
        sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));//放到backBuffer中
        const Rect bounds(backBuffer->width, backBuffer->height);
 
        .......
 
        void* vaddr;
        status_t res = backBuffer->lockAsync(//把buffer的handle中的地址传到vaddr中
                GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
                newDirtyRegion.bounds(), &vaddr, fenceFd);
 
        if (res != 0) {
            err = INVALID_OPERATION;
        } else {
            mLockedBuffer = backBuffer;
            outBuffer->width  = backBuffer->width;
            outBuffer->height = backBuffer->height;
            outBuffer->stride = backBuffer->stride;
            outBuffer->format = backBuffer->format;
            outBuffer->bits   = vaddr;//buffer地址
        }


    return err // 返回GraphicBuffer
}

int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {

    ...
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, reqWidth, reqHeight,
                                                            reqFormat, reqUsage, &mBufferAge,
                                                            enableFrameTimestamps ? &frameTimestamps
                                                                                  : nullptr);
                                                                                  
                                                                        
     if ((result & IGraphicBufferProducer::BUFFER_NEEDS_REALLOCATION) || gbuf == 0) {
        result = mGraphicBufferProducer->requestBuffer(buf, &gbuf);//根据需要拿到buffer
    }

    sp<GraphicBuffer>& gbuf(mSlots[buf].buffer);
    *buffer = gbuf.get();
    
    return OK;                                                                            
                                                                                                                                    
}

```
### View绘制
#### View.draw
执行[View的绘制流程](https://www.jianshu.com/p/9b759b4a1aa5) 
下面看常见的 canvas.drawRect

```
public void drawRect(float left, float top, float right, float bottom, @NonNull Paint paint) {
   native_drawRect(mNativeCanvasWrapper, left, top, right, bottom, paint.getNativeInstance());
}
```
#### android_graphics_Canvas.cpp

```

static void drawRect(JNIEnv* env, jobject, jlong canvasHandle, jfloat left, jfloat top,
                     jfloat right, jfloat bottom, jlong paintHandle) {
    const Paint* paint = reinterpret_cast<Paint*>(paintHandle);
    get_canvas(canvasHandle)->drawRect(left, top, right, bottom, *paint);

}
```

#### SkiaCanvas.drawRect
执行Skia库的绘制方法

```
void SkiaCanvas::drawRect(float left, float top, float right, float bottom,
        const SkPaint& paint) {
    mCanvas->drawRectCoords(left, top, right, bottom, paint);
 
}
```

#### SkCanvas.drawRectCoords
执行 到SkBitmap写入Buffer数据

```

void SkCanvas::drawRectCoords(SkScalar left, SkScalar top,
                              SkScalar right, SkScalar bottom,
                              const SkPaint& paint) {
    TRACE_EVENT0("disabled-by-default-skia", "SkCanvas::drawRectCoords()");
    SkRect  r;
 
    r.set(left, top, right, bottom);
    this->drawRect(r, paint);

}


void SkCanvas::DrawRect(const SkDraw& draw, const SkPaint& paint,
                        const SkRect& r, SkScalar textSize) {
    if (paint.getStyle() == SkPaint::kFill_Style) {
        // fDevice 即SKBitmap ，从而实现将数据写入buffer
        draw.fDevice->drawRect(draw, r, paint);
    } else {
        SkPaint p(paint);
        p.setStrokeWidth(SkScalarMul(textSize, paint.getStrokeWidth()));
        draw.fDevice->drawRect(draw, r, p);
    }

}
```
### View绘制完成后通知刷新
#### Surface.unlockCanvasAndPost(canvas)

```
public void unlockCanvasAndPost(Canvas canvas) {
   synchronized (mLock) {
       checkNotReleasedLocked();

       if (mHwuiContext != null) {
           mHwuiContext.unlockAndPost(canvas);
       } else {
            // 走软件绘制流程
           unlockSwCanvasAndPost(canvas);
       }
   }
}
```
#### android_view.Surface.nativeUnlockCanvasAndPost
设置一个空的SkBitmap
然后执行Surface的unlockANdPost函数

```
private void unlockSwCanvasAndPost(Canvas canvas) {
   if (canvas != mCanvas) {
       throw new IllegalArgumentException("canvas object must be the same instance that "
               + "was previously returned by lockCanvas");
   }
   if (mNativeObject != mLockedObject) {
       Log.w(TAG, "WARNING: Surface's mNativeObject (0x" +
               Long.toHexString(mNativeObject) + ") != mLockedObject (0x" +
               Long.toHexString(mLockedObject) +")");
   }
   if (mLockedObject == 0) {
       throw new IllegalStateException("Surface was not locked");
   }
   try {
       nativeUnlockCanvasAndPost(mLockedObject, canvas);
   } finally {
       nativeRelease(mLockedObject);
       mLockedObject = 0;
   }
}

```

#### Surface.cpp.unlockAndPost
解除buffer锁定，执行queueBuffer最后执行到GraphicBufferProducer的queueBuffer函数，将buffer清除

```
status_t Surface::unlockAndPost()

{
   
    int fd = -1;
    status_t err = mLockedBuffer->unlockAsync(&fd);//通过Gralloc模块，最后是操作的ioctl
     
    err = queueBuffer(mLockedBuffer.get(), fd);
 
    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = 0;
    return err;

}
```

#### Surface.queueBuffer

```
int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
    ...
    // 获取Slot数组保存的buffer
    int i = getSlotFromBufferLocked(buffer); 
    ..
    IGraphicBufferProducer::QueueBufferOutput output;
    IGraphicBufferProducer::QueueBufferInput input(timestamp, isAutoTimestamp,
            static_cast<android_dataspace>(mDataSpace), crop, mScalingMode,
            mTransform ^ mStickyTransform, fence, mStickyTransform,
            mEnableFrameTimestamps);
    ...
    // 插入buffer
    status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output); 
    ...
    // 通知
    mQueueBufferCondition.broadcast();
    return err;
}

// mSlog集合为按照顺序保存GraphicBuffer的数组
int Surface::getSlotFromBufferLocked(android_native_buffer_t* buffer) const {
    for (int i = 0; i < NUM_BUFFER_SLOTS; i++) {
        if (mSlots[i].buffer != NULL && mSlots[i].buffer->handle == buffer->handle) {
            return i;
        }
    }
    return BAD_VALUE;
}
```
#### BufferQueueProducer.cpp.queueBuffer
根据输入参数完善一个BufferItem，然后通知frameAvailabel

```
status_t BufferQueueProducer::queueBuffer(int slot, const QueueBufferInput &input, QueueBufferOutput *output) { 

    //从input中获取一些列参数
    input.deflate(&requestedPresentTimestamp, &isAutoTimestamp, &dataSpace,
        &crop, &scalingMode, &transform, &acquireFence, &stickyTransform,
        &getFrameTimestamps);


    sp<IConsumerListener> frameAvailableListener;
    sp<IConsumerListener> frameReplacedListener;
    BufferItem item; //一个待渲染的帧

    ...
    //item的一系列赋值操作

    item.mAcquireCalled = mSlots[slot].mAcquireCalled; 
    item.mGraphicBuffer = mSlots[slot].mGraphicBuffer; //根据slot获取GraphicBuffer。
    item.mCrop = crop;
    item.mTransform = transform &
            ~static_cast<uint32_t>(NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY);
    item.mTransformToDisplayInverse =
            (transform & NATIVE_WINDOW_TRANSFORM_INVERSE_DISPLAY) != 0;
    item.mScalingMode = static_cast<uint32_t>(scalingMode);
    item.mTimestamp = requestedPresentTimestamp;
    item.mIsAutoTimestamp = isAutoTimestamp;
    
    ...

    if (frameAvailableListener != NULL) {
        frameAvailableListener->onFrameAvailable(item); //item是一个frame，准备完毕，要通知外界
    } else if (frameReplacedListener != NULL) {
        frameReplacedListener->onFrameReplaced(item);
    }

    addAndGetFrameTimestamps(&newFrameEventsEntry,etFrameTimestamps ? &output->frameTimestamps : nullptr);

    return NO_ERROR;
}
```

#### BufferLayer.cpp
```
void BufferLayer::onFrameAvailable(const BufferItem& item) {
    ...
    mFlinger->signalLayerUpdate();
}
```
#### SurfaceFlinger.cpp
执行SurfaceFlinger的invalidate方法

```
void SurfaceFlinger::signalLayerUpdate() {
    mEventQueue->invalidate();
}
```

————————————————————————————————————
**推荐阅读**：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)

