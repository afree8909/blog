---
tags: 
- ID
categories:
- [Android, 应用]
---

### 目的
希望自主生成用于所有移动业务上标识设备唯一性的标识符


### 作用
业务支撑、日活和新增设备等数据统计、风控等
### uuid系统图
![](http://upload-images.jianshu.io/upload_images/9696036-3430540af001a4ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### UUID生成
#### 移动端相关参数
##### Android
    IMEI
 
    * International Mobile Equipment Identity，移动设备国际识别码，是手机的唯一识别号码。等价于DeviceId
    * 仅支持具有通话功能的设备，例如平板没有
    * 需要权限，并且在少数手机拿到的是不合法值
    * 一定的重复率（约5%）

    MAC Address

    * 支持带有Wifi和蓝牙硬件的设备
    * 需要权限

    Sim Serial Number

    * Sim卡序列号
    * 对于CDMA设备，返回一个空值

##### iOS
    IDFA: 广告标志符,重置系统、还原广告会重新生成）
    OpenUDID: 由OpenUDID的SDK包生成，该应用重置则会重新生成
    keychain: 系统级存储，用于生成的标识存储
    MACAddress和UDID:老版本兼容按需使用
#### uuid生成算法
相关参数：客户端类型、版本号、设备信息长度、设备信息内容（见上）、时间戳、设备信息类型、随机数
原则：生成uuid唯一，设备信息可反查，设备信息关联uuid，设备信息可校验等
算法：32位明文存储相关信息，AES64对称加密

#### 表结构

| uuid | imei | mac | simid | idfa | keychainid  | openudid | idfa | udid | Xxx |
| --- | --- | --- | --- | --- | --- | --- | --- | :-- | :-- |


### 相关问题
uuid没法保存
uuid合法性（去重、是否作弊）
兼容已有uuid
防止刷接口
防止伪造uuid









