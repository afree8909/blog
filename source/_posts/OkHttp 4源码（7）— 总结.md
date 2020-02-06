---
tags: 
- OkHttp
categories:
- [Android, 框架]
---


>本文基于OkHttp 4.3.1源码分析 Okio 2.4.3源码分析
>[OkHttp - 官方地址](https://square.github.io/okhttp/)
[OkHttp - GitHub代码地址](https://github.com/square/okhttp)
>[Okio - 官方地址](https://square.github.io/okio/)
[Okio - GitHub代码地址](https://github.com/square/okio)


## OkHttp 介绍
### OkHttp 是什么
超文本传输协议（HTTP）是一个用于传输超媒体或者数据的应用层协议。高效应用HTTP可以获得更快的传输速度以及更节省的流量
OkHttp来源Square公司，它是针对HTTP进行高效封装的一套网络库

### Okio 优点
#### 高效
* 支持HTTP/2
* 连接池技术，避免频繁的请求连接和断开 （非HTTP2情况下）
* 支持GZIP压缩
* 缓存支持，避免重复请求

#### 高可用
* 连接重试，支持多IP重试，支持IPV4和IPV6隧道连接
* 支持TLS特性
* 请求和响应Api简洁明了，支持同步和异步请求

## OkHttp 图文总结
[OkHttp 4源码（1）— OkHttp初始化和请求构造分析](https://www.jianshu.com/p/ff836d3cacd1)
[OkHttp 4源码（2）— 拦截器机制分析](https://www.jianshu.com/p/0c830962c6e3)
[OkHttp 4源码（3）— 缓存机制分析](https://www.jianshu.com/p/2eafcd161dd9)
[OkHttp 4源码（4）— 连接机制分析](https://www.jianshu.com/p/be6d09f2656b)
[OkHttp 4源码（5）— 请求和响应 I/O操作](https://www.jianshu.com/p/097b1904f580)
[OkHttp 4源码（6）— Okio源码解析](https://www.jianshu.com/p/7b7ba4333c5e)


![OkHttp整体流程](https://upload-images.jianshu.io/upload_images/9696036-87cf14865f96650a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![请求创建和分发流程](https://upload-images.jianshu.io/upload_images/9696036-d901334cea97e334.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![拦截器时序图](https://upload-images.jianshu.io/upload_images/9696036-037f59b6d4bbf080.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![HTTP缓存流程图](https://upload-images.jianshu.io/upload_images/9696036-e9f7b43d56529039.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![缓存处理流程图](https://upload-images.jianshu.io/upload_images/9696036-9a68e72386c130ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![连接时序图](https://upload-images.jianshu.io/upload_images/9696036-c629851ea8fb73d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![HTTP不同协议连接区分](https://upload-images.jianshu.io/upload_images/9696036-d16e8389d436637b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![IO操作流程图](https://upload-images.jianshu.io/upload_images/9696036-4f71931e4b7a5416.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Okio类图](https://upload-images.jianshu.io/upload_images/9696036-70458cdb60301a91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Okio Buffer相关数据结构](https://upload-images.jianshu.io/upload_images/9696036-a8366af16cf02cb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


