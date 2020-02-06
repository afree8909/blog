---
tags: 
- Charles
categories:
- [开发效率]
- [工具]
---


### 简介
##### 是什么？
Charles是常用的截取网络封包的工具，Mac、Windows和linux下均可用
##### 能做什么？
* 支持截取Http和Https（支持SSL代理）网络封包，
* 支持流量控制，可以模拟慢网、弱网等case
* 支持AJAX调试，可以自动格式化json或xml
* 支持重发网络请求，进行后端压测
* 支持网络请求和响应内容的mock修改

##### 相关软件
fiddler、tcpdump
##### 工作原理
![](http://upload-images.jianshu.io/upload_images/9696036-18c68b1c63bbda25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 安装配置
##### Charles安装

##### 代理设置
###### 设置charles为设置成系统代理
![](http://upload-images.jianshu.io/upload_images/9696036-37301ef7e2d3438d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 开启Charles的代理功能
![](http://upload-images.jianshu.io/upload_images/9696036-18643fd001f9b826.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###### 手机连Charles<br>
* Mac上查IP Address
* 手机Wifi连对应服务器+端口（8888）
* 第一次连接好后，Charles会弹出对应询问框（**连上才会有**），点击allow开始使用

##### Https截取配置
给Mac安装证书，并信任证书
给手机安装证书（需要安装伪CA证书）
![](http://upload-images.jianshu.io/upload_images/9696036-65d4c371e6b4171d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Charles设置SSL代理
![](http://upload-images.jianshu.io/upload_images/9696036-1ae0ba56926f6128.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 功能介绍

##### 模拟弱网
![](http://upload-images.jianshu.io/upload_images/9696036-cabadbe1187d5bab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 修改请求或响应内容
* Breakpoint
&emsp;临时性网络内容修改
![](http://upload-images.jianshu.io/upload_images/9696036-2190372ba3f82220.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* Rewrite 
&emsp;适合对某一类网络请求进行一些正则替换，以达到修改结果的目的。主要可以对某些匹配请求的header、host、url、path、query param、response status、body进行rewrite
* Repeat
&emsp;用作接口压力测试
* Map
&emsp;MapRemote重定向，适用于不同服务器切换测试
&emsp;MapLocal重定向到本地数据，适用于本地修改及时查看数据响应
* DNS Spoofing Setting
&emsp;适用于需要将**域名**打到**ip地址**的服务器上

##### 反向代理
![](http://upload-images.jianshu.io/upload_images/9696036-4dd5d0c100489910.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 常见坑
##### Charles与翻墙软件冲突
##### Charles设置MapLocal在Android中出现乱码
原因：MapLocal的Response的Headers中Content-Type值为text/plain,没有指定编码，Android的网络解析框架如果不支持的话则会出现乱码
解决：Tools->Rewrite->Rewrite Rule
![](http://upload-images.jianshu.io/upload_images/9696036-a93a904173cbe565.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### Charles连接不上
其它配置正确情况下，请先用网线试下，是否可以连接成功，如果可以的话，基本确认是Mac的wifi网络对手机不可见，如果属于公司网络，联系公司内网进行设置，保证手机和Mac的网段一致









