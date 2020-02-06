
---
tags: 
- 源码
- WMS
categories:
- [Android, 系统]
---


### 图文概括
#### 启动流程
![](https://upload-images.jianshu.io/upload_images/9696036-a6e6f84469e58d99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 重要成员

![](https://upload-images.jianshu.io/upload_images/9696036-726e431873d74004.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* Session
    * WMS的成员变量mSessions保存着所有的Session对象,Session继承于IWindowSession.Stub, 作为Binder服务端
    * 每一个应用进程都有一个唯一的 Session 对象与 WMS 通信
    * ViewRootImpl 和 WMS 之间的通信就是通过 Session 对象完成的
* WindowState
    * WMS中，通过mWindowMap（WindowHashMap ）保存所有的WindowState对象
    * WindowState 中保存了 WMS 对象、WMP 对象、Session 对象和 IWindow对象,一个 WindowState 对象就对应着一个应用进程中的 Window 对象; IWindow -> ViewRootImpl.W extends IWindow.Stub
* WindowToken
    * WMS成员变量mTokenMap: 保存所有的WindowToken对象; 以IBinder为key,可以是IAppWindowToken或者其他Binder的Bp端;IBinder -> ActivityRecord.Token extends IApplicationToken.Stub
    * 一个 WindowToken 就代表着一个应用组件，应用组件包括：Activity、InputMethod 等。在 WMS 中，会将属于同一 WindowToken 的做统一处理，比如在对窗口进行 ZOrder 排序时，会将属于统一 WindowToken 的排在一起
    * WindowToken 也具有令牌的作用。应用组件在创建 Window 时都需要提供一个有效的 WindowToken 以表明自己的身份，并且窗口的类型必须与所持有的 WindowToken 类型保持一致。



### 源码分析—启动流程
> 基于API23 


#### SystemServer启动WMS
* 执行WMS的main方法，进行WMS创建及初始化
* 执行WMS的displayReady方法，初始化显示信息
* 执行WMS的systemReady方法，通知 WMS 系统的初始化工作完成

参考：[Android系统_SystemServer启动流程分析
](https://www.jianshu.com/p/0556e0940115)

 ```
 // system server 执行启动其它系统服务
 private void startOtherServices() {
    ...
    // 执行 WMS的main方法
    WindowManagerService wm = WindowManagerService.main(context, inputManager,
    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
    !mFirstBoot, mOnlyCore);
    
    wm.displayReady(); 
    ...
    wm.systemReady(); 
}
 ```

#### main方法
* WMS为单例模式
* WMS的main方法，通过DisplayThread创建了一个WMS单例对象
* 运行在"android.display"线程，DisplayThread 线程是一个系统前台线程，用于执行一些延时要非常小的关于显示的操作，一般只会在 WindowManager、DisplayManager 和 InputManager 中使用

```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {

    private static WindowManagerService sInstance;
    static WindowManagerService getInstance() {
        return sInstance;
    }

    public static WindowManagerService main(final Context context, final InputManagerService im,
            final boolean haveInputMethods, final boolean showBootMsgs, final boolean onlyCore,
            WindowManagerPolicy policy) {
        // "android.display"线程
        DisplayThread.getHandler().runWithScissors(() ->
                sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs,
                        onlyCore, policy), 0);
        return sInstance;
    }

}
```

#### 构造方法
1. 赋值及初始化成员变量（context、inputManager、policy）等
2. 创建一个 WindowAnimator 对象，用于管理所有窗口的动画
3. 初始化 mPolicy ，具体的实现类是 PhoneWindowManager，WMS 的许多操作都是需要 WMP 规定的，比如：多个窗口的上下顺序，监听屏幕旋转的状态，预处理一些系统按键事件（例如HOME、BACK键等的默认行为就是在这里实现的）
4. 将 WMS 实例对象本身添加到 Watchdog 中，WMS 类实现了 Watchdog.Monitor 接口。Watchdog 用于监控系统的一些关键服务

```
private WindowManagerService(Context context, InputManagerService inputManager, boolean haveInputMethods, boolean showBootMsgs, boolean onlyCore) {
    mContext = context;
    mHaveInputMethods = haveInputMethods;
    mAllowBootMessages = showBootMsgs;
    mOnlyCore = onlyCore;
    ...
    mInputManager = inputManager; 
    mDisplayManagerInternal = LocalServices.getService(DisplayManagerInternal.class);
    mDisplaySettings = new DisplaySettings();
    mDisplaySettings.readSettingsLocked();

    LocalServices.addService(WindowManagerPolicy.class, mPolicy);
    mPointerEventDispatcher = new PointerEventDispatcher(mInputManager.monitorInput(TAG));

    mFxSession = new SurfaceSession();
    mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
    mDisplays = mDisplayManager.getDisplays();

    for (Display display : mDisplays) {
        createDisplayContentLocked(display);
    }

    mKeyguardDisableHandler = new KeyguardDisableHandler(mContext, mPolicy);
    ...

    mAppTransition = new AppTransition(context, mH);
    mAppTransition.registerListenerLocked(mActivityManagerAppTransitionNotifier);
    mActivityManager = ActivityManagerNative.getDefault();
    ...
    mAnimator = new WindowAnimator(this);
    

    LocalServices.addService(WindowManagerInternal.class, new LocalService());
    //初始化策略
    initPolicy();

    Watchdog.getInstance().addMonitor(this);

    ...
}
```

#### initPolicy
* UiThread进行policy的初始化，此过程为同步阻塞过程，运行在android.ui线程

```
private void initPolicy() {
    // 切换到 UiThread 执行，运行在"android.ui"线程，runWithScissors会判断当前执行线程，来决定是直接执行还是等待执行
    UiThread.getHandler().runWithScissors(new Runnable() {
        public void run() {
    WindowManagerPolicyThread.set(Thread.currentThread(), Looper.myLooper());
            // 初始化
            mPolicy.init(mContext, WindowManagerService.this, WindowManagerService.this);
        }
    }, 0);
}

```
#### displayReady

```
public void displayReady() {
    for (Display display : mDisplays) {
        displayReady(display.getDisplayId());
    }

    synchronized(mWindowMap) {
        final DisplayContent displayContent = getDefaultDisplayContentLocked();
        readForcedDisplayPropertiesLocked(displayContent);
        mDisplayReady = true;
    }

    mActivityManager.updateConfiguration(null);

    synchronized(mWindowMap) {
        mIsTouchDevice = mContext.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_TOUCHSCREEN);
        configureDisplayPolicyLocked(getDefaultDisplayContentLocked());
    }

    mActivityManager.updateConfiguration(null);
}
```

#### systemReady

```
public void systemReady() {
    mPolicy.systemReady();
}
```
```
public void systemReady() {
    mKeyguardDelegate = new KeyguardServiceDelegate(mContext);
    mKeyguardDelegate.onSystemReady();

    readCameraLensCoverState();
    updateUiMode();
    boolean bindKeyguardNow;
    synchronized (mLock) {
        updateOrientationListenerLp();
        mSystemReady = true;
        
        mHandler.post(new Runnable() {
            public void run() {
                updateSettings();
            }
        });

        bindKeyguardNow = mDeferBindKeyguard;
        if (bindKeyguardNow) {
            mDeferBindKeyguard = false;
        }
    }

    if (bindKeyguardNow) {
        mKeyguardDelegate.bindService(mContext);
        mKeyguardDelegate.onBootCompleted();
    }
    mSystemGestures.systemReady();
}
```

### 源码分析—重要成员
####  Session
* WMS的成员变量mSessions保存着所有的Session对象,Session继承于IWindowSession.Stub, 作为Binder服务端;
* 每一个应用进程都有一个唯一的 Session 对象与 WMS 通信
* ViewRootImpl 和 WMS 之间的通信就是通过 Session 对象完成的。


##### WMS.mSessions

```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
        // All currently active sessions with clients.
        final ArraySet<Session> mSessions = new ArraySet<>();

        @Override
        public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
            IInputContext inputContext) {
            
            Session session = new Session(this, callback, client, inputContext);
            return session;
    }
}
```
```
public class Session extends IWindowSession.Stub
        implements IBinder.DeathRecipient {
    final WindowManagerService mService; // 持有WMS
    private int mNumWindow = 0; 
    
    public Session(WindowManagerService service, IWindowSessionCallback callback,
            IInputMethodClient client, IInputContext inputContext) {
        mService = service; // 赋值

    }
    // 添加
    void windowAddedLocked(String packageName) {
        if (mSurfaceSession == null) {
            mService.mSessions.add(this);
        }
        mNumWindow++;
    }

    // 移除
    void windowRemovedLocked() {
        mNumWindow--;
        killSessionLocked();
    }

    private void killSessionLocked() {
        if (mNumWindow > 0 || !mClientDead) {
            return;
        }
        mService.mSessions.remove(this);
   }

}
```


####  WindowState
* WMS中，通过mWindowMap（WindowHashMap ）保存所有的WindowState对象
* WindowState 中保存了 WMS 对象、WMP 对象、Session 对象和 IWindow对象,一个 WindowState 对象就对应着一个应用进程中的 Window 对象; IWindow -> ViewRootImpl.W extends IWindow.Stub

##### WindowManagerService.addWindowToken
```
final HashMap<IBinder, WindowToken> mTokenMap = new HashMap<>();

public void addWindowToken(IBinder token, int type) {
   synchronized(mWindowMap) {
       WindowToken wtoken = mTokenMap.get(token);
       wtoken = new WindowToken(this, token, type, true);
       mTokenMap.put(token, wtoken);
       if (type == TYPE_WALLPAPER) {
           mWallpaperTokens.add(wtoken);
       }
   }
}
```

```
class WindowState extends WindowContainer<WindowState> implements WindowManagerPolicy.WindowState {
    final WindowManagerService mService; // WMS
    final WindowManagerPolicy mPolicy; // WMP
    final Context mContext; //ctx
    final Session mSession; // session 
    final IWindow mClient;  // 应用进程中的 Window 对象

    WindowState(WindowManagerService service, Session s, IWindow c, WindowToken token,
           WindowState parentWindow, int appOp, int seq, WindowManager.LayoutParams a,
           int viewVisibility, int ownerId, boolean ownerCanAddInternalSystemWindow) {
        mService = service;
        mSession = s;
        mClient = c;
        mAppOp = appOp;
        mToken = token;
        mAppToken = mToken.asAppWindowToken();

        ......
    }

    void attach() {
        mSession.windowAddedLocked(mAttrs.packageName);
    }
}

// IWindow mClient 就是 ViewRootImpl 中的 final W mWindow 成员
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    final W mWindow;

    static class W extends IWindow.Stub {
        ......
    }
}
```
####  WindowToken
* WMS成员变量mTokenMap: 保存所有的WindowToken对象; 以IBinder为key,可以是IAppWindowToken或者其他Binder的Bp端;IBinder -> ActivityRecord.Token extends IApplicationToken.Stub
* 一个 WindowToken 就代表着一个应用组件，应用组件包括：Activity、InputMethod 等。在 WMS 中，会将属于同一 WindowToken 的做统一处理，比如在对窗口进行 ZOrder 排序时，会将属于统一 WindowToken 的排在一起
* WindowToken 也具有令牌的作用。应用组件在创建 Window 时都需要提供一个有效的 WindowToken 以表明自己的身份，并且窗口的类型必须与所持有的 WindowToken 类型保持一致。

##### WMS.mTokenMap相关
```
final HashMap<IBinder, WindowToken> mTokenMap = new HashMap<>();

public void addWindowToken(IBinder token, int type) {
     synchronized(mWindowMap) {
       WindowToken wtoken = mTokenMap.get(token);
       wtoken = new WindowToken(this, token, type, true);
       mTokenMap.put(token, wtoken);
       if (type == TYPE_WALLPAPER) {
           mWallpaperTokens.add(wtoken);
       }
   }
}
```

-------
推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)
参考：
[WMS—启动过程](http://gityuan.com/2017/01/08/windowmanger/)


