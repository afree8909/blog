

---
tags: 
- 源码
- 启动流程
- SystemServer
categories:
- [Android, 系统]
---

### 大致流程
[Zygote启动流程分析](https://www.jianshu.com/p/65cf9a2a0725)

![](https://upload-images.jianshu.io/upload_images/9696036-de00110241787bfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 源码追踪
#### SystemServer启动
ZygoteInit.java

```
public static void main(String argv[]) {
        try {
            ......    
            // 调用启动 SystemServer方法
            if (startSystemServer) { 
                startSystemServer(abiList, socketName);
            }
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        }
        ......  
    }
    
    
    
private static boolean startSystemServer(String abiList, String socketName)
      throws MethodAndArgsCaller, RuntimeException {
      
    ... 参数准备
    int pid;
    try {
      // 解析参数，生成目标格式
      parsedArgs = new ZygoteConnection.Arguments(args);
      ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
      ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
    
      // fork SystemServer 进程 
      pid = Zygote.forkSystemServer(
              parsedArgs.uid, parsedArgs.gid,
              parsedArgs.gids,
              parsedArgs.debugFlags,
              null,
              parsedArgs.permittedCapabilities,
              parsedArgs.effectiveCapabilities);
    } catch (IllegalArgumentException ex) {
      throw new RuntimeException(ex);
    }
    
    // 子进程 SystemServer
    if (pid == 0) {    
      // 执行启动流程的剩余工作
      handleSystemServerProcess(parsedArgs); 
    }
    return true;
}

```
#### SystemServer进程fork
Zygote.java

```
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    ...
    // 调用native方法fork system_server进程
    int pid = nativeForkSystemServer(
            uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
    ...
    return pid;
}

```
com_android_internal_os_Zygote.cpp

主要：fork创建新进程SystemServer，采用copy on write方式（为了高效先全部复制，等需要的时候在修改）另外， fork方法会有两次返回，分别返回子进程和父进程的pid

```
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  // fork子进程，
  pid_t pid = ForkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities,effectiveCapabilities,
                                      MOUNT_EXTERNAL_DEFAULT, NULL, NULL, true, NULL,NULL, NULL);
  ...
  return pid;
}

static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
                                     jint debug_flags, jobjectArray javaRlimits,
                                     jlong permittedCapabilities, jlong effectiveCapabilities,
                                     jint mount_external,
                                     jstring java_se_info, jstring java_se_name,
                                     bool is_system_server, jintArray fdsToClose,
                                     jstring instructionSet, jstring dataDir) {
  ...
  
  pid_t pid = fork(); //!!! fork子进程 (COW 方式)
  
  if (pid == 0) { //进入子进程，初始化设置等
    ...
  } else if (pid > 0) { //进入父进程，即zygote进程
    ...
  }
  return pid;
}

```

#### 开始执行SystemServer进程fork后的ZygoteInit操作
ZygoteInit.java

```
private static void handleSystemServerProcess(
        ZygoteConnection.Arguments parsedArgs)
        throws ZygoteInit.MethodAndArgsCaller {
    // 关闭父进程zygote复制而来的Socket
    closeServerSocket(); 

    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName); //设置当前进程名为"system_server"
    }

    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
         // 执行dex优化操作
        performSystemServerDexOpt(systemServerClasspath);    }

    if (parsedArgs.invokeWith != null) {
       ...
    } else { // invokeWith = null
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            创建类加载器，并赋予当前线程
            cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
            Thread.currentThread().setContextClassLoader(cl);
        }

        // RuntimeInit 执行 zygoteInit初始化工作
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }

}
```

RuntimeInit.java

```
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
       throws ZygoteInit.MethodAndArgsCaller {
   ......
   commonInit();
   nativeZygoteInit();
   applicationInit(targetSdkVersion, argv, classLoader);
}

private static final void commonInit() {
    // 设置默认的未捕捉异常处理方法
    Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());

    // 设置市区，中国时区为"Asia/Shanghai"
    TimezoneGetter.setInstance(new TimezoneGetter() {
        @Override
        public String getId() {
            return SystemProperties.get("persist.sys.timezone");
        }
    });
    TimeZone.setDefault(null);

    //重置log配置
    LogManager.getLogManager().reset();
    new AndroidConfig();

    // 设置默认的HTTP User-agent格式,用于 HttpURLConnection。
    String userAgent = getDefaultUserAgent();
    System.setProperty("http.agent", userAgent);

    // 设置socket的tag，用于网络流量统计
    NetworkManagementSocketTagger.install();
}

//  /frameworks/base/core/jni/AndroidRuntime.cpp
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz){
   
    gCurRuntime->onZygoteInit();
}
//  /frameworks/base/cmds/app_process/app_main.cpp
virtual void onZygoteInit(){
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool(); //启动新binder线程
}


private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    ...
    //设置虚拟机的内存利用率参数值为0.75 , 设置目标sdk
    VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

    final Arguments args;
    try {
        args = new Arguments(argv); //解析参数
    } catch (IllegalArgumentException ex) {
        return;
    }

    // 调用startClass的static方法 main()；args.startClass为”com.android.server.SystemServer
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}  

private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl = Class.forName(className, true, classLoader);
        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (Exception ex) {
          ...
        }

        int modifiers = m.getModifiers();
        ...

        // ! 回到 ZygoteInit.main()方法中，直接进入catch语句（这样做好处是能清空栈帧，提高栈帧利用率，比较巧妙）
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

ZygoteInit.java


```
public static void main(String argv[]) {
    try {
        startSystemServer(abiList, socketName);//启动system_server
        ....
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        closeServerSocket();
        throw ex;
    }
}


public void run() {
            try {
                //根据传递过来的参数，可知此处通过反射机制调用的是SystemServer.main()方法
                mMethod.invoke(null, new Object[] { mArgs });
                ... 
            }
        }
```

#### SystemServer进程开始执行SystemServer.main方法
SystemServer.java

```
public static void main(String[] args) {
   new SystemServer().run(); //创建SystemServer对象，再调用对象的run()方法
}
    
private void run() {       
   // 设置系统时间、设置默认语言、虚拟机库文件、虚拟机内存 等
   ...
   // 当前线程作为mainLooper
   Looper.prepareMainLooper(); 

   // 初始化系统上下文
   createSystemContext();

   // 创建系统服务管理 用于创建和启动system service
   mSystemServiceManager = new SystemServiceManager(mSystemContext);
   LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

   // 启动各种系统服务
   try {
       startBootstrapServices(); // 启动引导服务
       startCoreServices();    // 启动核心服务
       startOtherServices();   // 启动其它服务
  }

   // 开启loop循环
   Looper.loop();
   throw new RuntimeException("Main thread loop unexpectedly exited");
}


private void createSystemContext() {
    // 创建主线程任务的管理和调度类 ActivityThread
   ActivityThread activityThread = ActivityThread.systemMain();
   // 会依次创建对象有ActivityThread，Instrumentation, ContextImpl，LoadedApk，Application
}

```

#### 依次启动引导服务、核心服务、其它服务

```
private void startBootstrapServices() {
    //阻塞等待 Installer 建立socket通道
    Installer installer = mSystemServiceManager.startService(Installer.class);

    //启动服务 ActivityManagerService
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);

    //启动服务 PowerManagerService
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
    mActivityManagerService.initPowerManagement();

    //启动服务 LightsService
    mSystemServiceManager.startService(LightsService.class);

    //启动服务 DisplayManagerService
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

    //Phase100: 服务启动阶段100 [100、480、500、550、600、1000]
      mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

    //启动服务 PackageManagerService
    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    mFirstBoot = mPackageManagerService.isFirstBoot();
    mPackageManager = mSystemContext.getPackageManager();

    //启动服务 UserManagerService，新建目录/data/user/
    ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

    AttributeCache.init(mSystemContext);

    //设置AMS
    mActivityManagerService.setSystemProcess();

    //启动传感器服务
    startSensorService();
}
```

```
private void startCoreServices() {
    //启动服务BatteryService，用于统计电池电量，需要LightService.
    mSystemServiceManager.startService(BatteryService.class);

    //启动服务UsageStatsService，用于统计应用使用情况
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));

    mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

    //启动服务WebViewUpdateService
    mSystemServiceManager.startService(WebViewUpdateService.class);
}
```

```
private void startOtherServices() {
    ...
    mContentResolver = context.getContentResolver(); // resolver
    mActivityManagerService.installSystemProviders(); //provider
    ActivityManagerNative.getDefault().showBootMessage(...); //显示启动界面
    ...
    
    //phase480 和phase500  [100、480、500、550、600、1000]
    mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
    ...
    
    // 准备好 window, power, package, display服务
    wm.systemReady();
    mPowerManagerService.systemReady(...);
    mPackageManagerService.systemReady();
    mDisplayManagerService.systemReady(...);
       
    // AMS ready 完成服务启动其它阶段 及Home启动等
    mActivityManagerService.systemReady(new Runnable() {
      public void run() {
         //phase550
         mSystemServiceManager.startBootPhase(
                 SystemService.PHASE_ACTIVITY_MANAGER_READY);
         ...
         //phase600
         mSystemServiceManager.startBootPhase(
                 SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
         ...
      }
    });
}

```

-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)


