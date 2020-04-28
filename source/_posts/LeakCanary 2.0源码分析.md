
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145659.png
tags: 
- 源码
- LeakCanary
categories:
- [Android, 框架]
---

>本文基于LeakCanary 2.0源码分析
>[LeakCanary - 官方地址](https://square.github.io/leakcanary/)
[LeakCanary - GitHub代码地址](https://github.com/square/leakcanary)


### LeakCanary 是什么
概念：LeakCanary是针对Android应用的一个内存泄漏监控三方库
能力：Activity、Fragment以及自主监控的任何对象
出品：Square

### LeakCanary 作用
基于对Android Framework层的认知，LeakCanary提供更精准的泄漏原因分析能力，从而帮助开发者快速减少OOM Crash问题

### LeakCanary 工作原理
#### 预备知识
##### 什么是内存泄漏
在Java运行环境下，内存泄漏是指某个程序错误导致应用长时间一直保留某个不在需要的对象，以至于它不能被回收，而它是会占用内存的，这就意味着内存泄漏了。持续累加，最终有可能导致发生内存溢出问题。
例如一个Activity执行完onDestroy方法后，它仍然被一个static变量强引用，从而阻止了Activity被GC回收，导致Activity发生内存泄漏

##### 怎么判断一个对象是否泄漏
从GC Roots出发进行遍历，强引用可到达对象，都是存活对象，不可达对象则为即将被回收的对象。如果那些存活对象本应该是要被回收的，那么这个对象就是发生了内存泄漏（见下图，引用一张图说明）
实际过程通常作法是针对核心对象Activity、Fragment进行监控分析

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145649.png)

#### 如何开始监控
LeakCanary通过实现四大组件中的ContentProvider，所以可以在App启动的时候执行到LeakCanary的AppWatcherInstaller.onCreate方法，从而完成**0侵入**实现注册监控流程

#### 监控什么对象以及如何监控到目标对象
Activity: 通过注册 Application.ActivityLifecycleCallbacks实现onActivityDestroyed执行回调，以监控需要被回收的Activity是否被回收
Fragment： 通过注册 FragmentManager.FragmentLifecycleCallbacks实现onFragmentViewDestroyed和onFragmentDestroyed回调，以监控需要被回收的Fragment或者View是否被回收

#### 如何确认目标对象泄漏
1. 执行回收：目标对象如果在一个缓冲时间（5s）仍未被回收，我们通过手动执行GC，然后在确认其是否真的不能被回收
2. 确认是否回收：JVM中，如果创建一个含有ReferenceQueue的WeakReference的A对象，这个WeakReference对应的A对象如果被回收了，则A会被自动加入到ReferenceQueue，所以我们可以通过维护一个ReferenceQueue，通过创建目标对象的含有ReferenceQueue的WeakReference，从而监听到目标对象是否被回收

更多参考：[Reference和ReferenceQueue深入解读
](https://www.jianshu.com/p/439a8f738153)

#### 如何分析目标对象泄漏的原因
泄漏原因即寻找泄漏路径

1. 通过Debug.dumpHprofData，dump一份hprof数据
2. 读取hprof数据，整理出一份GcRoots对象索引
3. 排序GcRoots对象，并构造一份ReferencePathNode树
4. BFS遍历，找到泄漏对象的最短路径节点
5. 根据节点，生成泄漏路径


#### 如何呈现目标对象泄漏的原因
1. 生成一个HeapDumpScreen，呈现泄漏信息
2. 发出通知


### LeakCanary 2.0与1.x版本对比

| 内容 | 2.0 | 1.0 |
| --- | --- | --- |
| 语言 | kotlin | java |
| 使用 | 仅需引入库，自动注册监控 | 除引入库，还需要手动执行install |
| 内存分析 | shark，基于Okio的自实现的轻巧内存分析库 | haha三方库 |
| 其它 | fragment，支持 androidx | 无 |

### LeakCanary 源码分析
#### 主要包的结构介绍

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145659.png)


#### 主要工作的时序图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145801.png)

#### 相关源码
##### 注册流程

AppWatcherInstaller.onCreate

```
// 继承 ContentProvider
internal sealed class AppWatcherInstaller : ContentProvider() {
    // App启动 执行 onCreate 
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    // 注册启动（0 侵入）
    InternalAppWatcher.install(application)
    return true
  }

}
```
InternalAppWatcher.install

```
fun install(application: Application) {
    InternalAppWatcher.application = application
    
    val configProvider = { AppWatcher.config }
    // Activity destroy方法监控注册
    ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
    // Fragment destroy方法监控注册
    FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
    onAppWatcherInstalled(application)
}

```
ActivityDestroyWatcher.install

```
internal class ActivityDestroyWatcher private constructor(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) {
    // 构造 Application.ActivityLifecycleCallbacks
  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          objectWatcher.watch(activity, "Activity received Activity#onDestroy() callback")
        }
      }
    }

  companion object {
    fun install(
      application: Application,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val activityDestroyWatcher =
        ActivityDestroyWatcher(objectWatcher, configProvider)
      // 注册 生命周期监听回调
      application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
}

```

InternalLeakCanary.invoke

```
  // 初始化相关类
override fun invoke(application: Application) {
    this.application = application
  
    // 添加保留对象监听
AppWatcher.objectWatcher.addOnObjectRetainedListener(this)
    // Android heap dumper类
    val heapDumper = AndroidHeapDumper(application, leakDirectoryProvider)
    // GC触发器
    val gcTrigger = GcTrigger.Default
    
    val configProvider = { LeakCanary.config }
    // 相关线程 handler
    val handlerThread = HandlerThread(LEAK_CANARY_THREAD_NAME)
    handlerThread.start()
    val backgroundHandler = Handler(handlerThread.looper)
    // Heap Dump 触发器
    heapDumpTrigger = HeapDumpTrigger(
       application, backgroundHandler, AppWatcher.objectWatcher, gcTrigger, heapDumper,
       configProvider
    )
    ...
}
```




##### 监控流程
以Activity.onDestory为例
ActivityDestroyWatcher.lifecycleCallbacks

```
  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
            // 开始监控 销毁的activity
          objectWatcher.watch(activity, "Activity received Activity#onDestroy() callback")
        }
      }
    }```

```


ObjectWatcher.watch

```
  @Synchronized fun watch(
    watchedObject: Any,
    description: String
  ) {
    // 移除已经回收的监听对象
    removeWeaklyReachableObjects()
    // 随机key
    val key = UUID.randomUUID()
        .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    // 构造KeyedWeakReference 用来监听目标对象
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    // 存储 key + reference
    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
        // 执行 没有回收流程
      moveToRetained(key)
    }
  }
