---
title: CSAPP之Y86-64模拟器安装
tags:
  - CSAPP
toc: false
date: 2017-10-31 09:33:48
---
在CSAPP第4章处理器体系结构中，需要使用Y86-64模拟器，在安装模拟过程中遇到一些问题，下面记录自己的安装过程：
## 安装环境
因每个人操作系统环境的不一致，因此下面列出自己的安装环境：
``` bash
root@sim$ uname -a
Linux zhoulee 4.10.0-37-generic #41~16.04.1-Ubuntu SMP Fri Oct 6 22:42:59 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

## 下载模拟器源码
首先，到CS:APP3e官方的[学生主页](http://csapp.cs.cmu.edu/3e/students.html)下载最新的Y86-64模拟源码，源码位于`Chapter 4: Processor Architecture`。
然后，在工作目录解压对应源码文件，我们假设编译无图形界面版本，编辑`Makefile`，注释下面两个选项：
``` bash
TKLIBS=-L/usr/lib -ltk -ltcl
TKINC=-isystem /usr/include/tcl8.5
```
最后，执行`make clean && make`编译生成Y86-64工具集。

## 安装问题
缺少词法分析器，链接时为找到`/usr/bin/ld: cannot find -lfl`
``` bash
apt-get install flex
```
缺少语法分析器，编译中出现`make[1]: bison: Command not found`
``` bash
apt-get install bison
```
<!--more-->
