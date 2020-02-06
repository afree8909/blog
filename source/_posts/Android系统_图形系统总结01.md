
---
tags: 
- 源码
- 总结
categories:
- [Android, 系统]
---


> 本文基于API23源码

![目录](https://upload-images.jianshu.io/upload_images/9696036-d0a3e5312975989d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 前序
#### Android系统启动流程介绍
[Android系统_启动流程分析](https://www.jianshu.com/p/4d02ac462733)
[Android系统_Zygote启动流程分析](https://www.jianshu.com/p/65cf9a2a0725)
[Android系统_SystemServer启动流程分析](https://www.jianshu.com/p/0556e0940115)
[Android系统_ActivityManagerService启动流程](https://www.jianshu.com/p/725c4e7e2230)
[Android系统_进程创建流程分析](https://www.jianshu.com/p/c7fb582987ad)
[Android系统_Launcher启动流程分析](https://www.jianshu.com/p/6df6ddac15d5)
#### Android图形系统重要流程介绍
[Android系统_View.MeasureSpec分析](https://www.jianshu.com/p/66eb92cca405)
[Android系统_View_LinearLayout.measure分析](https://www.jianshu.com/p/e893950d6cb3)
[Android系统_View.measure解析](https://www.jianshu.com/p/c7859e02cf25)
[Android系统_View.layout解析](https://www.jianshu.com/p/f36b54feb7c5)
[Android系统_View.draw解析](https://www.jianshu.com/p/9b759b4a1aa5)
[Android系统_WindowManagerService分析](https://www.jianshu.com/p/052c635a8853)
[Android系统_Window的创建和添加流程分析](https://www.jianshu.com/p/6571fbdd1bcb)
[Android系统_Surface相关创建过程分析](https://www.jianshu.com/p/956db9044cd8)
[Android系统_Surface绘制流程分析](https://www.jianshu.com/p/c247013616d0)
[Android系统_Choreographer工作流程分析](https://www.jianshu.com/p/538df44171b1)
[Android系统_SurfaceFlinger源码分析01](https://www.jianshu.com/p/23a722df662f)


### 一图以蔽之

![](https://upload-images.jianshu.io/upload_images/9696036-a39cc7b496fd8297.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Android系统启动流程图文总结
####  系统启动流程
![](https://upload-images.jianshu.io/upload_images/9696036-9bdd9830df6844aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  进程创建流程
![](https://upload-images.jianshu.io/upload_images/9696036-b317f94b27701458.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  WMS启动流程
![](https://upload-images.jianshu.io/upload_images/9696036-1a6efe3e69764146.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####  Launcher启动时序图
![](https://upload-images.jianshu.io/upload_images/9696036-2ade1db25b50bc50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### App、AMS、WMS三者关系类图

![](https://upload-images.jianshu.io/upload_images/9696036-8ed645ddb7c0855e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Android图形绘制流程图文总结
#### Window的创建和添加流程
![](https://upload-images.jianshu.io/upload_images/9696036-b817d3db3ce31ee5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Surface的创建时序图

![](https://upload-images.jianshu.io/upload_images/9696036-af8ac6a690297127.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Surface的绘制流程
surface的绘制时序图 （软件绘制流程、硬件绘制待后续补充）

![](https://upload-images.jianshu.io/upload_images/9696036-770103d0b6d79ada.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### View的measure、layout、draw三大流程
##### View.measure 流程
![](https://upload-images.jianshu.io/upload_images/9696036-101ad3306d8fb489.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### View.MeasureSpec 说明

![](https://upload-images.jianshu.io/upload_images/9696036-e3eac09a50090ec5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### View.layout 流程

![](https://upload-images.jianshu.io/upload_images/9696036-0ce23ae06edca160.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### View.draw 流程

![](https://upload-images.jianshu.io/upload_images/9696036-0bfd5198cdefa19c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### Android图形Vsync信号刷新原理总结
Android系统确保帧缓存刷新、图形合成、图形绘制一致的原理 ？
引用一张图
![](https://upload-images.jianshu.io/upload_images/9696036-a74b5b74f6eb0c0b.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Vsync信号流转流程图 （看完图相信就知道上面的疑问了）
![](https://upload-images.jianshu.io/upload_images/9696036-f015cda5adaad776.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




---


