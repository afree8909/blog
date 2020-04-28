---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428154534.png
tags: 
- 源码
categories:
- [Java, 系统]
---

### JVM类加载器
#### 概念
“通过一个类的全限定名来获取描述此类的二进制字节流。” ——实现这个动作的代码模块称为类加载器。
#### 加载阶段
* 根据一个类的全限定名来获取定义此类的二进制字节流
* 将这个字节流所代表的静态存储结构转化为 JVM 方法区中的运行时数据结构
* 在内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区这个类的各种数据的访问入口。

#### 扩展
* 如果想维持双亲委派机制，则覆写findClass方法
* 如果想打破双亲委派机制，则覆写loadClass方法

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428154534.png)


#### 示例
##### Tomcat类加载器
* Java类库的隔离：不同应用程序使用不同的类加载器，可以实现Java类库的隔离。

* Java类库的共享：通过Common类加载器和Shared类加载器可以实现Java类库的共享。

* 安全：服务器和应用程序有各自的类加载器加载Class文件。服务器的类库与应用程序的类库可以互相独立。

* 支持HotSwap（热替换）：JSP文件有独立的类加载器，服务器能通过替换JSP文件的类加载器来实现JSP的HotSwap功能。
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428154542.png)


### Android类加载图
#### 加载阶段
通过一个dex文件或者apk文件的路径完成类的加载
#### 类加载器
* BootClassLoader，主要用于加载系统的类，包括java和Android系统的类库
* PathClassLoader，主要用于加载应用内中的类，路径是固定的，只能加载/data/app中的apk，无法指定dex路径
* DexClassLoader，可任意加载.apk、zip或jar等，实现动态加载

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428154548.png)


#### TODO
类加载器相关应用，插件化、热补等


