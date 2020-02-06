---
tags: 
- 性能
categories:
- [Android, 应用]
---



### 稳定性
#### Crash
* 来源  
    * Java层：通过设置异常处理的Handler Thread.setDefaultUncaughtExceptionHandler实现
    * Native层：[参考](https://juejin.im/entry/5962e439f265da6c2810c8aa)


#### ANR
* 来源
    * 通过FileObserver监听/data/anr目录变化，当有traces.txt文件时代表有新ANR发生
    * 通过Handler定期发送消息，并计算5秒内消失是否被处理，如果没处理代表主线程消息队列被阻塞。


### 流畅度
#### FPS
* 来源
    * 在Anroid N以下版本中通过Choreographer的FrameCallback的doFrame中计算1秒内绘制的次数
    * 在Android N及以上版本中是通过向window中添加Window.OnFrameMetricsAvailableListener，在回调中计算获得 
* 页面FPS
* 滚动FPS
* 自动义FPS

#### 启动时长
* app冷启动时长
    * 来源，使用Linux进程启动时间作为冷启动开始时间，从"/proc/$pid/stat"文件中解析得到
* 页面启动时长
* 页面首屏时长
* 自定义测速时长


### 耗损
#### 流量
* 来源
    * 系统路径：查看 /proc/net/xt_qtaguid/stats
    * 系统函数： 调用接口TrafficStats.getUidRxBytes()和TrafficStats.getUidTxBytes()
    * 系统路径： 进入 /proc/uid_stat/[uid命名的目录]/tcp_snd 和 /proc/uid_stat/[uid命名的目录]/tcp_rcv 
    * AOP：流量的统计是在网络层在编译阶段通过gradle插件插入字节码基于AOP实现流量统计，针对不同网络库主要包含HttpClient、okhttp2/okhttp3、HttpUrlConnection、自定义网络框架。通过计算Request和Response body的大小统计上行和下行流量

#### 电量
* 来源
    * 一种是利用系统提供的Api（BatteryStatsHelper等）来计算，
    * 一种是Battery Historian工具。

#### CPU
* 来源
    * CPU总使用率，在proc/stat下有具体的CPU使用情况

#### Memory
* 来源
    * 内存的统计是通过Debug.getPss()抽样的获取PSS内存计算平均值所得 

### 业务指标
#### 端到端
* 网络成功率
* 业务成功率
#### 图片
* 加载显示成功率
* 加载平均耗时
* 大小监控

### 软硬件指标
设备型号
操作系统
屏幕分辨率
手机RAM
root状态
存储空间
网络类型


### 后续计划
* 指标定义、数据收集、问题分析、问题解决方案等方面一一分析
* 数据平台建设指导

