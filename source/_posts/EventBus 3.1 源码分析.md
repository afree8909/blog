
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145500.png
tags: 
- 源码
- EventBus
categories:
- [Android, 框架]
---

>基于V3.1.1
[EventBus 官方地址](http://greenrobot.org/eventbus/documentation/)
[EventBus GitHub地址](https://github.com/greenrobot/EventBus)


### EventBus 是什么
概念：EventBus是一个Android事件发布/订阅框架
同类：Otto、RxBus
出品方：greenrobot

### EventBus 优缺点
#### 优点
* 通过消息总线形式，解耦发布者和订阅者，简化Android事件传递，从而代替Android传统的Intent、Handler、Broadcast方式或接口回调
* 支持注解配置消息
#### 不足
* 消息事件满天飞，维护难
* 无延迟发送
* 不支持跨进程/App
* Android生命周期无感知


### EventBus 工作原理
#### 消息发布和订阅模型
生产者：发布者发布消息
消费者：订阅者注册消息
调度中心： 执行消息的分发到位


![图片来源EventBus](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145447.png)
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145451.png)


#### 线程切换
主要就是两个切换，主线程切子线程，子线程切主线程
一个进程内通过Looper.getMainLooper可以顺利切换到主线程
子线程通过是通过主线程执行某个线程，自动切换，也可以借助ThreadLocal缓存对象来切换

参考：[Android线程消息机制](https://www.jianshu.com/p/75258d19a964)

### EventBus 类图
* EventBus：核心类，代表了一个事件总线。实现发布-订阅模式。
* SubscriberMethodFinder：寻找订阅者的订阅方法
* FindState：寻找订阅方法过程中的缓存Buffer，目的内存复用
* SubscriberMethod：订阅者的方法封装类
* Subscription：订阅者对象的封装
* ThreadMode：线程模型，用来标识不同线程执行消息
* PostingThreadState：消息发布状态封装
* Poster：不同线程消息发布器

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145456.png)

### EventBus 订阅-发布时序图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145500.png)

### EventBus 订阅-发布流程图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428145508.png)

### 订阅流程源码分析

EventBus.getDefault
获取EventBus单例，

```
// 提供默认单例 EventBus
public static EventBus getDefault() {
   EventBus instance = defaultInstance;
   if (instance == null) {
       synchronized (EventBus.class) {
           instance = EventBus.defaultInstance;
           if (instance == null) {
               instance = EventBus.defaultInstance = new EventBus();
           }
       }
   }
   return instance;
}

EventBus(EventBusBuilder builder) {
   logger = builder.getLogger();
   // <订阅事件类型，订阅对象集合>
   subscriptionsByEventType = new HashMap<>();
   // <订阅者，订阅事件类型集合>
   typesBySubscriber = new HashMap<>();
   // 粘性事件 map
   stickyEvents = new ConcurrentHashMap<>();
   // 主线程支持类
   mainThreadSupport = builder.getMainThreadSupport();
   // 主线程分发者
   mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
   // 后天线程分发者
   backgroundPoster = new BackgroundPoster(this);
   // 异步现场分发者
   asyncPoster = new AsyncPoster(this);
   indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
   // 订阅方法寻找器
   subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
           builder.strictMethodVerification, builder.ignoreGeneratedIndex);
   logSubscriberExceptions = builder.logSubscriberExceptions;
   logNoSubscriberMessages = builder.logNoSubscriberMessages;
   sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
   sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
   throwSubscriberException = builder.throwSubscriberException;
   // 是否执行分发父类事件类型
   eventInheritance = builder.eventInheritance;
   // 线程执行器 
   executorService = builder.executorService;
}

```

EventBus.register

1. 获取订阅者Class
2. 通过反射，获取Class的所有订阅方法集合
3. 遍历订阅方法集合，存储两个Map，Map<订阅事件类型，订阅对象集合>， Map<订阅者，订阅事件集合>
4. 如果订阅的是Stick事件类型，则直接寻找对应事件，然后分发到订阅方法

```
public void register(Object subscriber) {
    // 获取 订阅对象的 Class
   Class<?> subscriberClass = subscriber.getClass();
   //  通过注解反射拿到 订阅Class的订阅方法
   List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
   synchronized (this) {
       for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // 注册 订阅
           subscribe(subscriber, subscriberMethod);
       }
   }
}

```

SubscriberMethodFinder.findSubscriberMethods

1. 先从内存缓存 Map<订阅对象Class,订阅对象Class的订阅方法> 找
2. 然后通过反射方法获取订阅者Class的所有订阅方法
3. Map 存储
4. 返回订阅者的订阅方法集合

```
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 内存缓存 Map<订阅对象Class,订阅对象Class的订阅方法>
   List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
   if (subscriberMethods != null) {
       return subscriberMethods;
   }
    
   if (ignoreGeneratedIndex) {  // 默认false
       subscriberMethods = findUsingReflection(subscriberClass);
   } else {
        // 反射查找所有订阅方法
       subscriberMethods = findUsingInfo(subscriberClass);
   }
   if (subscriberMethods.isEmpty()) {
       throw new EventBusException("Subscriber " + subscriberClass
               + " and its super classes have no public methods with the @Subscribe annotation");
   } else {
        // map存储
       METHOD_CACHE.put(subscriberClass, subscriberMethods);
       return subscriberMethods;
}


```

SubscriberMethodFinder.findUsingInfo

1. 通过FIND_STATE_POOL构建FindState，用做找寻订阅方法时的缓存对象
2. 执行反射寻找订阅者的订阅方法，存储到FindState中
3. 循环遍历找订阅者的父类的订阅方法
4. 返回订阅者的订阅方法集合，并释放FindState

```
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    // FindState 在寻找订阅方法过程中 用于存储相关信息，为了内存复用，通过FIND_STATE_POOL维护了4个FindState来循环使用
   FindState findState = prepareFindState();
   findState.initForSubscriber(subscriberClass);
   while (findState.clazz != null) {
        // 获取SubscriberInfo 包含订阅者Class、Methods等
       findState.subscriberInfo = getSubscriberInfo(findState);
       
       if (findState.subscriberInfo != null) {
            ... // 如果有 ，直接遍历 设置methods数据
       } else {
            // 执行 反射获取订阅methods
           findUsingReflectionInSingleClass(findState);
       }
       // 父类Class，然后继续遍历找所有订阅方法
       findState.moveToSuperclass();
   }
   // 返回订阅者的订阅方法集合，并释放FindState
   return getMethodsAndRelease(findState);
}

private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
   List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
   findState.recycle();
   synchronized (FIND_STATE_POOL) {
       for (int i = 0; i < POOL_SIZE; i++) {
           if (FIND_STATE_POOL[i] == null) {
               FIND_STATE_POOL[i] = findState;
               break;
           }
       }
   }
   return subscriberMethods;
}

```

SubscriberMethodFinder.findUsingReflectionInSingleClass
遍历类中所有方法，寻找目标订阅方法，封装成SubscriberMethod存储到findState的subscriberMethods集合中

```
private void findUsingReflectionInSingleClass(FindState findState) {
   Method[] methods;
   try {
       // 获取所有声明的方法
       methods = findState.clazz.getDeclaredMethods();
   }
   ...
   // 遍历方法
   for (Method method : methods) {
       int modifiers = method.getModifiers();
       // 找 public 方法
       if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            // 找参数为1的方法
           Class<?>[] parameterTypes = method.getParameterTypes();
           if (parameterTypes.length == 1) {
                // 找 含有 Subscribe注解的方法
               Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
               if (subscribeAnnotation != null) {
                   Class<?> eventType = parameterTypes[0];
                   if (findState.checkAdd(method, eventType)) {
                        // 添加 ThreadMode
                       ThreadMode threadMode = subscribeAnnotation.threadMode();
                       // findState 添加订阅的方法
                       findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                               subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                   }
               }
      }
   }
}

```

EventBus.subscribe

```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // 事件类型 （订阅的事件）
   Class<?> eventType = subscriberMethod.eventType;
   // 构造 Subscription （封装订阅对象及对应方法）
   Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
   // 针对eventType取对应的 CopyOnWriteArrayList【主要用来实现事件优先级接受】
   CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
   if (subscriptions == null) { // 没有，则创建 订阅集合
       subscriptions = new CopyOnWriteArrayList<>();
       // 存储 <订阅事件类型，订阅对象集合>
       subscriptionsByEventType.put(eventType, subscriptions);
   } 

   int size = subscriptions.size();
   // 根据优先级添加 订阅对象
   for (int i = 0; i <= size; i++) {
       if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
           subscriptions.add(i, newSubscription);
           break;
       }
   }
   
   // 缓存 Map<订阅者，订阅事件集合>

   List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
   if (subscribedEvents == null) {
        // 存储
       subscribedEvents = new ArrayList<>();
       typesBySubscriber.put(subscriber, subscribedEvents);
   }
   subscribedEvents.add(eventType); // 添加订阅事件

    // 如果订阅方法是 支持sticky事件,则直接执行分发sticky事件
   if (subscriberMethod.sticky) {
        //表示是否分发订阅了响应事件类父类事件的方法
       if (eventInheritance) {
           Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
           for (Map.Entry<Class<?>, Object> entry : entries) {
               Class<?> candidateEventType = entry.getKey();
               if (eventType.isAssignableFrom(candidateEventType)) {
                   Object stickyEvent = entry.getValue();
                   checkPostStickyEventToSubscription(newSubscription, stickyEvent);
               }
           }
       } else {
            // 根据事件类型，获取事件
           Object stickyEvent = stickyEvents.get(eventType);
           // 分发到订阅方法
           checkPostStickyEventToSubscription(newSubscription, stickyEvent);
       }
   }
}

```

#### 发布流程源码分析

EventBus.post

参考：[深入理解ThreadLocal](https://www.jianshu.com/p/8a7fe7d592f8)

```
public void post(Object event) {
    //通过ThreadLocal<PostingThreadState> 获取当前现场的 PostingThreadState 
   PostingThreadState postingState = currentPostingThreadState.get();
   // 通过List 存储 事件
   List<Object> eventQueue = postingState.eventQueue;
   eventQueue.add(event);
    // 是否正在分发中，没有则开始分发
   if (!postingState.isPosting) {
       postingState.isMainThread = isMainThread(); // 是否主线程
       postingState.isPosting = true; // 标记为true
       if (postingState.canceled) {
           throw new EventBusException("Internal error. Abort state was not reset");
       }
       try {
            // 循环取事件，进行分发
           while (!eventQueue.isEmpty()) {
                // 分发单个事件
               postSingleEvent(eventQueue.remove(0), postingState);
           }
       } finally {
            // 结束重置状态
           postingState.isPosting = false;
           postingState.isMainThread = false;
       }
   }
}
```

EventBus.postSingleEvent
根据eventInheritance确认是否需要分发父类Class，最终执行postSingleEventForEventType 进行事件分发

```
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
   Class<?> eventClass = event.getClass();
   boolean subscriptionFound = false;
   if (eventInheritance) {
        // 查询事件对象的父类对象Class
       List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
       int countTypes = eventTypes.size();
       for (int h = 0; h < countTypes; h++) {
           Class<?> clazz = eventTypes.get(h);
           subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
       }
   } else {
       subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
   }
}

```

EventBus.postSingleEventForEventType

1. 获取订阅对象集合
2. 遍历订阅对象，执行事件分发

```
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    // 获取 订阅对象集合
   CopyOnWriteArrayList<Subscription> subscriptions;
   synchronized (this) {
       subscriptions = subscriptionsByEventType.get(eventClass);
   }
   if (subscriptions != null && !subscriptions.isEmpty()) {
       // 遍历订阅对象集合 一个个分发
       for (Subscription subscription : subscriptions) {
           postingState.event = event;
           postingState.subscription = subscription;
           boolean aborted = false;
           try {
                // 执行分发
               postToSubscription(subscription, event, postingState.isMainThread);
               aborted = postingState.canceled;
           } finally {
               postingState.event = null;
               postingState.subscription = null;
               postingState.canceled = false;
           }
           if (aborted) {
               break;
           }
       }
       return true;
   }
   return false;
}
```

EventBus.postToSubscription
根据不同的 ThreadMode 进行处理 

1. 执行线程与目标线程匹配，则直接反射执行订阅方法
2. 不匹配，则先将事件添加到PendingPostQueue，然后通过ExecutorService执行Runnable

```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    // 根据不同的 ThreadMode 进行处理 
   switch (subscription.subscriberMethod.threadMode) {
       case POSTING:
           invokeSubscriber(subscription, event);
           break;
       case MAIN:
           if (isMainThread) {
               invokeSubscriber(subscription, event);
           } else {
               mainThreadPoster.enqueue(subscription, event);
           }
           break;
       case MAIN_ORDERED:
           if (mainThreadPoster != null) {
               mainThreadPoster.enqueue(subscription, event);
           } else {
               // temporary: technically not correct as poster not decoupled from subscriber
               invokeSubscriber(subscription, event);
           }
           break;
       case BACKGROUND:
           if (isMainThread) {
               backgroundPoster.enqueue(subscription, event);
           } else {
               invokeSubscriber(subscription, event);
           }
           break;
       case ASYNC:
           asyncPoster.enqueue(subscription, event);
           break;
       default:
           throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
   }
}

// 执行订阅者的订阅方法
void invokeSubscriber(Subscription subscription, Object event) {
   try {
        // 反射调用执行方法
       subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
   } catch (InvocationTargetException e) {
       handleSubscriberException(subscription, event, e.getCause());
   } catch (IllegalAccessException e) {
       throw new IllegalStateException("Unexpected exception", e);
   }
}

```

BackgroundPoster
以BackgroundPoster为例，进行代码分析

1. 构建PendingPost，然后加入PendingPostQueue队列，接着调用线程执行器执行线程
2. 线程执行，循环获取PendingPost，调用EventBus.invokeSubscriber反射执行订阅者的订阅方法

```

final class BackgroundPoster implements Runnable, Poster {

    ...

    // 入队
    public void enqueue(Subscription subscription, Object event) {
        // 将 订阅者和事件封装为一个 PendingPost
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost); // 入队
            if (!executorRunning) {
                executorRunning = true;
                // 调用现场执行器，执行
                eventBus.getExecutorService().execute(this); 
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                while (true) {
                    PendingPost pendingPost = queue.poll(1000);
                    轮训取 post，1s没有则停止执行
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    // 最后执行eventBus的invokeSubscriber方法
                    eventBus.invokeSubscriber(pendingPost);
                }
            }
            ...
    }

}

```

---
![](https://upload-images.jianshu.io/upload_images/9696036-70673e2d55e85b18.gif?imageMogr2/auto-orient/strip)


