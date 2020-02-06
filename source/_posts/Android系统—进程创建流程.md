
---
tags: 
- 源码
- 启动流程
- Android进程
categories:
- [Android, 系统]
---


### 前言
当某个应用组件启动且该应用没有运行其他任何组件时，Android 系统会使用单个执行线程为应用启动新的 Linux 进程。默认情况下，同一应用的所有组件在相同的进程和线程（称为“主”线程）中运行。 如果某个应用组件启动且该应用已存在进程（因为存在该应用的其他组件），则该组件会在此进程内启动并使用相同的执行线程。 但是，您可以安排应用中的其他组件在单独的进程中运行，并为任何进程创建额外的线程。

##### 进程
每个App在启动前必须先创建一个进程，该进程是由Zygote fork出来的，进程具有独立的资源空间，用于承载App上运行的各种Activity/Service等组件。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。
进程是能在系统中独立运行并作为资源分配的基本单位，是CPU分配资源的最小单位，它包括独立的地址空间，资源以及一至多个线程。
##### 线程
线程是进程中的一个实体，是CPU调度的最小单位，没有独立空间地址，多线程共享所属进程的空间地址，线程主要负责是CPU执行代码的过程
APP应用启动时，系统会为应用创建一个名为“主线程”的执行线程。

### 进程创建流程图
1. 进程创建起点，Process.start，然后告知ZygoteProcess执行启动，它会构造socket通道最后将构造指令及参数发送给zygote进程
2. zygote进程，socket会循环监听请求，在接受请求后，会封装好构建进程参数通过Zygote.forkAndSpecialize及其native方法fork出一个子进程
3. 子进程fork后，会进行一系列fork后处理事项及Runtime的init初始化工作，最后回调到子进程的zygote.runSelectLoop方法抛出异常走到ActivityThread.main方法，进入子进程世

