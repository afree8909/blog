
---
tags: 
- 源码
- 启动流程
- Launcher
categories:
- [Android, 系统]
---



前期系列：
[Zygote进程启动分析](https://www.jianshu.com/p/65cf9a2a0725)
[SystemServer启动分析](https://www.jianshu.com/p/0556e0940115)
[AMS启动分析](https://www.jianshu.com/p/725c4e7e2230)


### Launcher启动期流程图
![](https://upload-images.jianshu.io/upload_images/9696036-2ade1db25b50bc50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 代码追踪

##### AMS系统ready方法中开始启动HomeActivity
ActivityManagerService.java

* 获取和生成HomeActivity的Intent信息，并析构出ActivityInfo
* 调用ActivityStarter的startHomeActivityLocked方法

```
public void systemReady(final Runnable goingCallback) {
        ...
        // Start up initial activity.
        startHomeActivityLocked(mCurrentUserId, "systemReady");
        ...
}

boolean startHomeActivityLocked(int userId, String reason) {
    // 构建 Home Intent
    Intent intent = getHomeIntent();
    // 通过PM 构建 ActivityInfo
    ActivityInfo aInfo =
      resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
     
    intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
    // 创建ActivityInfo 对象  
    aInfo = new ActivityInfo(aInfo);
    aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
    ProcessRecord app = getProcessRecordLocked(aInfo.processName,
         aInfo.applicationInfo.uid, true);
      
    intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
    // 通过ActivityStart 启动 
    mActivityStarter.startHomeActivityLocked(intent, aInfo, myReason);    

    return true;
}

// 获取系统的启动页面Activity Intent
Intent getHomeIntent() {
   // mTOPAction = Intent.ACTION_MAIN;
   Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
   intent.setComponent(mTopComponent);
   // 添加 "android.intent.category.HOME"; 
   intent.addCategory(Intent.CATEGORY_HOME);
   return intent;
}
// 通过PM将HomeIntent解析出ActivityInfo
private ActivityInfo resolveActivityInfo(Intent intent, int flags, int userId) {
    ...
    ResolveInfo info= AppGlobals.getPackageManager().resolveIntent(intent,
    intent.resolveTypeIfNeeded(mContext.getContentResolver()),
    flags, userId);
    ai = info.activityInfo;
    return ai;
}

```

##### Activity启动前检查及任务栈管理等
ActivityStarter.java

* startHomeActivityLocked，将HomeStack移至顶部（第一次为空）
* startActivityLocked，调用startActivity，并重新记录lastStartActivityResult
* startActivity，参数校验、权限检查等，构建ActivityRecord等
* startActivityUnchecked，涉及启动模式和位运算，以及调用ActivityStack的startActivityLocked来处理回退栈


```
void startHomeActivityLocked(Intent intent, ActivityInfo aInfo, String reason) {
   // 将HomeStack 移到top
   mSupervisor.moveHomeStackTaskToTop(reason);
   // 执行 startActivityLocked
   mLastHomeActivityStartResult = startActivityLocked(null /*caller*/, intent,
           null /*ephemeralIntent*/, null /*resolvedType*/, aInfo, null /*rInfo*/,
           null /*voiceSession*/, null /*voiceInteractor*/, null /*resultTo*/,
           null /*resultWho*/, 0 /*requestCode*/, 0 /*callingPid*/, 0 /*callingUid*/,
           null /*callingPackage*/, 0 /*realCallingPid*/, 0 /*realCallingUid*/,
           0 /*startFlags*/, null /*options*/, false /*ignoreTargetSecurity*/,
           false /*componentSpecified*/, mLastHomeActivityStartRecord /*outActivity*/,
           null /*container*/, null /*inTask*/, "startHomeActivity: " + reason);
   if (mSupervisor.inResumeTopActivity) {
       // 调度
       mSupervisor.scheduleResumeTopActivities();
   }
}

final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
      
   // 参数校验、权限检查等工作，然后构建 ActivityRecord (存储Activity的重要信息)
    ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
           callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
           resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
           mSupervisor, container, options, sourceRecord);
   ...
   // 然后调用 startActivity
   return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
                options, inTask, outActivity);
}

private int startActivity(...) {
    // 执行 startActivityUnchecked
    result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
         startFlags, doResume, options, inTask, outActivity);
}

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
    IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
                                   boolean doResume, ActivityOptions options, TaskRecord inTask,
                                   ActivityRecord[] outActivity) {
    //初始化状态（该方法会校验Intent的Flag是否是特定的Flag，会涉及到各种启动模式和Android的位运算）
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor);
    //判断是否需要启动新的task
    computeLaunchingTaskFlags();
    //记录父Activity对应的TaskRecord信息
    computeSourceStack();
    mIntent.setFlags(mLaunchFlags);
    //决定是否将新Activity插入到现有的Task中
    ActivityRecord reusedActivity = getReusableIntentActivity();

    ...
  
    //任务栈历史栈配置（处理和WindowManagerService之间的交互、保证Activity对应的UI能在屏幕上显示出来）
   mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition, mOptions);
   
    if (mDoResume) { // true 
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {
            //目标Task的focusable为false或者源Task栈顶Activity总是在其他Activity之上
            //不恢复目标Task，只需确保它可见
            mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            //通过WindowManagerService执行app启动动画
            mWindowManager.executeAppTransition();
        } else { // true
            //如果目标栈之前不是可聚焦状态，那么将目标栈变为可聚焦
            if (mTargetStack.isFocusable( && !mSupervisor.isFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityUnchecked");
            }
            // 最后走到ASS执行 resumeFocusedStackTopActivityLocked
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity, mOptions);
        }
    } else {
        //如果不需要恢复，则将Activity加入到最近活动栈中
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }
    ...
    return START_SUCCESS;
}

```

ActivityStackSupervisor.java

* resumeFocusedStackTopActivityLocked,
* resumeTopActivityInnerLocked,
* resumeTopActivityInnerLocked, 暂停栈内所有Activity，继续调用
* startSpecificActivityLocked, 查找ActivityRecord对应进程，存在则realStartActivityLocked，这里是第一次启动，不存在进程,调用AMS.startProcessLocked


```
boolean resumeFocusedStackTopActivityLocked(
  ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    //  执行 resumeTopActivityUncheckedLocked
    return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
}

// ActivityStack.java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    //执行 resumeTopActivityInnerLocked
    result = resumeTopActivityInnerLocked(prev, options);
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options){
    ...
    final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

    //该方法会遍历所有任务栈，并调用ActivityStack#startPausingLocked()暂停处于栈内的所有Activity
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
    
    if (next.app != null && next.app.thread != null) {
        ...
    } else { // true
        //调用了startSpecificActivityLocked
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
}


void startSpecificActivityLocked(ActivityRecord r,
                                 boolean andResume, boolean checkConfig) {
    //这里根据processName和UID在系统中查找是否已经有相应的进程存在
    //如果之前app进程不存在，则app=null
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);
    
    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags& ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                //向PreocessRecord中增加对应的package信息
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                        mService.mProcessStats);
            }
            //若app进程存在，通知进程启动目标Activity
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
        }
    }
    
    //若进程不存在，则使用AMS开启一个新进程(进程不存在)
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,"activity", r.intent.getComponent(), false, false, true);
}
```

##### AMS开启进程启动
ActivityManagerService.java

* startProcessLocked, 进程的维护和清理等工作，然后调用重载方法
* startProcessLocked, 该方法里主要干了三件事：

    1. 设置了各种debug参数，若AndroidManifest.xml将android:debuggable设置为true，这些参数就会生效。
    2. 通过Process.start()开启一个新进程，实际上是通过Socket与Zygote通信，使用Zygote fork新进 程，同时将ActivityThread类加入到新进程,并调用ActivityThread.main()。
    3. 发送一条延时消息，若新创建的进程在消息接收前未与AMS交互，则进程启动失败

```
final ProcessRecord startProcessLocked(String processName,
       ApplicationInfo info, boolean knownToBeDead, int intentFlags,
       String hostingType, ComponentName hostingName, boolean allowWhileBooting,
       boolean isolated, boolean keepIfLarge) {
   return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
           hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
           null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
           null /* crashHandler */);
}

// 主要：创建或获取 ProcessRecord ，清理bad进程，然后启动进程
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
                                       boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName, boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge, String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
    ProcessRecord app;
    if (!isolated) { // 非孤立进程
        //根据进程名和UID查找相应的ProcessRecord，
        //当第一次启动app时这里返回值为null
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);

        if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
            //如果当前进程处于后台进程，检查当前进程是否为bad进程
            if (mAppErrors.isBadProcessLocked(info)) {
                return null;
            }
        } else {
            //当用户明确要启动一个进程时，则清空它的crash次数
            //在看见crash对话框之前它才不会成为一个bad进程
            mAppErrors.resetProcessCrashTimeLocked(info);
            if (mAppErrors.isBadProcessLocked(info)) {
                mAppErrors.clearBadProcessLocked(info);
                if (app != null) {
                    app.bad = false;
                }
            }
        }
    } else {
        //如果它是一个孤立的进程，则它无法使用现存的进程
        app = null;
    }

    //当已经存在ProcessRecord且其pid大于0(app早已经运行或者正在启动)
    //则不会清理该进程
    if (app != null && app.pid > 0) {
        if ((!knownToBeDead && !app.killed) || app.thread == null) {
            // 如果它是新的包，则将其添加到列表中
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            return app;
        }

        //当ProcessRecord被attach到之前的进程，就清理它
        killProcessGroup(app.uid, app.pid);
        handleAppDiedLocked(app, true, true);
    }

    String hostingNameStr = hostingName != null
            ? hostingName.flattenToShortString() : null;
    if (app == null) {
        //根据ApplicationInfo、processName、UID创建一个ProcessRecord对象
        app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
        if (app == null) {
            return null;
        }
        app.crashHandler = crashHandler;
    } else {
        //如果在进程中它是新的一个包，则添加它到列表里
        app.addPackage(info.packageName, info.versionCode, mProcessStats);
    }

    //如果系统仍未准备好，则推迟启动它，将app加入hold列表
    ...
    
    //调用重载方法启动进程
    startProcessLocked(app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
    return (app.pid != 0) ? app : null;
}


```
```
private final void startProcessLocked(ProcessRecord app, String hostingType, String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    //如果ProcessRecord的pid>0且不为当前进程的pid
    //就从mPidsSelfLocked移除该pid
    //当进程不存在时，pid=0
    if (app.pid > 0 && app.pid != MY_PID) {
        synchronized (mPidsSelfLocked) {
            mPidsSelfLocked.remove(app.pid);
            mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
        }
        app.setPid(0);
    }
    //从hold列表移除该ProcessRecord
    mProcessesOnHold.remove(app);
    //更新Cpu状态
    updateCpuStats();
    try {
        try {
            final int userId = UserHandle.getUserId(app.uid);
            //通过PMS检查待启动进程对应的package是否满足启动条件
            AppGlobals.getPackageManager().checkPackageStartable(app.info.packageNam     e, userId);
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }
        //
        if (!app.isolated) {
            int[] permGids = null;
            try {
                checkTime(startTime, "startProcess: getting gids from package manager");
                final IPackageManager pm = AppGlobals.getPackageManager();
                //得到对应的GID
                permGids = pm.getPackageGids(app.info.packageName,
                        MATCH_DEBUG_TRIAGED_MISSING, app.userId);
                StorageManagerInternal storageManagerInternal = LocalServices.getService(
                        StorageManagerInternal.class);
                //获得进程对外部存储的访问模式
                mountExternal = storageManagerInternal.getExternalStorageMountMode(uid,
                        app.info.packageName);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            ...
            //不同情况下设置debugFlags的值，具体的值请看Zygote类的static属性
            ...
            boolean isActivityProcess = (entryPoint == null);
            //当entryPoint为空的情况下，设置它的值
            //这里的entryPoint是第一个startProcessLocked()传进来的null值
            //这里是指定反射需要的className
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";

            ProcessStartResult startResult;
            if (hostingType.equals("webview_service")) {
              ...
            } else { // true
                //开启新进程的
                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, debugFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith, entryPointArgs);
            }
        }
        ...
        // 启动进程后更新ProcessRecord
        app.setPid(startResult.pid);
        app.usingWrapper = startResult.usingWrapper;
        app.removed = false;
        app.killed = false;
        app.killedByAm = false;

        synchronized (mPidsSelfLocked) {
            //将启动结果的pid和PreocessRecord添加到mPidsSelfLocked
            this.mPidsSelfLocked.put(startResult.pid, app);
            if (isActivityProcess) {
                //发送一个延时消息
                // PROC_START_TIMEOUT 值为 10
                //在消息未被处理前，若新创建的进程没有和AMS交互，则该进程启动失败
                Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                msg.obj = app;
                mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                        ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
            }
        } catch(RuntimeException e){
            //当创建进程出现异常的时候就会清理相关的记录forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid), false,    false, true, false, false, UserHandle.getUserId(app.userId), "start failure");
        }
    }
}

```

##### 进程fork
Process.start
参考：[Android系统—进程创建流程分析](https://www.jianshu.com/p/c7fb582987ad)

##### ActivityThread.main方法
* Looper.prepareMainLooper
* attach AMS
* Looper.loop

```
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper(); // 准备 main looper 和消息队列
    ActivityThread thread = new ActivityThread(); // 构建AT
    //将应用进程绑定到ActivityManagerService
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    Looper.loop(); // 开启循环，接收message并分发处理
}

private void attach(boolean system) { 
   sCurrentActivityThread = this;
   mSystemThread = system; // false
   if (!system) { // true
       ...
       // 获取 AMS，调用AMS的 attachApplication
       final IActivityManager mgr = ActivityManager.getService();
       try {
           mgr.attachApplication(mAppThread);
       } catch (RemoteException ex) {
           throw ex.rethrowFromSystemServer();
       }
       // Watch for getting close to heap limit.
       ...
   } else { // 系统进程处理逻辑
       ...
   }

   ...
}

```

##### AMS 绑定Application
ActivityManagerService.java

* attachApplication, 获取当前调用pid，调用重载方法
* attachApplication
    * 重置ProcessRecord信息   
    * 将进程的ApplicationThread绑定到AMS，初始化Application
    * 最后执行到 ASS.attachApplicationLocked方法进行Activity的启动

```
public final void attachApplication(IApplicationThread thread) {
   synchronized (this) {
       int callingPid = Binder.getCallingPid();
       final long origId = Binder.clearCallingIdentity();
       attachApplicationLocked(thread, callingPid);
       Binder.restoreCallingIdentity(origId);
   }
}

private final boolean attachApplicationLocked(IApplicationThread thread,
                                              int pid) {
    ProcessRecord app;
    ... // 根据pid 获取 对应 ProcessRecord
    // 新进程的名字
    final String processName = app.processName;

    try {
        //在这个地方会注册该进程的死亡回调 ， Thread指的是ApplicationThread
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        //出现异常则重新开启一个进程
        startProcessLocked(app, "link fail", processName);
        return false;
    }
    
    //重置 ProcessRecord
    app.makeActive(thread, mProcessStats); //给ProcessRecord的thread赋值
    app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
    app.curSchedGroup = app.setSchedGroup = ProcessList.SCHED_GROUP_DEFAULT;
    app.forcingToImportant = null;
    updateProcessForegroundLocked(app, false, false);
    app.hasShownUi = false;
    app.debugging = false;
    app.cached = false;
    app.killedByAm = false;
    app.killed = false;
    
    ... 
    // 移除startProcessLocked()中发出的延时消息
    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    ... // contentProvider相关处理
    
    // 这里通过AIDL调用了ApplicationThread. bindApplication方法，
    // 这里是将新进程的ApplicationThread对象绑定到AMS的真正操作 ，两个方法只是参数不同
    // app.instr 为ProcessRecord.ActiveInstrumentation对象
    if (app.instr != null) {
        thread.bindApplication(/*参数省略*/);
    }else{
        thread.bindApplication(/*参数省略*/);
    }
    
    //更新进程情况
    updateLruProcessLocked(app, false, null);
    //将ProcessRecord从正在启动列表和hold列表中移除
    mPersistentStartingProcesses.remove(app);
    mProcessesOnHold.remove(app);

    //检查最顶层可见的Activity是否等待运行在该进程中
    if (normalMode) {
        try {
            //调用ActivityStackSupervisor# attachApplicationLocked
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }
    //查找所有可运行在该进程中的服务
    //检查这个进程中是否有下一个广播接收者
    //检查这个进程中是否有下一个备份代理
    //上述操作如果出现异常就杀死进程
    ...
}

```

##### ActivityThread.ApplicationThread处理Application绑定
ActivityThread.ApplicationThread

* bindApplication，构造AppBindData，发送bind消息
* handleBindApplication 
    * 进程、系统配置等初始化设置
    * 构建Instrumentation、Application等app对象
    * 调用Application.onCreate 生命周期方法  

```
public final void bindApplication(/*省略参数*/){
	...
	// 构造 AppBindData，并赋值
	AppBindData data = new AppBindData();
	sendMessage(H.BIND_APPLICATION, data);
}

private void handleBindApplication(AppBindData data) {
    //注册UI线程到VMRuntime作为一个敏感线程
    VMRuntime.registerSensitiveThread();
    //设置进程的启动时间
    Process.setStartTimes(SystemClock.elapsedRealtime(), SystemClock.uptimeMillis());

    ...
    //设置进程名
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName, UserHandle.myUserId());
    //当app版本<= 3.1 时，设置AsyncTask使用线程池实现
    if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
        AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
    }
    //重置时区（跟随系统时区）
    TimeZone.setDefault(null);
    LocaleList.setDefault(data.config.getLocales());

    //更新系统配置
    synchronized (mResourcesManager) {
        mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
        mCurDefaultDisplayDpi = data.config.densityDpi;
        applyCompatConfiguration(mCurDefaultDisplayDpi);
    }
    // 获得LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    //判断是否需要为进程设置新的分辨率密度
    if ((data.appInfo.flags& ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
            == 0) {
        mDensityCompatMode = true;
        Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
    }
    updateDefaultDensity();

    // StrictMode
    //表示只为系统应用(FLAG_SYSTEM, FLAG_UPDATED_SYSTEM_APP)开启了
    //StrictMode，其他应用还是需要自行开启
    if ((data.appInfo.flags &
            (ApplicationInfo.FLAG_SYSTEM |
                    ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {
        StrictMode.conditionallyEnableDebugLogging();
    }
    //当api>=HONEYCOMB时,Android不允许在主线程中使用网络
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB) {
        StrictMode.enableDeathOnNetwork();
    }
    // Android 7.0以后，Android引入了FileProvider
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.N) {
        StrictMode.enableDeathOnFileUriExposure();
    }

    ...
    final InstrumentationInfo ii;
    // Instrumentation信息会影响类加载器,所以应该在设置app context之前加载它
    if (data.instrumentationName != null) {
        try {
            ii = new ApplicationPackageManager(null, getPackageManager())
                    .getInstrumentationInfo(data.instrumentationName, 0);
        } catch (PackageManager.NameNotFoundException e) {
            ...
        }
        //设置InstrumentationInfo信息
        ...
    }else{
        ii = null;
    }
    //在这里创建了ContextImpl对象
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    updateLocaleListFromAppContext(appContext,
            mResourcesManager.getConfiguration().getLocales());

    //继续加载Instrumentation对象
    if (ii != null) {
        ...
        try {
            final ClassLoader cl = instrContext.getClassLoader();
            //创建Instrumentation对象
            // 之前提到，Instrumentation的作用就是监控系统和应用的交互，
            // 因此Activity的生命周期也会被Instrumentation所监控
            mInstrumentation =(Instrumentation)cl.loadClass(
                    data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            ...
        }
        final ComponentName component = new ComponentName(ii.packageName, ii.name);
        //初始化Instrumentation参数
        mInstrumentation.init(this, instrContext, appContext, component,
                data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
        ...
    } else {
        mInstrumentation = new Instrumentation();
    }

   
    try{
        // 构建 Applicaiton
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        //设置ActivityThread.mInitialApplication
        mInitialApplication = app;
        ...
        try {
            //这里会调用到Application的onCreate()方法
            mInstrumentation.callApplicationOnCreate(app);
        } catch(Exception e){
            ...
        }
    }
}
```

##### 执行ASS的attachApplication
ActivityStackSupervisor.java

* attachApplicationLocked，找到对应ActivityRecord等
* realStartActivityLocked，更新进程信息，获取发送启动Activity参数，最后调用ActivityThread中的ApplicationThread执行 scheduleLaunchActivity 方法

```
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
   final String processName = app.processName;
   boolean didSomething = false;
   for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
       ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
       for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
           final ActivityStack stack = stacks.get(stackNdx);
           if (!isFocusedStack(stack)) {
               continue;
           }
           // 找到 当前stack 下的top ActivityRecord
           ActivityRecord hr = stack.topRunningActivityLocked();
           if (hr != null) {
               if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                       && processName.equals(hr.processName)) {
                   try {
                       // 调用 realStartActivityLocked 
                       if (realStartActivityLocked(hr, app, true, true)) {
                           didSomething = true;
                       }
                   }
                   ...
               }
           }
       }
   }
   if (!didSomething) {
       ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
   }
   return didSomething;
}


final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
       boolean andResume, boolean checkConfig) throws RemoteException {
    ...
    //获得已存在的Task和Stack
    final TaskRecord task = r.getTask();
    final ActivityStack stack = task.getStack();

    //推迟resume，避免在一个循环中多次resume
    beginDeferResume();
    //开始冻结屏幕
    r.startFreezingScreenLocked(app, 0);
    //开始收集启动信息
    r.startLaunchTickingLocked();
    r.app = app;

    if (checkConfig) {
        final int displayId = r.getDisplayId();
        final Configuration config =mWindowManager.updateOrientationFromAppTokens(
                getDisplayOverrideConfiguration(displayId),r.mayFreezeScreenLocked(app) ? r.appToken : null, displayId);
        //当显示方向改变时，推迟resume，防止启动多余的Activity
        mService.updateDisplayOverrideConfigurationLocked(config, r, true /* deferResume */,displayId);
    }
    //更新进程使用情况
    mService.updateLruProcessLocked(app, true, null);
    //更新进程OomAdj
    mService.updateOomAdjLocked();
    try {
        
        ...
        //通过Binder调用ApplicationThread的scheduleLaunchActivity()
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken, System.identityHashCode(r),r.info,mergedConfiguration.getGlobalConfiguration(),mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                r.persistentState, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profilerInfo);

        //... 处理进程臃肿的情况
    } catch(RemoteException e){
        ...
        //会进行两次操作，第一次重启进程失败后再抛出异常执行第二次操作
        //第二次失败后就放弃
    }
}


```

##### ActivityThread 执行Activity启动
* ActivityThread.Application.scheduleLaunchActivity，构建ActivityClientRecord，发送H.LAUNCH_ACTIVITY消息
* ActivityThread.handleLaunchActivity，执行Activity启动后的生命周期方法
* performLaunchActivity，主要是调用Activity的onCreate(),onStart(),onRestoreInstance(),onPostCreate()生命周期
* handleResumeActivity()，回调Activity的onResume()方法，并将DecorView添加到WindowManager中，这里的WindowManager是a.getWindowManager()得到的，其实现是WindowManagerImpl，这步操作在onResume()方法之后执行。

```
public final void scheduleLaunchActivity(...) {

    updateProcessState(procState, false);
    ActivityClientRecord r = new ActivityClientRecord();
    ...
    sendMessage(H.LAUNCH_ACTIVITY, r);
}

private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // 当前进程正活跃，避免GC
    unscheduleGcIdler();
    //确保使用的是最近的配置
    handleConfigurationChanged(null, null);
    //在创建Activity之前初始化WindowManagerService
    if (!ThreadedRenderer.sRendererDisabled) {
        GraphicsEnvironment.earlyInitEGL()
    }
    WindowManagerGlobal.initialize();
    
    //执行performLaunchActivity(),并返回Activity对象
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        
        //启动成功后，恢复Activity
        handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        // ... 
    }
}
```
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        //从PackageManagerService获取应用包信息
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    ComponentName component = r.intent.getComponent();
    if (component == null) {
        //获取组件信息
        component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }
    //如果提前设置好了目标Activity，则重新设置组件信息
    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        //利用ClassLoader去加载Activity
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //利用Instrumentation创建Activity实例
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    }
    try{
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        if (activity != null) {
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,r.embeddedID, r.lastNonConfigurationInstances, config,r.referrer, r.voiceInteractor, window, r.configCallback);
        }
        ...
        //回调Activity的onCreate()方法
        //这里回调的重载函数由ActivityInfo的persistableMode参数决定
        if (r.isPersistable()) {
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        r.activity = activity;
        r.stopped = true;
        if (!r.activity.mFinished) {
            //回调Activity的onStart()方法，同时会改变FragmentManager的状态信息
            // mInstrumentation.callActivityOnStart(this);
            activity.performStart();
            r.stopped = false;
        }

        //回调Activity的onRestoreInstanceState()方法
        //这里的回调方法同样由ActivityInfo的persistableMode参数决定
        if (!r.activity.mFinished) {
            if (r.isPersistable()) {
                if (r.state != null || r.persistentState != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                            r.persistentState);
                }
            } else if (r.state != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }
        //回调Activity的OnPostCreate()方法
        if (!r.activity.mFinished) {
            activity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnPostCreate(activity, r.state,
                        r.persistentState);
            } else {
                mInstrumentation.callActivityOnPostCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                                " did not call through to super.onPostCreate()");
            }
        }
        r.paused = true;
        mActivities.put(r.token, r);
    } catch(SuperNotCalledException e){
        ...
    }
    return activity;
}

```
```
final void handleResumeActivity(IBinder token,
                                boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    //回调Activity的onResume()方法
    r = performResumeActivity(token, clearHide, reason);
    if (r != null) {
        final Activity a = r.activity;
        final int forwardBit = isForward ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
        //显示window
        if (r.window == null && !a.mFinished && willBeVisible) {
	          ...
            ViewManager wm = a.getWindowManager();
	          ...
            //将decorView添加到WindowManager中
            wm.addView(decor, l);
        }
        ...
        //更新布局
        wm.updateViewLayout(decor, l);
        ...
        if (reallyResume) {
            try {
                //通知AMS已经Resume了
                ActivityManager.getService().activityResumed(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        } else {
            try {
                //如果在onResume之前抛出异常了,则通知AMS结束该Activity
                ActivityManager.getService()
                        .finishActivity(token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            }
        }
    }
}
```

> 您的👍，激励我的前行 👻👻👻

-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)


