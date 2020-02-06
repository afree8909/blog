
---
tags: 
- 源码
- 启动流程
- AMS
categories:
- [Android, 系统]
---



上一集：[SystemServer启动分析](https://www.jianshu.com/p/0556e0940115)


#### 系统服务启动AMS
SystemServer.startBootstrapServices

```
private void startBootstrapServices() {
    Installer installer = mSystemServiceManager.startService(Installer.class);
    // 启动 AMS 
    mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
    // AMS设置 系统服务管理器
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    // AMS设置 APP安装器
    mActivityManagerService.setInstaller(installer);
    // 启动电源管理器，AMS对其进行初始化
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
    mActivityManagerService.initPowerManagement();
    // 设置系统进程及相关
    mActivityManagerService.setSystemProcess();

}
```

#### AMS构建和启动
ActivityManagerService构造函数

* 初始化一些对象属性，包括 Context、ActivityThread、ServiceThread、MainHandler、ActivityManagerConstants 等对象
* 创建和管理四大组件相关的类对象，包括 BroadcastQueue、ActiveServices、ProviderMap、ActivityStackSupervisor、RecentTasks 和 ActivityStarter 等对象
* 创建一个 CPU 监控线程 mProcessCpuThread


```

public static final class Lifecycle extends SystemService{
    private final ActivityManagerService mService;
    // ServiceManager反射调用构造方法构造
    public Lifecycle(Context context) {
      super(context);
      mService = new ActivityManagerService(context);
    }
    
    @Override
    public void onStart() {
      mService.start(); // AMS 启动
    }
}

public ActivityManagerService(Context systemContext) {
    mContext = systemContext; // 赋值 SystemServer的context
    mFactoryTest = FactoryTest.getMode();
    // 赋值 SystemServer 的ActivityThread
    mSystemThread =  ActivityThread.currentActivityThread();
    // 创建带Handler的前台线程和MainHandler，AMS内部通信用
    mHandlerThread = new ServiceThread(TAG,
          android.os.Process.THREAD_PRIORITY_FOREGROUND, false);
    mHandlerThread.start();
    mHandler = new MainHandler(mHandlerThread.getLooper());
    // 创建UIHandler，AMS所需要的界面交互用
    mUiHandler = new UiHandler(); 
    
    // 创建前台广播接受队列 和 后台广播接受队列
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
          "foreground", BROADCAST_FG_TIMEOUT, false);
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
          "background", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;
    // 创建Service 和Provider 容器
    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);
    
    // 创建 /data/system 目录
    File dataDir = Environment.getDataDirectory();
    File systemDir = new File(dataDir, "system");
    systemDir.mkdirs();
    // 创建 电量统计服务
    mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
    mBatteryStatsService.getActiveStatistics().readLocked();
    mBatteryStatsService.scheduleWriteToDisk();
    mOnBattery = DEBUG_POWER ? true
          : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
    mBatteryStatsService.getActiveStatistics().setCallback(this);
    // 创建 进程统计服务
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
    
    mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);
    
    mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));
    
    // User 0 is the first and only user that runs at boot.
    mStartedUsers.put(UserHandle.USER_OWNER, new UserState(UserHandle.OWNER, true));
    mUserLru.add(UserHandle.USER_OWNER);
    updateStartedUserArrayLocked();
    ...
    // CPU 追踪器初始化
    mProcessCpuTracker.init();
    // 创建Activity相关对象
    mRecentTasks = new RecentTasks(this);
    mStackSupervisor = new ActivityStackSupervisor(this, mRecentTasks);
    mTaskPersister = new TaskPersister(systemDir, mStackSupervisor, mRecentTasks);
    // 创建‘CpuTracker’的现场
    mProcessCpuThread = new Thread("CpuTracker") {
      @Override
      public void run() {
          while (true) {
               synchronized(this) {
                   ... // 更新cpu状态
                   updateCpuStatsNow();
               }
          }
      }
    };
    
   ...
}

private void start() {
    // 启动 CPU 监控线程，在启动 CPU 监控线程之前，首先将进程复位
    // 注册电池状态服务和权限管理服务
    Process.removeAllProcessGroups(); //移除所有的进程组
    mProcessCpuThread.start(); //启动CpuTracker线程
    //启动电池统计服务
    mBatteryStatsService.publish(mContext);
    mAppOpsService.publish(mContext);
    //创建LocalService，并添加到LocalServices
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
}

```
SystemServer调用AMS注册各种服务

```
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this));
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this));
        }
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS);

        //创建用于性能统计的Profiler对象
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
        synchronized (this) {
            //创建ProcessRecord对象
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true; //设置为persistent进程
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);//维护进程lru
            updateOomAdjLocked(); //更新adj
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException("", e);
    }
}

```

#### 系统服务Ready-AMS
SystemServer.startOtherServices

```
private void startOtherServices() {
  ...
  //安装系统Provider
  mActivityManagerService.installSystemProviders();
  ...
  //调用AMS systemReady , 传递了 一个Runnable对象
  mActivityManagerService.systemReady(new Runnable() {
     public void run() {
         ... // AMS的systemReady方法会执行该Runnable   
     }
  }
}

```
AMS.systemReady

* task清理和恢复、是否更新广播、进程清理等systemReady前任务执行
* 系统准备好后,回调runnable，启动webView、系统UI、一系列服务ready和systemRunning
* 启动persistent进程，启动HomeActivity，发送系统广播等

```
public void systemReady(final Runnable goingCallback) {
    // before goingCallback执行
    // goingCallback执行
    // after goingCallback执行
}
```
```
public void systemReady(final Runnable goingCallback) {
    // before goingCallback执行
    
    // 同步执行 待 systemReady ok
    synchronized(this) {
        if (mSystemReady) {
            if (goingCallback != null) {
                goingCallback.run();
            }
            return;
        }
           
        mLocalDeviceIdleController
                = LocalServices.getService(DeviceIdleController.LocalService.class);
           
        updateCurrentProfileIdsLocked();
        // 清理最近task，把需要恢复的task添加上
        mRecentTasks.clear();
        mRecentTasks.addAll(mTaskPersister.restoreTasksLocked());
        mRecentTasks.cleanupLocked(UserHandle.USER_ALL);
        mTaskPersister.startPersisting();
           
        // 检查是否需要更新
        if (!mDidUpdate) {
            if (mWaitingUpdate) {
                return;
            }
            final ArrayList<ComponentName> doneReceivers = new ArrayList<ComponentName>();
            mWaitingUpdate = deliverPreBootCompleted(new Runnable() {
                public void run() {
                    synchronized (ActivityManagerService.this) {
                        mDidUpdate = true;
                    }
                    showBootMessage(mContext.getText(
                            R.string.android_upgrading_complete),
                            false);
                    writeLastDonePreBootReceivers(doneReceivers);
                    systemReady(goingCallback);
                }
            }, doneReceivers, UserHandle.USER_OWNER);
           
            if (mWaitingUpdate) {
                return;
            }
            mDidUpdate = true;
        }
           
        mAppOpsService.systemReady();
        mSystemReady = true;
    }
  
    // 将非persistent进程，添加到procsToKill
    ArrayList<ProcessRecord> procsToKill = null;
    synchronized(mPidsSelfLocked) {
       for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
           ProcessRecord proc = mPidsSelfLocked.valueAt(i);
           if (!isAllowedWhileBooting(proc.info)){
               if (procsToKill == null) {
                   procsToKill = new ArrayList<ProcessRecord>();
               }
               procsToKill.add(proc);
           }
       }
    }
   // 杀掉进程
   synchronized(this) {
       if (procsToKill != null) {
           for (int i=procsToKill.size()-1; i>=0; i--) {
               ProcessRecord proc = procsToKill.get(i);
               removeProcessLocked(proc, true, false, "system update done");
           }
       }
       // 进程 ready    
       mProcessesReady = true;
   }
   // system 现在进入ready 状态
   Slog.i(TAG, "System now ready");
   EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY,
       SystemClock.uptimeMillis());

   
   retrieveSettings();
   loadResourcesOnSystemReady();

   synchronized (this) {
       readGrantedUriPermissionsLocked(); // 权限检查
   }

        
    ...
    // goingCallback 执行
    if (goingCallback != null) goingCallback.run();
}
```
```
private void startOtherServices() {
  ...
  mActivityManagerService.systemReady(new Runnable() {
    public void run() {
      // phase550
      mSystemServiceManager.startBootPhase(
              SystemService.PHASE_ACTIVITY_MANAGER_READY);

      mActivityManagerService.startObservingNativeCrashes();
      //启动WebView
      WebViewFactory.prepareWebViewInSystemServer();
      //启动系统UI
      startSystemUi(context);

      // 执行一系列服务的systemReady方法
      networkScoreF.systemReady();
      networkManagementF.systemReady();
      networkStatsF.systemReady();
      networkPolicyF.systemReady();
      connectivityF.systemReady();
      audioServiceF.systemReady();
      Watchdog.getInstance().start(); //Watchdog开始工作

      //phase600
      mSystemServiceManager.startBootPhase(
              SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);

      //执行一系列服务的systemRunning方法
      wallpaper.systemRunning();
      inputMethodManager.systemRunning(statusBarF);
      location.systemRunning();
      countryDetector.systemRunning();
      networkTimeUpdater.systemRunning();
      commonTimeMgmtService.systemRunning();
      textServiceManagerService.systemRunning();
      assetAtlasService.systemRunning();
      inputManager.systemRunning();
      telephonyRegistry.systemRunning();
      mediaRouter.systemRunning();
      mmsService.systemRunning();
    }
  });
}
```
```
public void systemReady(final Runnable goingCallback) {
    ... // before goingCallback执行
    ... // goingCallback执行
    // after goingCallback执行
    ...
    synchronized (this) {
    if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        //通过pms获取所有的persistent进程
        List apps = AppGlobals.getPackageManager().
            getPersistentApplications(STOCK_PM_FLAGS);
        if (apps != null) {
            int N = apps.size();
            int i;
            for (i=0; i<N; i++) {
                ApplicationInfo info = (ApplicationInfo)apps.get(i);
                if (info != null && !info.packageName.equals("android")) {
                    //启动persistent进程
                    addAppLocked(info, false, null);
                }
            }
        }
    }

    mBooting = true;
    // 启动桌面Activity 
    startHomeActivityLocked(mCurrentUserId, "systemReady");

    ...
    long ident = Binder.clearCallingIdentity();
    try {
        //system发送广播USER_STARTED
        Intent intent = new Intent(Intent.ACTION_USER_STARTED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                | Intent.FLAG_RECEIVER_FOREGROUND);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, mCurrentUserId);
        broadcastIntentLocked(...);  

        //system发送广播USER_STARTING
        intent = new Intent(Intent.ACTION_USER_STARTING);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, mCurrentUserId);
        broadcastIntentLocked(...);
    } finally {
        Binder.restoreCallingIdentity(ident);
    }
    // 恢复栈顶Activity
    mStackSupervisor.resumeTopActivitiesLocked();
    sendUserSwitchBroadcastsLocked(-1, mCurrentUserId);
}
```


下一集：[Launcher启动分析](https://www.jianshu.com/p/6df6ddac15d5)




-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)