![](https://upload-images.jianshu.io/upload_images/9696036-b317f94b27701458.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 进程创建代码分析
##### 进程创建入口
Process.java

```
public static final ProcessStartResult start(final String processClass,
                             final String niceName,
                             int uid, int gid, int[] gids,
                             int debugFlags, int mountExternal,
                             int targetSdkVersion,
                             String seInfo,
                             String abi,
                             String instructionSet,
                             String appDataDir,
                             String invokeWith,
                             String[] zygoteArgs) {
   return zygoteProcess.start(processClass, niceName, uid, gid, gids,
               debugFlags, mountExternal, targetSdkVersion, seInfo,
               abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
}
```

##### 进程创建参数准备
主要工作是生成argsForZygote数组，该数组保存了进程的uid、gid、groups、target-sdk、nice-name等一系列的参数。

ZygoteProcess.java

```

public final Process.ProcessStartResult start(final String processClass,
                                             final String niceName,
                                             int uid, int gid, int[] gids,
                                             int debugFlags, int mountExternal,
                                             int targetSdkVersion,
                                             String seInfo,
                                             String abi,
                                             String instructionSet,
                                             String appDataDir,
                                             String invokeWith,
                                             String[] zygoteArgs) {
    return startViaZygote(processClass, niceName, uid, gid, gids,
         debugFlags, mountExternal, targetSdkVersion, seInfo,
         abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
}

private static ProcessStartResult startViaZygote(final String processClass, final String niceName, final int uid, final int gid, final int[] gids, int debugFlags, int mountExternal, int targetSdkVersion, String seInfo, String abi, String instructionSet, String appDataDir, String[] extraArgs) throws ZygoteStartFailedEx {
    synchronized(Process.class) {
        ArrayList<String> argsForZygote = new ArrayList<String>();
        ...
        // 生成argsForZygote数组，该数组保存了进程的uid、gid、groups、target-sdk、nice-name等一系列的参数
        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

        if (niceName != null) {
            argsForZygote.add("--nice-name=" + niceName);
        }
        if (appDataDir != null) {
            argsForZygote.add("--app-data-dir=" + appDataDir);
        }
        argsForZygote.add(processClass);

        if (extraArgs != null) {
            for (String arg : extraArgs) {
                argsForZygote.add(arg);
            }
        }
        
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }
}

```

##### 连接Zygote服务，写入创建指令并获取返回结果
ZygoteProcess.java
通过socket向Zygote进程发送消息，从而唤醒Zygote进程，来响应socket客户端的请求

```
private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {    
    // 向主zygote发起connect()操作
    primaryZygoteState = ZygoteState.connect(mSocket); 
    //当主zygote没能匹配成功，则采用第二个zygote，发起connect()操作
    ...
}


private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
       ZygoteState zygoteState, ArrayList<String> args)
       throws ZygoteStartFailedEx {
    try {
         
      final BufferedWriter writer = zygoteState.writer;
      final DataInputStream inputStream = zygoteState.inputStream;
       
      writer.write(Integer.toString(args.size()));
      writer.newLine();
      // 写入参数
      for (int i = 0; i < sz; i++) {
          String arg = args.get(i);
          writer.write(arg);
          writer.newLine();
      }
    
      writer.flush(); 
      
      Process.ProcessStartResult result = new Process.ProcessStartResult();
    
      //等待socket服务端（即zygote）返回新创建的进程pid;
      result.pid = inputStream.readInt();
      result.usingWrapper = inputStream.readBoolean();
    
      if (result.pid < 0) {
          throw new ZygoteStartFailedEx("fork() failed");
      }
      return result;
    } catch (IOException ex) {
      zygoteState.close();
      throw new ZygoteStartFailedEx(ex);
    }
}
```

##### Zygote接收客户端Socket请求
ZygoteInit.java

```
public static void main(String argv[]) {
    try {
        // 服务端等待请求，然后处理
        zygoteServer.runSelectLoop(abiList); 
    } catch (MethodAndArgsCaller caller) {
        // 新的进程会通过抛出异常后走到这里，通过反射调用main方法执行后续任务
        caller.run(); 
    } catch (RuntimeException ex) {
        zygoteServer.closeServerSocket();
        throw ex;
    }
}
```

ZygoteServer

* 客户端通过openZygoteSocketIfNeeded()来跟zygote进程建立连接。zygote进程收到客户端连接请求后执行accept()；然后再创建ZygoteConnection对象,并添加到fds数组列表；
* 建立连接之后，可以跟客户端通信，进入runOnce()方法来接收客户端数据，并执行进程创建工作

```
void runSelectLoop(String abiList) throws Zygote.MethodAndArgsCaller {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    /sServerSocket是socket通信中的服务端，即zygote进程 
    fds.add(mServerSocket.getFileDescriptor());
    peers.add(null);
        
while (true) {
     StructPollfd[] pollFds = new StructPollfd[fds.size()];
     for (int i = 0; i < pollFds.length; ++i) {
         pollFds[i] = new StructPollfd();
         pollFds[i].fd = fds.get(i);
         pollFds[i].events = (short) POLLIN;
     }
     try {
         //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
         Os.poll(pollFds, -1);
     } catch (ErrnoException ex) {
         throw new RuntimeException("poll failed", ex);
     }
     for (int i = pollFds.length - 1; i >= 0; --i) {
         //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；否则进入continue，跳出本次循环。
         if ((pollFds[i].revents & POLLIN) == 0) {
             continue;
         }
         if (i == 0) {
            //即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；则创建ZygoteConnection对象,并添加到fds。
             ZygoteConnection newPeer = acceptCommandPeer(abiList);
             peers.add(newPeer);
             fds.add(newPeer.getFileDesciptor());
         } else {
             //i>0，则代表通过socket接收来自对端的数据，并执行 runOnce操作
             boolean done = peers.get(i).runOnce(this);
             if (done) {
                 peers.remove(i);
                 fds.remove(i);
             }
        }
     }
   }
}

private static ZygoteConnection acceptCommandPeer(String abiList) {
    // 接收客户端发送过来的connect()操作，Zygote作为服务端执行accept()操作。 再后面客户端调用write()写数据，Zygote进程调用read()读数据。
   return new ZygoteConnection(sServerSocket.accept(), abiList);
}
```

##### java层fork参数准备
ZygoteConnection.java

```
boolean runOnce(ZygoteServer zygoteServer) throws Zygote.MethodAndArgsCaller {

    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;
    
    // 读取socket客户端发送过来的参数列表
    args = readArgumentList();
    descriptors = mSocket.getAncillaryFileDescriptors();
    
    int pid = -1;
    FileDescriptor childPipeFd = null;
    FileDescriptor serverPipeFd = null;
    try {
      // 准备一系列fork进程的参数
      parsedArgs = new Arguments(args);
      int[][] rlimits = null;
      if (parsedArgs.rlimits != null) {
          rlimits = parsedArgs.rlimits.toArray(intArray2d);
      }
      int[] fdsToIgnore = null;
      if (parsedArgs.invokeWith != null) {
          FileDescriptor[] pipeFds = Os.pipe2(O_CLOEXEC);
          childPipeFd = pipeFds[1];
          serverPipeFd = pipeFds[0];
          Os.fcntlInt(childPipeFd, F_SETFD, 0);
          fdsToIgnore = new int[] { childPipeFd.getInt$(), serverPipeFd.getInt$() };
      }
      int [] fdsToClose = { -1, -1 };
      FileDescriptor fd = mSocket.getFileDescriptor();
      if (fd != null) {
          fdsToClose[0] = fd.getInt$();
      }
      fd = zygoteServer.getServerSocketFileDescriptor();
      if (fd != null) {
          fdsToClose[1] = fd.getInt$();
      }
      fd = null;
      
      // 执行fork操作
      pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
              parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
              parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.instructionSet,
              parsedArgs.appDataDir);
              
    } catch (ErrnoException ex) {
      ...
    }

   try {
       if (pid == 0) {
           // 子进程 
           zygoteServer.closeServerSocket();
           IoUtils.closeQuietly(serverPipeFd);
           serverPipeFd = null;
           // 进程初始化操作（最后抛出异常 回到runSelectLoop捕获异常方法）
           handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

           // should never get here, the child is expected to either throw Zygote.MethodAndArgsCaller or exec().
           return true;
       } else { // 父进程 （zygote进程）
           // in parent...pid of < 0 means failure
           IoUtils.closeQuietly(childPipeFd);
           childPipeFd = null;
           return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
       }
   } finally {
       IoUtils.closeQuietly(childPipeFd);
       IoUtils.closeQuietly(serverPipeFd);
   }
}

```

##### nativeFork前准备
Zygote.java
调用虚拟机执行preFork工作：停止Daemon子线程、等待所有子线程结束、完成gc堆的初始化工作

```
public static int forkAndSpecialize(int uid, int gid, int[] gids, int debugFlags,
     int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
     int[] fdsToIgnore, String instructionSet, String appDataDir) {
   VM_HOOKS.preFork();
   // Resets nice priority for zygote process.
   resetNicePriority();
   int pid = nativeForkAndSpecialize(
             uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
             fdsToIgnore, instructionSet, appDataDir);
   VM_HOOKS.postForkCommon();
   return pid;
}

```

##### 调用native的fork方法
com_android_internal_os_Zygote.cpp

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
    DetachDescriptors(env, fdsToClose); //关闭并清除文件描述符

    if (!is_system_server) {
        //对于非system_server子进程，则创建进程组
        int rc = createProcessGroup(uid, getpid());
    }
    SetGids(env, javaGids); //设置设置group
    SetRLimits(env, javaRlimits); //设置资源limit

    int rc = setresgid(gid, gid, gid);
    rc = setresuid(uid, uid, uid);

    SetCapabilities(env, permittedCapabilities, effectiveCapabilities);
    SetSchedulerPolicy(env); //设置调度策略

     //selinux上下文
    rc = selinux_android_setcontext(uid, is_system_server, se_info_c_str, se_name_c_str);

    if (se_info_c_str == NULL && is_system_server) {
      se_name_c_str = "system_server";
    }
    if (se_info_c_str != NULL) {
      SetThreadName(se_name_c_str); //设置线程名为system_server，方便调试
    }
    //在Zygote子进程中，设置信号SIGCHLD的处理器恢复为默认行为
    UnsetSigChldHandler();
    //调用zygote.callPostForkChildHooks() 
    env->CallStaticVoidMethod(gZygoteClass, gCallPostForkChildHooks, debug_flags,
                              is_system_server ? NULL : instructionSet);
    ...
  } else if (pid > 0) { //进入父进程，即zygote进程
    ...
  }
  return pid;
}

