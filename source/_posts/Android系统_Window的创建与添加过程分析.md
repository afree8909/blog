

---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428151445.png
tags: 
- 源码
- Window
categories:
- [Android, 系统]
---


### 流程图
（大体流程，从Activity接受启动开始）

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428151445.png)


### 源码分析
> 基于API 23

#### Window创建
##### ActivityThread.handleLaunchActivity
##### ActivityThread.performLaunchActivity

ActivityThread处理并执行Activity启动

```
//ActivityThread
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    //获取WindowManagerService的Binder引用(proxy端)。
    WindowManagerGlobal.initialize();

    //执行Activity的onCreate,onStart,onResotreInstanceState方法
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        ...
        //执行Activity的onResume方法.
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        ...
    } 

}


private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

   //通过类加载器创建Activity
   Activity activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);

   ...      

   //通过LoadedApk的makeApplication方法来创建Application对象
   Application app = r.packageInfo.makeApplication(false, mInstrumentation);


   if (activity != null) {
       ...
       // 执行 attach 方法
       activity.attach(appContext, this, getInstrumentation(), r.token,
               r.ident, app, r.intent, r.activityInfo, title, r.parent,
               r.embeddedID, r.lastNonConfigurationInstances, config,
               r.referrer, r.voiceInteractor, window);
       ...

       //onCreate
       mInstrumentation.callActivityOnCreate(activity, r.state);

       //onStart
       activity.performStart();

   }
   return activity;
}

```

##### Activity.attach
* 构造PhoneWindow （Window唯一具体实现）
* 设置自身为Window的Callback，从而Activity能作为callback接受window的key和touch事件
* 初始化且设置WindowManager，每个Activity对应一个WindowManager，通过WM与WMS进行通信

```
final void attach(...) {
    //绑定上下文
    attachBaseContext(context);
    
    //创建Window,PhoneWindow是Window的唯一具体实现类
    mWindow = new PhoneWindow(this, window);//此处的window==null，但不影响
    mWindow.setWindowControllerCallback(this);
    // 设置 Window.Callback
    mWindow.setCallback(this);
    ...
    //设置WindowManager
    mWindow.setWindowManager(
          (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
          mToken, mComponent.flattenToString(),
          (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
      mWindow.setContainer(mParent.getWindow());
    }
    //创建完后通过getWindowManager就可以得到WindowManager实例
    mWindowManager = mWindow.getWindowManager();//其实它是WindowManagerImpl

}

```

#### Window添加View过程
前面在handleLauncherActivity完成了PhoneWindow的创建过程，下面我们继续查看Activity执行生命周期handleResumeActivity方法，查看Activity对应View被加载到PhoneWindow容器过程

##### ActivityThread.handleResumeActivity

* 获取ActivityClientRecord，将对应DecorView设置为不可见，因为当前View还未绘制
* 通过Window对应的WindowManager执行addView（decor，l）操作
* View绘制完成后，会将decorView设置可见，展示该Activity对应内容，最后onResume完成

```
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {

   //把activity数据记录更新到ActivityClientRecord
   ActivityClientRecord r = performResumeActivity(token, clearHide);

   if (r != null) {

       if (r.window == null && !a.mFinished && willBeVisible) {
           r.window = r.activity.getWindow();
           View decor = r.window.getDecorView();
           decor.setVisibility(View.INVISIBLE);//不可见
           ViewManager wm = a.getWindowManager();
           WindowManager.LayoutParams l = r.window.getAttributes();
           a.mDecor = decor;
           l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;

           ...
           if (a.mVisibleFromClient && !a.mWindowAdded) {
               a.mWindowAdded = true;
               wm.addView(decor, l);// 把decor添加到窗口上
           }

       } 
           //屏幕参数发生了改变
           performConfigurationChanged(r.activity, r.tmpConfig);

           WindowManager.LayoutParams l = r.window.getAttributes();

               if (r.activity.mVisibleFromClient) {
                   ViewManager wm = a.getWindowManager();
                   View decor = r.window.getDecorView();
                   wm.updateViewLayout(decor, l);//更新窗口状态
               }


           ...
           if (r.activity.mVisibleFromClient) {
               //已经成功添加到窗口上了（绘制和事件接收），设置为可见
               r.activity.makeVisible();
           }


       //通知ActivityManagerService，Activity完成Resumed
        ActivityManagerNative.getDefault().activityResumed(token);
   } 

}

```

