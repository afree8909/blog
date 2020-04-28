
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428153232.png
tags: 
- 源码
- 启动流程
categories:
- [Android, 系统]
---


### Android启动流程
1. [Zygote进程启动分析](https://www.jianshu.com/p/65cf9a2a0725)
2. [SystemServer启动分析](https://www.jianshu.com/p/0556e0940115)
3. [AMS启动分析](https://www.jianshu.com/p/725c4e7e2230)
3. [Launcher启动流程分析](https://www.jianshu.com/p/6df6ddac15d5)


![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428153232.png)


### Loader
启动电源以及系统启动

* 电源按下时引导芯片会从预定义地方（固化在ROM）开始执行，加载引导程序BootLoader到RAM，然后执行
* BootLoader，引导执行程序。主要作用就是把操作系统拉起来运行

### Kernel
内核启动，初始化各种软硬件环境，加载驱动程序，挂载根文件系统，并执行init程序，由此开启Android的世界

* swapper进程：又叫idle进程，系统初始化Kernel由无到有开创的第一个进程，用于初始化进程管理、内存管理，加载Binder Drive、Disply、Camera Driver等相关工作
* kthreadd进程：Linux系统内核进程，所有内核进程鼻祖，会创建其它内核守护进程


### Native

##### init进程主要流程

* 创建和挂载启动所需的文件目录
* 初始化属性服务
* 处理子进程的终止(**signal方式**)
    *  Zygote进程异常终止的重启动
    *  回收僵尸进程 
* fork出 logd 、 healthd 、 installd 、 adbd 等用户守护进程
* 启动属性服务
    * 启动servicemanager（binder服务大管家）、bootanim（开机动画）、mediaserver等重要服务
    * 本地服务是指运行在C++层的系统守护进程
* 解析init.rc配置文件并启动Zygote进程 
    * 解析 init.zygote.rc 
    * 启动 main 类型服务 
    * 启动 zygote 服务 
    * 创建 Zygote 进程 
    * 创建 Zygote Socket 
* 进入无限循环，执行action、检查是否需要重启、处理系统属性变化、回收僵尸进程等


###  C++ Framework层
主要运行本地服务，即MediaServer进程，由init进程fork而来，负责启动和管理整个C++ framework，包含AudioFlinger、CameraService等服务


### Java Framework层
##### Zygote
[Zygote启动流程分析](https://www.jianshu.com/p/65cf9a2a0725)

* 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
* 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
* 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
* preload()预加载通用类、drawable和color资源、openGL以及共享库以及WebView，这样当程序被fork处理后，应用的进程内已经包含了这些系统资源，大大节省了应用的启动时间。
* 调用startSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。
* 最后调用runSelectLoop()，进入监听和接收消息的循环，当接收到请求创建新进程请求时立即唤醒并执行相应工作。（采用高效的I/O多路复用机制，保证没有客户端连接请求或数据处理时休眠，否则相应客户端的请求）
 
##### System Server
[System Server启动流程分析](https://www.jianshu.com/p/4d02ac462733)

* SystemServer的启动
    * 初始化设置
    * 调用createSystemContext()来创建系统上下文
    * 创建SystemServiceManager
    * 启动各种服务（引导服务、核心服务、其它服务）
    * 进入Looper.loop循环    
* 初始化系统上下文
    * 创建ActivityThread对象
    * 创建SystemContext对象（ContextImpl） 
* 创建SystemServiceManager
* 启动各种服务
    * startBootstrapServices()
    * startCoreServices()
    * startOtherServices()

##### ActivityManagerService
是Android中核心的服务之一，主要负责系统中四大组件的启动、切换、调度及应用程序的管理和调度等工作
[[Android系统—ActivityManagerService启动流程](https://www.jianshu.com/p/725c4e7e2230)
](https://www.jianshu.com/p/725c4e7e2230)

* 创建AMS实例对象，创建Andoid Runtime，ActivityThread和Context对象；
* 启动AMS服务，创建ActivityManagerService.Lifecycle对象
* setSystemProcess：注册AMS、meminfo、cpuinfo等服务到ServiceManager
* 启动SystemUIService，再调用一系列服务的systemReady()方法
* AMS.startHomeActivityLocked，启动HomeAcitivity


### App层

Zygote进程孵化出的第一个应用进程是Launcher进程（桌面），它还会孵化出Browser进程（浏览器）、Phone进程（电话）等。我们每个创建的应用都是一个单独的进程。

* 当我们点击应用图标启动应用时或者在应用内启动一个带有process标签的Activity时，都会触发创建新进程的请求，这种请求会先通过Binder
* 发送给system_server进程，也即是发送给ActivityManagerService进行处理。
system_server进程会调用Process.start()方法，会先收集uid、gid等参数，然后通过Socket方式发送给Zygote进程，请求创建新进程。
* Zygote进程接收到创建新进程的请求后，调用ZygoteInit.main()方法进行runSelectLoop()循环体内，当有客户端连接时执行ZygoteConnection.runOnce()方法，最后fork生成新的应用进程。
* 新创建的进程会调用handleChildProc()方法，最后调用我们非常熟悉的ActivityThread.main()方法。


![](https://upload-images.jianshu.io/upload_images/9696036-3eb0e8834c129aeb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)






