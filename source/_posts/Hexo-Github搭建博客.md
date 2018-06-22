---
title: Hexo+Github搭建博客
tags:
  - 技术点滴
toc: true
date: 2018-06-22 22:56:45
---
# 前言
前期使用`Hexo+Github`搭建部署了个人博客，并且在Github上使用分支技术，其中`hexo`分支用于编写博客，`master`分支用于发布博客。因为很长一段时间没有使用博客系统，关于博客的部署又得重新查找资料，故将个人博客的重新部署归纳总结。

# Hexo安装
## 安装Node.js和Git
直接在官网下载最新的[Node.js](https://nodejs.org/en/)和[Git](https://git-scm.com/)。安装完成后，启动`Git Bash`，使用`git --version,node -v和npm -v`检测安装是否成功。
## 安装Hexo
在`Git Bash`中，使用`npm install hexo --save`安装Hexo；然后使用`hexo -v`检测安装是否成功。
<!--more-->

# 配置SSH
使用如下命令生成ssh信息：
```
ssh-keygen -t rsa -C "295861542@qq.com"
```
找到`~/.ssh/id_rsa.pub`文件并复制里面的内容，登录Github并添加密钥，将复制的内容添加到`SSH Key`中，然后使用如下命令检验是否添加成功：
```
ssh -T git@github.com
```
添加SSH Key之后，再配置git的全局信息和命令别名：
```
git config --global user.name zhoulee
git config --global user.email 295861542@qq.com
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```
配置完git信息后，从Github克隆个人博客仓库：
```
git clone https://github.com/octocat9lee/octocat9lee.github.io.git
```
最后，切换到`hexo`分支，使用`npm install`命令安装所需的模块。至此，关于博客基础环境搭建完毕。

# Hexo命令简介
```
hexo g # 生成静态文件
hexo s -p 4001 # 部署监听4001端口
hexo d # 部署
hexo d -g # 生成并部署到Github
hexo new "Title" # 新建一篇文章
hexo version # 查看版本
hexo clean # 清除缓存文件和已生成的静态文件
```
