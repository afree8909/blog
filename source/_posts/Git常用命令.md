---
tags: 
- git
- 命令
categories:
- [开发效率]
- [工具]
---


![经典Git操作与对应区域图](https://upload-images.jianshu.io/upload_images/9696036-f719bd62f295b4fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Git库创建

```
// 远程仓库克隆到本地
git clone [ssh]

// 本地构建仓库
git init

```

## Git配置
以MAC系统，Git配置文件一般有两个配置文件，其作用域分别为全局级、仓库级

1. 全局级配置： ~/.gitconfig
2. 仓库级配置： ${ProjectFile}/.git/.gitconfig

```

// 查看当前配置信息
git config -l

// 设置git用户信息
git config [--local | --global] user.name "[name]"
git config [--local | --global] user.email "[email address]"

```


## Git分支相关

### 查看

```
// 列出所有本地分支
git branch 

// 列出所有远程分支
git branch -r

// 列出所有本地分支和远程分支
git branch -a

// 列出本地分支及其对应的远端分支，并附最新commit信息 （-v ）
git branch -vv 
```

### 新建

```
// 新建一个分支，但依然停留在当前分支
git branch [branch-name]

// 新建一个分支，指向指定commit
git branch [branch] [commit]

// 如果本地有branch-name分支，则切换到该分支，如果没有切远程有branch-name分支，则直接以远程分支为基准创建branch-name本地分支
git checkout [branch-name]

// 新建一个分支，并切换到该分支
git checkout -b [branch-name]

// 执行本地分支 branch 追踪远端 remote-branch
git branch --set-upstream [branch] [remote-branch]

```

### 同步&合并
```
// 同步远端变到本地
git fetch

// 同步远端变更，并与本地分支合并，可能有conflict
git pull

// 合并N个提交记录为一个
git rebase -i HEAD~N

// 场景：个人分支 rebase 其它分支 ，（分支commit记录更清爽，相比merge操作少一个Merge commit）
git rebase [branch]

// 合并指定分支到当前分支
git merge [branch]

```

### 推送
```
// 上传当前分支到跟踪的远端分支
git push 

// 上传本地指定分支到远程仓库
git push [remote] [branch]

// 强行推送当前分支到远程仓库，即使有冲突
git push [remote] --force

```

### 删除
```
// 删除本地分支
git branch -d [branch-name]

// 删除远端分支
git push origin --delete [branch-name]
```


## Git提交相关

### 查看
```

// 显示当前分支的提交历史
git log


// 显示当前分支的最近几次提交
git reflog

```


### 提交

```
// 提交暂存区到本地仓库区
git commit 

// 提交快捷方式 带message
git commit -m [message]

// 提交到上一次commit，可变更message
git commit --amend -m [message]

// 将某个commit，合并到当前分支
git cherry-pick [commit]
```

### 撤销

```
// 重置暂存区的指定文件，与上一次commit保持一致，但工作区不变
git reset [file]

// 重置暂存区与工作区，与上一次commit保持一致
git reset --hard

// 重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
git reset [commit]

// 重置当前分支的HEAD为指定commit，同时重置暂存区和工作区，与指定commit一致
git reset --hard [commit]

// 新建一个提交，并对指定commit后的所有变更进行回滚
git revert [commit]
```

## Git文件相关

### 查看

```
// 显示变更的文件
git status

// 显示暂存区和工作区的差异
git diff
```

### 操作
```
// 添加指定文件到工作区
git add [file]

// 添加所有文件到工作区
git add .

// 移除工作区指定文件
git rm [file]

```

### 撤销
```
// 恢复暂存区的指定文件到工作区
git checkout [file]

// 恢复暂存区的所有文件到工作区
git checkout .

// 将所有工作区文件 存储到stash区
git stash

// 将stash 存储区最上面的一个，恢复到工作区
git stash pop

```

## Git标签

```

// 列出所有的标签
git tag

// 新建一个tag
git tag [tag]

// 指定commit傻姑娘新建一个tag
git tag [tag] [commit]

// 删除本地tag
git tag -d [tag]

// 删除远程tag
git push origin :refs/tags/[tagName]

// 推送指定tag
git push [remote] [tag]

// 以指定某个分支的tag处新建一个分支
git checkout -b [branch][tag]

```

>

![](https://upload-images.jianshu.io/upload_images/9696036-70673e2d55e85b18.gif?imageMogr2/auto-orient/strip)