```

ObjectWatcher.moveToRetained

```
  @Synchronized private fun moveToRetained(key: String) {
    // 再次移除被回收的对象
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    // 如果没有被回收 则开始执行对象未被回收流程
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }
```


HeapDumpTrigger

```
  override fun onObjectRetained() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.onObjectRetained()
    }
  }
  
  fun onObjectRetained() {
    scheduleRetainedObjectCheck("found new object retained")
  }

  private fun scheduleRetainedObjectCheck(reason: String) {
    checkScheduled = true
    backgroundHandler.post {
      checkScheduled = false
      checkRetainedObjects(reason)
    }
  }

private fun checkRetainedObjects(reason: String) {
    val config = configProvider()
    // A tick will be rescheduled when this is turned back on.
    if (!config.dumpHeap) {
      SharkLog.d { "No checking for retained object: LeakCanary.Config.dumpHeap is false" }
      return
    }
    SharkLog.d { "Checking retained object because $reason" }

    var retainedReferenceCount = objectWatcher.retainedObjectCount
    // 如果还有未被回收的目标对象，则出发GC，
    if (retainedReferenceCount > 0) {
      gcTrigger.runGc()
      retainedReferenceCount = objectWatcher.retainedObjectCount
    }
    
    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

    // 触发GC后，对象仍未被回收，开始dump
    dumpHeap(retainedReferenceCount, retry = true)
  }