```

##### fork
fork()采用copy on write技术，这是linux创建进程的标准方法，调用一次，返回两次，返回值有3种类型。父进程中，fork返回新创建的子进程的pid;子进程中，fork返回0；当出现错误时，fork返回负数。（当进程数超过上限或者系统内存不足时会出错）

fork()的主要工作是寻找空闲的进程号pid，然后从父进程拷贝进程信息，例如数据段和代码段，fork()后子进程要执行的代码等。 Zygote进程是所有Android进程的母体，包括system_server和各个App进程。zygote利用fork()方法生成新进程，对于新进程A复用Zygote进程本身的资源，再加上新进程A相关的资源，构成新的应用进程A。

![](https://upload-images.jianshu.io/upload_images/9696036-6fe66a172ebcb763.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




```
int fork() {
  __bionic_atfork_run_prepare(); //[见小节2.1.1]

  pthread_internal_t* self = __get_thread();

  //fork期间，获取父进程pid，并使其缓存值无效
  pid_t parent_pid = self->invalidate_cached_pid();
  //系统调用【见小节2.2】
  int result = syscall(__NR_clone, FORK_FLAGS, NULL, NULL, NULL, &(self->tid));
  if (result == 0) {
    self->set_cached_pid(gettid());
    __bionic_atfork_run_child(); //fork完成执行子进程回调方法
  } else {
    self->set_cached_pid(parent_pid);
    __bionic_atfork_run_parent(); //fork完成执行父进程回调方法
  }
  return result;
}

