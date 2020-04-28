
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428153154.png
tags: 
- 源码
- 启动流程
- Zygote
categories:
- [Android, 系统]
---


#### Zygote简介
Zygote中文翻译为“受精卵”，正如其名，它主要用于孵化子进程。
Zygote是一个C/S模型，Zygote进程作为服务端，其他进程作为客户端向它发出“孵化”请求，而Zygote接收到这个请求后就“孵化”出一个新的进程。
此篇文章着重介绍 Zygote进程的创建和启动流程

#### Zygote进程启动流程图

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428153154.png)


#### init进程main方法

##### /system/core/init/init.c

```
     int main(int argc, char **argv){      
        ... // 初始化 文件、属性服务等
        
        // 解析 init.rc 
1077    init_parse_config_file("/init.rc");
1078    // 执行 rc解析后的data （见 rc 语法）
1079    action_for_each_trigger("early-init", action_add_queue_tail);
1081    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
1082    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
1083    queue_builtin_action(keychord_init_action, "keychord_init");
1084    queue_builtin_action(console_init_action, "console_init");
1087    action_for_each_trigger("init", action_add_queue_tail);
1092    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
1093    queue_builtin_action(property_service_init_action, "property_service_init");
1094    queue_builtin_action(signal_init_action, "signal_init");
1097    if (is_charger) {
1098        action_for_each_trigger("charger", action_add_queue_tail);
1099    } else {
1100        action_for_each_trigger("late-init", action_add_queue_tail);
1101    }
1104    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");
1108    queue_builtin_action(bootchart_init_action, "bootchart_init");

1110    // 无限循环，执行action、检查是否需要重启、处理系统属性变化、回收僵尸进程等
1111    for(;;) {
            ......
1173    }
1174
1175    return 0;
1176}
1177


```

#### rc配置文件
##### /system/core/rootdir/init.rc
init.rc是一个配置文件，内部由Android初始化语言编写（Android Init Language）编写的脚本，它主要包含五种类型语句： 
Action、Commands、Services、Options和Import
##### Action
init.c的main方法中通过触发器trigger执行，执行顺序依次为 
early-init、init、late-init、boot/charger、property等
##### Service
init.c先解析rc文件将service添加到service链表中，然后有 Action配置在on XXX时机下触发启动服务
init生成的子进程，定义在rc文件，其中每一个service在启动时会通过fork方式生成子进程
##### Command
执行命令，例如：start <service_name>： 启动指定的服务
##### Options
是Service的可选项，例如 socket：创建名为/dev/soket/<name>的socket

```
...

422  on nonencrypted  // 执行启动 main服务
423    class_start main
424    class_start late_start
...  
     // service 例子
519  service servicemanager /system/bin/servicemanager
520    class core
521    user system
522    group system
523    critical
524    onrestart restart healthd
525    onrestart restart zygote
526    onrestart restart media
527    onrestart restart surfaceflinger
528    onrestart restart drm
...

```

##### /system/core/rootdir/init.zygote64.rc

```
1  service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
2    class main
3    socket zygote stream 660 root system
4    onrestart write /sys/android_power/request_state wake
5    onrestart write /sys/power/state on
6    onrestart restart media
7    onrestart restart netd

```

#### rc配置文件解析
/system/core/init/init_parser.c

```
      // 解析 rc文件
404   int init_parse_config_file(const char *fn){
409    // 解析 rc文件数据
410    parse_config(fn, data);
412    ......
413 }

347  static void parse_config(const char *fn, char *s){
       ... // 初始化变量等
365    for (;;) {
366        switch (next_token(&state)) {
367        ...
370        case T_NEWLINE:
371            state.line++;
372            if (nargs) {
373                int kw = lookup_keyword(args[0]);
374                if (kw_is(kw, SECTION)) {
375                    state.parse_line(&state, 0, 0); // 解析一份配置的每一项
376                    parse_new_section(&state, kw, nargs, args); // 解析一份配置
377                } else {
378                    state.parse_line(&state, nargs, args);
379                }
380                nargs = 0;
381            }
382            break;
384         ...
388        }
389    }
390     .....
401    }
402}

      static void parse_new_section(struct parse_state *state, int kw,
321                       int nargs, char **args){
323    
326    case K_service: // 解析服务
327        state->context = parse_service(state, nargs, args);
328        if (state->context) {
329            state->parse_line = parse_line_service;
330            return;
331        }
332        break;
333    case K_on: // 解析 Action
334        state->context = parse_action(state, nargs, args);
335        if (state->context) {
336            state->parse_line = parse_line_action;
337            return;
338        }
339        break;
340    ......
345}
346

616  static void *parse_service(struct parse_state *state, int nargs, char **args){
       // 创建service ， 并添加到 Service链表中
640    svc->name = args[1];
641    svc->classname = "default";
642    memcpy(svc->args, args + 2, sizeof(char*) * nargs);
643    svc->args[nargs] = 0;
644    svc->nargs = nargs;
645    svc->onrestart.name = "onrestart";
646    list_init(&svc->onrestart.commands);
647    list_add_tail(&service_list, &svc->slist);
648    return svc;
649 }
650

```

