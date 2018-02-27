---
title: Git操作指南
tags:
  - 技术点滴
toc: true
date: 2017-11-24 15:07:19
---
# 全局配置与帮助
>`git config --global user.name "your name"`
`git config --global user.email "email@gmai.com"` `--global`表示机器上所有Git仓库都使用该配置

# 版本库管理
## 创建版本库
>步骤1:本地创建版本库目录
步骤2: 使用`git init`将目录变成git可以管理的仓库，生成.git目录

## 添加文件
>步骤1: `git add file` 把文件添加到仓库
步骤2: `git commit -m "comment"` 把文件提交到仓库

## 查看状态
>`git status` 显示仓库当前状态
`git diff file` 显示指定文件差异
`git show commitId ` 显示某次提交的修改内容
`git show <commit-hashId> filename` 显示某次提交某个文件的修改信息
`git log`  显示提交日志记录
`git log --pretty=online --graph --color`
`git reflog` 查看命令历史，以便确认回到未来哪个版本

<!--more-->
## 版本回退
>`git reset --hard commit_id` HEAD指向当前版本；HEAD^上个版本；上100个版本 HEAD~100

## 管理修改
>git自动创建称为stage的暂存区
`git add`实际上是把文件修改添加到暂存区
`git commit`实际上是把暂存区的所有内容提交到当前分支
git管理的是修改，而不是文件，每次修改如果不add到暂存区，就不会加入到commit中

## 撤销修改
>`git checkout -- file`丢弃工作区修改，`--`必须存在，否则表示切换分支
添加到了暂存区时但未提交，想丢弃修改
步骤1：`git reset HEAD file` 暂存区修改撤销，重新放回工作区
步骤2：`git checkout -- file` 丢弃工作区修改

## 删除文件
>`git rm file`
`git checkout -- file` 恢复误删

# 分支管理
## 创建分支
>`git branch -a` 查看当前分支
`git branch branchname` 创建分支
`git checkout branchname` 切换分支
`git checkout -b branchname` 创建分支同时切换到创建分支
`git checkout -b branchname master` 从指定的master分支创建分支并切换到branchname分支
`git branch -d branchname` 删除分支
`git branch -m master altername` 将master分支名称修改为altername

## 合并分支
通常，合并分支时，如果可能，Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。
>`git merge --no-ff -m "merge with no-ff" dev` 将`dev`分支合入当前分支，其中`-m`表示commit描述

## 分支策略
在实际开发中，我们应该按照几个基本原则进行分支管理：
>首先，`master`分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；
那在哪干活呢？干活都在`dev`分支上，也就是说，`dev`分支是不稳定的，到某个时候，比如1.0版本发布时，再把`dev`分支合并到`master`上，在`master`分支发布1.0版本；
你和你的小伙伴们每个人都在`dev`分支上干活，每个人都有自己的分支，时不时地往`dev`分支上合并就可以了

## 多人协作
当你从远程仓库克隆时，实际上Git自动把本地的`master`分支和远程的`master`分支对应起来了，并且，远程仓库的默认名称是`origin`。
>`git remote -v` 查看远程库信息
`git push origin branch-name` 推送本地分支，如果推送失败，先用`git pull`抓取远程新的提交
`git checkout -b branch-name origin/branch-name` 在本地创建和远程分支对应的分支，本地和远程分支的名称最好一致
`git checkout -t origin/branch-name` 默认会在本地建立一个和远程分支名字一样的分支
`git branch --set-upstream branch-name origin/branch-name` 建立本地分支和远程分支的关联
`git pull` 从远程抓取分支，如果有冲突，要先处理冲突

## 多次commit合并为一次commit
有的时候我们提交了多次commit，但是这些历史记录我们没有必要都要放到远程服务器上，在推送到远端时，需要在合并的时候先合并一下
``` bash
首先我们在master分支上创建一个新分支，叫dev：
git checkout -b dev
然后我们在dev分支上提交三次更新，分别取名为first commit 、second commit、third commit。
将多次提交合并为一次：
git merge dev --squash
一定要注意，git merge后一定要commit一下
git commit -m "update README.md"
```
在同一分支中使用`git rebase -i HEAD~3`的交互模式将多次的提交合并成一次提交。
具体的`git rebase -i`详细介绍：[Doing git rebase --interactive, including merge conflicts](https://www.youtube.com/watch?v=_ajGPzBBOoQ)

## 拣选合并
有时候分之间只需要合并一个提交，而不需要合并一条分支上的全部改动。
``` bash
git cherry-pick commitId #合并指定commitId到当前分支
git cherry-pick -n commitId #拣选多个提交
```

## 合并远程最新代码
在多人协同开发中，我们经常会遇到这样的问题：A在本地开发完成后，将代码推送到远程，这时候B的本地代码的版本就低于远程代码的版本，这时候B该如何从远程拉取最新的代码，并与自己的本地代码合并呢？
具体步骤如下：
``` bash
从远程origin仓库获取master分支最新版本代码到本地的tmp分支
git fetch origin master:tmp
查看tmp分支与本地分支的差异
git diff tmp
将tmp分支与本地分支合并，B的本地代码已经和远程仓库处于同一个版本了
git merge tmp
最后，可以删除tmp分支
git branch -d tmp
```

## 多人协作开发模式
>当从远程库clone时，默认情况下，只能看到本地的`master`分支。现在，要在dev分支上开发，就必须创建远程`origin`的`dev`分支到本地，创建本地dev分支:
``` bash
 git checkout -b dev origin/dev
```
>在`dev`上继续修改，然后，时不时地把`dev`分支push到远程：
``` bash
git commit -m "fix bug"
git push origin dev
```
>如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并
如果合并有冲突，则解决冲突，并在本地提交
没有冲突或者解决掉冲突后，再用`git push origin branch-name`推送至远端

# 标签管理
发布一个版本时，我们通常先在版本库中打一个标签（tag），这样，就唯一确定了打标签时刻的版本。将来无论什么时候，取某个标签的版本，就是把那个打标签的时刻的历史版本取出来。所以，标签也是版本库的一个快照。
## 创建标签
>`git tag` 查看所有标签
`git tag -a tagname -m "tag comment"` 用于新建一个标签，默认为`HEAD`
`git tag -a tagname -m "tag comment" commit-id` 对指定的commitId创建标签

## 推送标签
>`git push origin tagname` 推送本地一个标签
`git push origin --tags` 推送本地全部未推送的标签

## 删除标签
>`git tag -d tagname` 删除本地一个标签
`git push origin :refs/tags/tagname` 删除远程标签

# 日志
## 配置别名
>`git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"`

## 查看日志
>`git lg file.c` 查看file.c文件的提交记录
`git lg dir/`   查看dir目录下提交记录
`git lg --grep "opencl"` 匹配日志信息中指定的的关键字
`git show commitId` 显示某次提交的详细信息
`git blame somefile` 显示文件各个部分的修改作者及相关提交信息
`git blame -M somefile`
`git blame -C -C somefile`

# 参考资料
<strong>[Git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)</strong>
<strong>[Pro Git](https://git-scm.com/book/zh/v2#%E5%9F%BA%E6%9C%AC%E7%9A%84%E8%A1%8D%E5%90%88%E6%93%8D%E4%BD%9C)</strong>
<strong>[Git分支Rebase详解](http://blog.csdn.net/endlu/article/details/51605861)</strong>
<strong>[Git Merge和Git Rebase小结](http://blog.csdn.net/wh_19910525/article/details/7554489)</strong>
