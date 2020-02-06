---
tags: 
- 源码
categories:
- [Java, JVM]
---


### JVM大体结构图
![](http://upload-images.jianshu.io/upload_images/9696036-10ef43c4b4f8a6eb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 运行时数据区
* 线程隔离数据区
    * 程序计算器
        * 当前线程执行字节码的行号指示器
        * 如分支、循环、跳转、异常处理、线程恢复等
    * 虚拟机栈
        * 栈帧存储区域，栈帧包含局部变量、操作数栈、动态链接、方法出口等
        * 异常情况，StackOverflowError、OutOfMemoryError
    * 本地方法栈 
        * JNI服务使用的栈，作用于本地方法（C/C++） 
* 线程共享数据区
    * Java堆
        * 存储对象实例
        * 异常情况， OutOfMemoryError、内存泄漏
    * 方法区 
        * 存储加载的类信息、常量、静态变量、JIT编译后的代码等数据
        * 异常情况，OutOfMemoryError
        * 运行时常量池，字面量、符号引用
    
### 对象创建与回收过程
![](http://upload-images.jianshu.io/upload_images/9696036-a2f5045f5ab5eccb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### JVM垃圾收集算法
#### 复制
* 内存分成大小相等的两块，每次使用其中一块，回收的时候，把存活的对象复制到另一块，然后把这块内存整个清理掉
* 优点，回收效率提高、不存在碎片化
* 缺点，内存利用率低
* 解决，采用非对称法，eden:survivor = 8:1      
![](http://upload-images.jianshu.io/upload_images/9696036-c07b3961929e9872.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 标记清除
* 标记阶段，确定所有要回收的对象，做标记
* 清除阶段，将标记阶段确定的回收对象进行清除
* 优点，简单，最基础算法 
* 缺点，效率低，碎片化 
![](http://upload-images.jianshu.io/upload_images/9696036-af74e3fbdcf1e7ba.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 标记整理
* 把存活对象移动到内存的一端，然后直接回收边界以外的内存
* 场景,适合老年代存活对象较多的情况，减少内存复制量
 
![](http://upload-images.jianshu.io/upload_images/9696036-2af17d66179a1ad3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 垃圾收集器
![](http://upload-images.jianshu.io/upload_images/9696036-8457b2b3c2eae67d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Serial/SerialOld
* 过程
    1. 收集GC_ROOTS
    2. 对象可达性分析
    3. 标记垃圾对象
    4. 清理垃圾对象 
* 优点，简单高效    
* 缺点，STW，服务停顿时间长 
* 场景，几百兆以内客户端程序 
![](http://upload-images.jianshu.io/upload_images/9696036-1f48e73d09b24f8b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Parnew/Parallel Scavenge/Parallel Old
##### Serial算法多线程版
* 优点，多线程，回收速度快 
* 缺点，依然会暂停服务
* 场景，对响应时间要求不高的Server端

ParNew与Parallel Scanvenge区别
* Parnew关注回收速度，多线程减少单词GC时间
* Parallel Scanvenge关注吞吐量，减少GC时间占比
![](http://upload-images.jianshu.io/upload_images/9696036-ba913709dfc65c9c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### CMS(Concurrent Mark Sweep)算法
* 优点，并发，暂停时间短
* 缺点，耗CPU、GC时间长，GC提前，浮动垃圾，碎片化
* 场景，对响应时间敏感的Server服务，大部分线上服务应该是CMS
![](http://upload-images.jianshu.io/upload_images/9696036-6c2a95b66e1b3040.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### G1垃圾回收器
* 新一代垃圾回收算法
* 场景，大内存、高响应的服务端应用
![](http://upload-images.jianshu.io/upload_images/9696036-9fd6fdd04512a592.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### JVM工具集
#### JPS
* 查看当前用户java进程，类似于ps
* -q 只输出LVMID(与PID一致)，省略主类的名称 
* -m 输出启动时传递给主类main函数的参数
* -l 输出主类的全名，如果是jar包，输出jar路径
* -v 输出启动时的jvm参数

#### jstack
* 用于监视虚拟机各种运行状态信息，例如类装载、内存、垃圾收集、 jit编译等运行参数
*  -class 监视类装载、卸载数量、总空间以及装载所耗费时间等
* -gc 监视java堆状况，包括eden区，survivor区，老年代，永久代的容量、已用 空间和 GC时间等信息
* -gcnew 监视新生代GC状况
* -gcold 监视老年代GC状况
* -compiler 输出JIT编译器编译过的方法、耗时等信息


#### jmap
* java内存映像工具，用于生成堆转储快照，即dump文件，结合JHAT、MAT或者VisualVM等软件 来分析java内存的详细使用情况，便于排查java内存问题
* -dump 生成堆转储快照
* -finalizerinfo 显示在等待执行finalize方法的对象
* -heap 显示java堆详细信息，如使用的回收器、参数配置、分代状况等
* -histo 显示堆中对象统计信息，包括类、实例数量、合计容量等
* -permstat 以classloader为统计口径显示永久代内存状态
* -F 强制生成堆转出快照




### 参考
https://plumbr.io/handbook/what-is-garbage-collection



