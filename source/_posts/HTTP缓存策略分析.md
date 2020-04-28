
---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152801.png
tags: 
- 缓存
- http
categories:
- [HTTP]
---

### 什么是HTTP缓存
HTTP缓存通常指浏览器缓存，基于HTTP中header字段实现
HTTP缓存分为强缓存和协商缓存，见下图


![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152733.png)




#### Cache-Control主要字段说明

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152738.png)


#### 缓存校验字段
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152745.png)


#### 缓存字段对比
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152750.png)





### HTTP缓存流程
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152801.png)

### 为什么使用HTTP缓存

终端缓存策略，可以缩短端到端的请求资源的距离，减少延迟，而且缓存重用，也能减少宽带流量，降低网络负荷。
最终用户体验和性能得到优化，避免无用资源请求浪费

