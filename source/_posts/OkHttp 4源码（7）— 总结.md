---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143202.png
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
![OkHttp整体流程](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143202.png)

![请求创建和分发流程](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143210.png)

![拦截器时序图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143543.png)

![HTTP缓存流程图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143220.png)

![缓存处理流程图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143236.png)

![连接时序图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143230.png)


![HTTP不同协议连接区分](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143236.png)



![IO操作流程图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143241.png)



![Okio类图](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143246.png)


![Okio Buffer相关数据结构](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428143252.png)