##### WindowManagerImply.addView
##### WindowManagerGlobal.addView
* WindowManagerImpl的全局变量通过单例模式初始化了WindowManagerGlobal，（一个进程一个WMG对象）
* Window管理类添加View过程，创建ViewRootImpl，将View处理操作交给ViewRootImpl实现


```
// WindowManagerImply
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mDisplay, mParentWindow);
}

// WindowManagerGlobal
public void addView(View view, ViewGroup.LayoutParams params,
           Display display, Window parentWindow) {
     
    synchronized (mLock) { 
        ...
        // 创建ViewRootImpl，并将View与之绑定
        root = new ViewRootImpl(view.getContext(), display);
        
        view.setLayoutParams(wparams);
        
        mViews.add(view); // 将当前view添加到mView集合中
        mRoots.add(root); // 将当前root添加mRoots集合中
        mParams.add(wparams); // 将当前window的params添加到mParams集合中
    }
    
    // setView 完成View的绘制流程，并添加到window上 
    root.setView(view, wparams, panelParentView);
   
}
```

#### ViewRootImpl.setView
* ViewRootImpl构建过程中，会通过WindowManagerGlobal的getWindowSession（static方法）获取IWindowSession即WindowSession
* 执行requestLayout方法完成View的绘制流程
* 通过WindowSession与WindowManagerService通信，将View和InputChannel添加到WMS，从而展示View并可以接受输入事件

```

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {

      ...
      int res
      requestLayout();//执行 View的绘制流程

      if ((mWindowAttributes.inputFeatures
              & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
          // 创建InputChannel
          mInputChannel = new InputChannel();
      }

      try {

          //通过WindowSession进行IPC调用，将View添加到Window上
          //mWindow即W类，用来接收WmS信息
          //同时通过InputChannel接收触摸事件回调
          res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                  getHostVisibility(), mDisplay.getDisplayId(),
                  mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                  mAttachInfo.mOutsets, mInputChannel);
      }

      ...

      //处理触摸事件回调
      mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                  Looper.myLooper());

}

```

#### WindowSession.addToDisplay
* Session执行addToDisplay方法，通过调用成员变量WMS进行addWindow操作

```
public int addToDisplay(...) {
   return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
           outContentInsets, outStableInsets, outOutsets, outInputChannel);

}

```

#### WindowManagerService.addWindow
* 创建WindowState，保存Window状态
* 调整LayoutParams参数、设置input、设置window zOrder
* 执行WindowState的attach（创建Surface过程，详情见下一篇）

```
public int addWindow(Session session, IWindow client, int seq,
       WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
       Rect outContentInsets, Rect outStableInsets, Rect outOutsets,InputChannel outInputChannel) {
   
    ... // 一堆check逻辑

    WindowToken token = mTokenMap.get(attrs.token);
    //创建 WindowState
    WindowState win = new WindowState(this, session, client, token,
          attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
    ...

    //调整 WindowManager 的 LayoutParams 参数
    mPolicy.adjustWindowParamsLw(win.mAttrs);
    res = mPolicy.prepareAddWindowLw(win, attrs);
    addWindowToListInOrderLocked(win, true);

    // 设置 input
    mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);

    // 创建 Surface 与 SurfaceFlinger 通信，
    win.attach();
    mWindowMap.put(client.asBinder(), win);
        
    if (win.canReceiveKeys()) {
    //当该窗口能接收按键事件，则更新聚焦窗口
    focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
          false /*updateInputWindows*/);
    }
    assignLayersLocked(displayContent.getWindowList());

    ...

}

```



-------
推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)
参考
[Android Window 机制探索
](https://juejin.im/entry/5a123c31f265da430d579cda)