#### Zygote服务启动和进程创建
服务进程启动，7.0之前面向过程实现，之后面向对象实现
/system/core/init/builtins.c

```
     int do_class_start(int nargs, char **args){
221    service_for_each_class(args[1], service_start_if_not_disabled);
222    return 0;
223}

194 static void service_start_if_not_disabled(struct service *svc){
196    if (!(svc->flags & SVC_DISABLED)) {
197        service_start(svc, NULL); // 启动service
198    } else {
199        svc->flags |= SVC_DISABLED_START;
200    }
201 }

```
/system/core/init/init.c

```
    void service_start(struct service *svc, const char *dynamic_args) {
       .....
251    pid = fork();  // fork 进程
252
253    if (pid == 0) { // init 子进程
254       if (!dynamic_args) {
        // 通过execve(svc->args[0], (char**)svc->args, (char**) ENV)，进入App_main.cpp的main()函数
335            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
336                ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
337            }
338       }
356        _exit(127);
357    }
358     ...
359 }
```

#### Zygote进程main方法

/frameworks/base/cmds/app_process/app_main.cpp
主要事情：创建一个AppRuntime，调用它的start函数

```
132 int main(int argc, const char* const argv[]){
       .....
       // 传递的参数 args 为 “-Xzygote /system/bin --zygote --start-system-server”
197    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv)); // 初始一个 runtime
       
162    while (i < argc) {
163        const char* arg = argv[i++];
164        if (!parentDir) {
165            parentDir = arg;
166        } else if (strcmp(arg, "--zygote") == 0) {
167            zygote = true;   // 标识启动zygote
168            niceName = "zygote";  
169        } else if (strcmp(arg, "--start-system-server") == 0) {
170            startSystemServer = true;
171        } else if (strcmp(arg, "--application") == 0) {
172            application = true;
173        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
174            niceName = arg + 12;
175        } else {
176            className = arg;
177            break;
178        }
179    }
180
        ......
187
188    if (zygote) { // 调用AppRuntime启动ZygoteInit
189        runtime.start("com.android.internal.os.ZygoteInit",
190                startSystemServer ? "start-system-server" : "");
191    } else if (className) {
196        runtime.start("com.android.internal.os.RuntimeInit",
197                application ? "application" : "tool");
198    } else {
202        return 10;
203    }
204}
205
```

#### AndroidRuntime初始化及启动Zygote初始化
AndroidRuntime.cpp
主要事情：启动虚拟机、注册JNI方法，调用ZygoteInit的main函数

```
     void AndroidRuntime::start(const char* className, const char* options) {
        ......
836
837    /* start the virtual machine 启动虚拟机 */
838    JNIEnv* env; 
839    if (startVm(&mJavaVM, &env) != 0) {
840        return;
841    }
842    onVmCreated(env);
843
844    /*
845     * Register android functions. JNI方法注册
846     */
847    if (startReg(env) < 0) {
848        ALOGE("Unable to register all android natives\n");
849        return;
850    }
        ......
871
        // 将 "com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
876    char* slashClassName = toSlashClassName(className);
877    jclass startClass = env->FindClass(slashClassName);
878    if (startClass == NULL) {
879       ......
881    } else {
882        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
883            "([Ljava/lang/String;)V");
884        if (startMeth == NULL) {
885            ......
887        } else {
            // 调用ZygoteInit.main()方法
888            env->CallStaticVoidMethod(startClass, startMeth, strArray);
889
895    }
896   ......
903}

```
#### ZygoteInit的main方法

ZygoteInit.java
主要事情：创建Socket用来和AMS通讯、启动SystemServer、调用runSelectLoop进入循环等待唤醒并执行相应工作

```
public static void main(String argv[]) {
        try {
            RuntimeInit.enableDdms(); // 开启DDMS功能
            ......
            // 注册 ZygoteSocket 进程间通信
            registerZygoteSocket(socketName);
            // 预加载 类和资源（fork 进程共享）
            preload();
            
            // Do an initial gc to clean up after startup
            gcAndFinalize();
            // 启动 SystemServer 此处会fork出服务进程
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }

            // 开启无限循环模式 处理进程消息
            runSelectLoop(abiList);
            // 关闭 socket
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run(); // 子进程 System_Server抛异常后调用
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }

```


-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)




























