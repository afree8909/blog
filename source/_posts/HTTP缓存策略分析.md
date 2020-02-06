
---
tags: 
- 缓存
- http
categories:
- [HTTP]
---

### 什么是HTTP缓存
HTTP缓存通常指浏览器缓存，基于HTTP中header字段实现
HTTP缓存分为强缓存和协商缓存，见下图

![](https://upload-images.jianshu.io/upload_images/9696036-cce700528e23bc81.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Cache-Control主要字段说明

![](https://upload-images.jianshu.io/upload_images/9696036-2a26a690808fb292.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 缓存校验字段
![](https://upload-images.jianshu.io/upload_images/9696036-4656eec189b891ab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 缓存字段对比
![](https://upload-images.jianshu.io/upload_images/9696036-0d9fb787867f2f04.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### HTTP缓存流程
![](https://upload-images.jianshu.io/upload_images/9696036-e9f7b43d56529039.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 为什么使用HTTP缓存

终端缓存策略，可以缩短端到端的请求资源的距离，减少延迟，而且缓存重用，也能减少宽带流量，降低网络负荷。
最终用户体验和性能得到优化，避免无用资源请求浪费

