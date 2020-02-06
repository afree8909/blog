
---
tags: 
- 源码
- 生命周期
categories:
- [Java, 系统]
---

### Java对象生命周期图
![](http://upload-images.jianshu.io/upload_images/9696036-0dd562d1863f77b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 创建阶段（Created）
* 分配存储空间
* 开始构造对象
* 父类到子类依次初始化类变量
* 父类成员初始化，父类构造函数初始化
* 子类成员初始化，子类构造函数初始化

### 应用阶段（In Use）
对象被一个或多个强引用持有，并且在作用域内

### 不可见阶段（Invisible）
其它区域已经不可以再引用它，本地变量超出了可见范围

```
void test(){
    try{
        Object a = new Object();
    }catch(Exception e){}
    a.clone(); // 该区域a不可见
}
```

### 不可达阶段（Unreachable）
不再被任何强引用所持有

### 收集阶段（Collected）
对象不可达，并且垃圾回收器已经对该对象的内存空间重新分配做好准备时

### 终结阶段（Finalized）
对象执行完finalize方法后，等待垃圾回收器对对象空间进行回收

### 对象重新分配阶段（Deallocated）
垃圾回收器对该对象的所占用的内存空间进行回收或者再分配了，则该对象彻底消失了，此时称为对象空间重新分配阶段

### 子父类代码执行顺序图
![](http://upload-images.jianshu.io/upload_images/9696036-ad430f40c254dee1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




