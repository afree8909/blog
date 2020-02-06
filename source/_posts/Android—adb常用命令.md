---
tags: 
- 命令
categories:
- [Android, 工具]
- [开发效率]
---

## 什么是adb
ADB全称Android Debug Bridge, 是android sdk里的一个工具, 用这个工具可以直接操作管理android模拟器或者真实的andriod设备(手机)。

## adb命令的主要用途
1. 运行android设备的shell(命令行)
2. 管理模拟器或android设备的映射端口
3. 安装和卸载应用程序
4. 计算机和android设备之间的上传和下载文件。

## 常用命令
### 指定设备
```
//  显示所有已连接上adb的设备
adb devices       

//  使用adb devices中展示的第一个真机作为$command目标
adb -d $command  

// 使用adb devices中展示的第一个模拟器作为$command目标 
adb -e $command  

// 使用adb devices中展示的特定serialNumber的devices作为$command目标
adb -s <serialNumber> $command   
```
### 安装&卸载
```
adb install

// 适合覆盖/升级安装（前提是apk的签名一致，如果不一致，请卸载后再安装）
adb install -r  

adb uninstall $packageName

// 保留数据信息，但是要求新安装的apk保持一致的签名
adb uninstall -k $packageName     
```
### 上传&下载
```
// 从电脑传文件到手机
adb push /users/desktop /sdcard/data/   

// 从手机传文件到电脑
adb pull /sdcard/screenshot.png /users/desktop  
```
### shell相关
#### 启动
```
 // 通过scheme打开页面
adb shell am start -d ${appScheme}

// 通过package打开
adb shell start ${packageName} 

// 通过Activity打开
adb shell am start -n ${Activity} 
```
#### 清除
```
// uninstall卸载   
adb shell pm uninstall ${packageName}    

// clear清除数据   
adb shell pm clear ${packageName}  
```

#### dumpsys
```
// 获取Activity信息
adb shell dumpsys activity
adb shell dumpsys activity top

// 获取cpu信息
adb shell dumpsys cpuinfo

// 获取设备电池信息
adb shell dumpsys 

// 获取电源管理信息
adb shell dumpsys power
```

### 截屏与录屏
```
//截屏
adb shell screencap  /sdcard/screen.png   

// 录屏  
adb shell screenrecord -size 720*1280  /sdcard/record.mp4 
```
### 屏幕参数
```
// 屏幕分辨率
adb shell wm size    

// 屏幕密度  
adb shell wm density 
```
### 系统参数
```
// 系统版本
adb shell getprop ro.build.version.release  
```

### logcat
```
//手机本地log导出
adb logcat -d >> ~/Desktop/1.txt

// 手机ANR log导出 
adb pull /data/anr/traces.txt 

// 输出 Error/Fatal 日志
adb logcat *:ef   
```




---
![](https://upload-images.jianshu.io/upload_images/9696036-70673e2d55e85b18.gif?imageMogr2/auto-orient/strip)