```

HeapDumpTrigger.dumpHeap

```
  private fun dumpHeap(
    retainedReferenceCount: Int,
    retry: Boolean
  ) {
    // 储存Android 的资源id 及其对应的name 
    saveResourceIdNamesToMemory()
    val heapDumpUptimeMillis = SystemClock.uptimeMillis()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    // dumpHeap 到file
    val heapDumpFile = heapDumper.dumpHeap()

    lastDisplayedRetainedObjectCount = 0
    objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
    // 启动service 开始dump分析
    HeapAnalyzerService.runAnalysis(application, heapDumpFile)
  }
  
```
AndroidHeapDumper.dumpHeap

```
 override fun dumpHeap(): File? {
    val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return null
    ...
    // 调用Debug的dumpHprofData 到目标文件
    return Debug.dumpHprofData(heapDumpFile.absolutePath)
             heapDumpFile
  }

```

##### 分析流程

HeapAnalyzerService.onHandleIntentInForeground

```
  override fun onHandleIntentInForeground(intent: Intent?) {
    // 获取目标 heap dump file
     val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File
    // 构造 heap分析器
    val heapAnalyzer = HeapAnalyzer(this)
    // ...
    // 执行分析流程
    val heapAnalysis =
      heapAnalyzer.analyze(
          heapDumpFile,
          config.referenceMatchers,
          config.computeRetainedHeapSize,
          config.objectInspectors,
          if (config.useExperimentalLeakFinders) config.objectInspectors else listOf(
              ObjectInspectors.KEYED_WEAK_REFERENCE
          ),
          config.metatadaExtractor,
          proguardMappingReader?.readProguardMapping()
      )

    // 回调分析完成
    config.onHeapAnalyzedListener.onHeapAnalyzed(heapAnalysis)
  }
```

HeapAnalyzer.analyze

```
fun analyze(
    heapDumpFile: File,
    referenceMatchers: List<ReferenceMatcher> = emptyList(),
    computeRetainedHeapSize: Boolean = false,
    objectInspectors: List<ObjectInspector> = emptyList(),
    leakFinders: List<ObjectInspector> = objectInspectors,
    metadataExtractor: MetadataExtractor = MetadataExtractor.NO_OP,
    proguardMapping: ProguardMapping? = null
  ): HeapAnalysis {
    val analysisStartNanoTime = System.nanoTime()

    try {
      listener.onAnalysisProgress(PARSING_HEAP_DUMP)
      // 读取 文件，然后执行分析
      Hprof.open(heapDumpFile)
          .use { hprof ->
            // Hprof -> graph 转换过程 目标获取GcRoots index
            val graph = HprofHeapGraph.indexHprof(hprof, proguardMapping)

            listener.onAnalysisProgress(EXTRACTING_METADATA)
            // 获取Android相关 metadata （如sdk版本、收集厂商等信息）
            val metadata = metadataExtractor.extractMetadata(graph)
            
            val findLeakInput = FindLeakInput(
                graph, leakFinders, referenceMatchers, computeRetainedHeapSize, objectInspectors
            )
            // 找泄漏最短路径
            val (applicationLeaks, libraryLeaks) = findLeakInput.findLeaks()
            listener.onAnalysisProgress(REPORTING_HEAP_ANALYSIS)
            // 返回分析成功结果
            return HeapAnalysisSuccess(
                heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime), metadata,
                applicationLeaks, libraryLeaks
            )
          }

    }

