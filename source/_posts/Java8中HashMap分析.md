---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428155317.png
tags: 
- 源码
- 集合
categories:
- [Java, 数据结构]
---


### Put方法流程图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428155317.png)

### 特殊值
#### hash
高16bit不变，低16bit和高16bit做了一个异或

```
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

```
#### size
* map中存储的KV数目
* 值为2的n次幂
    * 哈希表是通过除法散列法，取模会用到除法运算，效率很低，而HashMap通过 h&（length－1）替代取模
    * length 为2的整数次幂，是为了使不同hash值发生碰撞概率小即更均匀散列
    * length 的值为100..0，length－1为011..1，相当于对取模，而且保证了hash可以奇偶都有
    
#### threshold
* 阀值，=容量*loadFactor（负载因子，默认0.75）

### 扩容机制
* hash与旧链表大小做 & 运算，=0不变，=1移动到原位置+旧链表大小的位置 
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428155324.png)


### 线程是否安全
* JDK1.7，扩容前后链表导致，转移过程中修改了原来链表中节点引用关系，可能导致死循环
* JDK1.8，不会引起死循环，但put／get不一定同步