```

#### 执行子进程fork完成后的hooks

```
private static void callPostForkChildHooks(int debugFlags, boolean isSystemServer, String instructionSet) {
    //调用ZygoteHooks.postForkChild()
    VM_HOOKS.postForkChild(debugFlags, isSystemServer, instructionSet);
}
public void postForkChild(int debugFlags, String instructionSet) {
    // 调用native方法
    nativePostForkChild(token, debugFlags, instructionSet);
Math.setRandomSeedInternal(System.currentTimeMillis());
}
```

dalvik_system_ZygoteHooks.cc
设置新进程的主线程id，重置gc性能数据，设置信号处理函数等功能。

```
static void ZygoteHooks_nativePostForkChild(JNIEnv* env, jclass, jlong token, jint debug_flags, jstring instruction_set) {
    //此处token是由nativePreFork创建的，记录着当前线程
    Thread* thread = reinterpret_cast<Thread*>(token);
    //设置新进程的主线程id
    thread->InitAfterFork();
    ..
    if (instruction_set != nullptr) {
      ScopedUtfChars isa_string(env, instruction_set);
      InstructionSet isa = GetInstructionSetFromString(isa_string.c_str());
      Runtime::NativeBridgeAction action = Runtime::NativeBridgeAction::kUnload;
      if (isa != kNone && isa != kRuntimeISA) {
        action = Runtime::NativeBridgeAction::kInitialize;
      }
      //Runtime 执行 fork事项
      Runtime::Current()->DidForkFromZygote(env, action, isa_string.c_str());
    } else {
      Runtime::Current()->DidForkFromZygote(env, Runtime::NativeBridgeAction::kUnload, nullptr);
    }
}

```

Runtime.cc
创建Java堆处理的线程池、设置信号处理函数

```
void Runtime::DidForkFromZygote(JNIEnv* env, NativeBridgeAction action, const char* isa) {
  is_zygote_ = false;
  if (is_native_bridge_loaded_) {
    switch (action) {
      case NativeBridgeAction::kUnload:
        UnloadNativeBridge(); //卸载用于跨平台的桥连库
        is_native_bridge_loaded_ = false;
        break;
      case NativeBridgeAction::kInitialize:
        InitializeNativeBridge(env, isa);//初始化用于跨平台的桥连库
        break;
    }
  }
  //创建Java堆处理的线程池
  heap_->CreateThreadPool();
  //重置gc性能数据，以保证进程在创建之前的GCs不会计算到当前app上。
  heap_->ResetGcPerformanceInfo();
  if (jit_.get() == nullptr && jit_options_->UseJIT()) {
    //当flag被设置，并且还没有创建JIT时，则创建JIT
    CreateJit();
  }
  //设置信号处理函数
  StartSignalCatcher();
  //启动JDWP线程，当命令debuger的flags指定"suspend=y"时，则暂停runtime
  Dbg::StartJdwp();
}
```

##### VM_HOOKS的fork后续操作
ZygoteHooks.java
主要功能是在fork新进程后，启动Zygote的4个Daemon线程，java堆整理，引用队列，以及析构线程。

```
public void postForkCommon() {
    Daemons.start();
}

public static void start() {
    ReferenceQueueDaemon.INSTANCE.start();
    FinalizerDaemon.INSTANCE.start();
    FinalizerWatchdogDaemon.INSTANCE.start();
    HeapTaskDaemon.INSTANCE.start();
}
```

##### 新进程创建后的初始化事项

ZygoteConnection.java

```
private void handleChildProc(Arguments parsedArgs,
       FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
       throws Zygote.MethodAndArgsCaller {
   // 关闭子进程的socket链接
   closeSocket();
   if (descriptors != null) {
   
   if (parsedArgs.invokeWith != null) {
       WrapperInit.execApplication(parsedArgs.invokeWith,
               parsedArgs.niceName, parsedArgs.targetSdkVersion,
               VMRuntime.getCurrentInstructionSet(),
               pipeFd, parsedArgs.remainingArgs);
   } else { // true
       // 初始化
       ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion,
               parsedArgs.remainingArgs, null /* classLoader */);
   }
}
```

RuntimeInit.java

* commonInit，初始化时区、代理、异常捕获处理类等
* nativeZygoteInit，启动binder相关初始化
* applicationInit，app相关初始化，最后抛出异常回到runSelectLoop，该方法的参数m是指main()方法, argv是指ActivityThread. 根据前面的中可知，下一步进入caller.run()方法，也就是执行ActivityThread的main方法。

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

##### 最后执行ActivityThread的main方法
ActivityThread.java

```
public static void main(String[] args) {
    ...
    Environment.initForCurrentUser();
    ...
    Process.setArgV0("<pre-initialized>");
    //创建主线程looper
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    //attach到系统进程
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    //主线程进入循环状态
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)