```

HeapAnalyzer.findLeaks

```
  private fun FindLeakInput.findLeaks(): Pair<List<ApplicationLeak>, List<LibraryLeak>> {
    // 找未被回收对象的 objectId
    val leakingInstanceObjectIds = findRetainedObjects()
    // 构造pathFinder对象
    val pathFinder = PathFinder(graph, listener, referenceMatchers)
    val pathFindingResults =
    // ⚠️ 找泄漏对象到GcRoots的最短路径
    pathFinder.findPathsFromGcRoots(leakingInstanceObjectIds, computeRetainedHeapSize)

   // 返回 泄漏路径
    return buildLeakTraces(pathFindingResults)
  }

```

PathFinder.findPatchsFromGcRoots

```
  fun findPathsFromGcRoots(
    leakingObjectIds: Set<Long>,
    computeRetainedHeapSize: Boolean
  ): PathFindingResults {
    listener.onAnalysisProgress(FINDING_PATHS_TO_RETAINED_OBJECTS)

    val sizeOfObjectInstances = determineSizeOfObjectInstances(graph)

    val state = State(leakingObjectIds, sizeOfObjectInstances, computeRetainedHeapSize)
    // 执行 state。findPathsFromGcRoots
    return state.findPathsFromGcRoots()
  }
  
  private fun State.findPathsFromGcRoots(): PathFindingResults {
    // GcRoots 生成节点队列树
    enqueueGcRoots()

    val shortestPathsToLeakingObjects = mutableListOf<ReferencePathNode>()
    visitingQueue@ while (queuesNotEmpty) {
      val node = poll() // 循环取node

        // 泄漏的节点，则添加到shortestPathsToLeakingObjects 直到，全部找完 
      if (node.objectId in leakingObjectIds) {
        shortestPathsToLeakingObjects.add(node)
        // Found all refs, stop searching (unless computing retained size)
        if (shortestPathsToLeakingObjects.size == leakingObjectIds.size) {
          if (computeRetainedHeapSize) {
            listener.onAnalysisProgress(FINDING_DOMINATORS)
          } else {
            break@visitingQueue
          }
        }
      }
    }
    return PathFindingResults(shortestPathsToLeakingObjects, dominatedObjectIds)
  }
```

##### 呈现流程

DefaultOnHeapAnalyzedListener.onHeapAnalyzed

```
  override fun onHeapAnalyzed(heapAnalysis: HeapAnalysis) {
    // 写入 db
    val (id, groupProjections) = LeaksDbHelper(application)
        .writableDatabase.use { db ->
      val id = HeapAnalysisTable.insert(db, heapAnalysis)
      id to LeakTable.retrieveHeapDumpLeaks(db, id)
    }

    // 生成 泄漏信息到屏幕展示
    val (contentTitle, screenToShow) = when (heapAnalysis) {
      is HeapAnalysisFailure -> application.getString(
          R.string.leak_canary_analysis_failed
      ) to HeapAnalysisFailureScreen(id)
      is HeapAnalysisSuccess -> {
        var leakCount = 0
        var newLeakCount = 0
        var knownLeakCount = 0
        var libraryLeakCount = 0

        for ((_, projection) in groupProjections) {
          leakCount += projection.leakCount
          when {
            projection.isLibraryLeak -> libraryLeakCount += projection.leakCount
            projection.isNew -> newLeakCount += projection.leakCount
            else -> knownLeakCount += projection.leakCount
          }
        }

        application.getString(
            R.string.leak_canary_analysis_success_notification, leakCount, newLeakCount,
            knownLeakCount, libraryLeakCount
        ) to HeapDumpScreen(id)
      }
    }

    val pendingIntent = LeakActivity.createPendingIntent(
        application, arrayListOf(HeapDumpsScreen(), screenToShow)
    )

    val contentText = application.getString(R.string.leak_canary_notification_message)
    // 构建通知
    Notifications.showNotification(
        application, contentTitle, contentText, pendingIntent,
        R.id.leak_canary_notification_analysis_result,
        LEAKCANARY_MAX
    )
  }

```



---
![](https://upload-images.jianshu.io/upload_images/9696036-70673e2d55e85b18.gif?imageMogr2/auto-orient/strip)


