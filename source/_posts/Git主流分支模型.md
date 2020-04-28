---
title: Git主流分支模型
date: 2020-04-27
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blogbranch_comparison.png
tags: 
- 软件开发模式
categories:
- [开发效率]
---

### 前言
分支模型的抉择可以概括为围绕 **持续集成** 和 **特性隔离** 两个特征进行博弈。

### 分支模型对比结论

#### 优缺点分析
[TBD说明](https://trunkbaseddevelopment.com/alternative-branching-models/index.html#modern-claimed-high-throughput-branching-models)

![](https://raw.githubusercontent.com/afree8909/pictures/master/blogbranch_comparison.png)



#### 使用分析

| 分支模型 | 主干数 | 特性分支数 | 集成频率 | 多版本并行开发 | 需求中途撤销 | 打包方式 |
| --- | --- | --- | --- | --- | --- | --- |
| Git Flow | 2 | 5类 | 特性分支完成后一起集成 | 特性分支 | 合并前：删除特性分支 合并后：手动剔除代码 | 开发分支和发布分支分别打包 |
| Aone FLow | 1 | 3类 | 指定特性分支频繁集成 | 特性分支且控制合并时间 | 删除特性分支重新集成 | 发布分支分别打包 |
| GitHub Flow | 1 | 1类 | 特性分支立即集成 | 特性分支 | 手工剔除代码 | 特性分支打包 |
| TBD | 1 | 1类 | 所有提交立即集成 | 特性开关 | 手工剔除代码 | 一次打包多次部署 |


### 分支模型详细分析
#### GitFlow
[详情参考](https://nvie.com/posts/a-successful-git-branching-model/)

##### 分支情况
* 主干分支（长期）
    * 主分支：master
    * 开发分支：develop
* 特性分支（短期）
    * 功能分支：feature
    * 预发分支：release
    * 补丁分支：hotfix

##### 玩法
* 开发&发布
    * develop分支创建feature分支
    * feature开发、测试完提pr到develop分支
    * code review 和合并进develop
    * 等待各个feature合并到develop
    * develop创建release分支并进行测试
    * release 开始发布，进行bug fix 且需要合并回develop
    * release 发布完成，merge到master和develop
* 修复
    * 通过tag创建对应hotfix进行修复，然后合并回develop和master



![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428122337.png)


#### GitHubFlow
[详情参考](https://guides.github.com/introduction/flow/)

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428122414.png)

##### 分支情况
主干分支：master
特性分支：feature

##### 玩法
开发：主分支创建feature分支进行开发、PR、Review、发布完成后，建立PR回master
修复：特性分支未合入master前特性分支修复，合入后针对tag单开分支修复并合入主干分支

它有一个变种版本，更好的支持多环境和多版本 ，可以参考 [GitLab Flow](https://docs.gitlab.com/ee/topics/gitlab_flow.html)

#### TBD
[详情参考](https://trunkbaseddevelopment.com/)

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428122414.png)


##### 分支情况
主干分支：master

##### 玩法
开发：所有团队成员都在单个主干分支上进行开发，
符合约定后commit到主干分支。也可创建短周期分支进行开发rebase主干分支后提交PR

发布：优先Tag，Tag不能满足则创建发布分支

修复：主干分支修复，cherry pick到发布分支，新tag与发布

其它辅助方案策略

* 如何避免引入未完成feature？ feature toggle（功能开关）
* 如果重构？ BBA（抽象分支）



#### AoneFlow
[详情参考](https://mp.weixin.qq.com/sJsBX3UPgZL_HUOTCIopr_A)
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428122511.png)
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428122517.png)
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428122521.png)




##### 分支情况
主干分支：master
特性分支：feature、release

##### 玩法
* 开始工作前，从主干创建特性分支。
* 通过合并特性分支，形成发布分支。
* 发布到线上正式环境后，合并相应的发布分支到主干，在主干添加标签，同时删除该发布分支关联的特性分支。







