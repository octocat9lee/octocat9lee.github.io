---
title: GDB调试器使用总结
tags:
  - 调试技术
toc: true
date: 2017-11-09 13:40:24
---
# 前言
GDB是GNU开源组织发布的一个强大的UNIX下的程序调试工具。

# 输出线程堆栈
在调试多线程程序时，经常需要查看线程堆栈信息，如果线程数目过多，每次查看一个线程堆栈，繁琐耗时。下面介绍一种一次性将所有线程堆栈输出到文件的方法：
``` bash
# 将gdb attach到调试进程
gdb -p pid
# 在GDB中设置调试文件路径,并开启日志选项
set logging file thread_stack.info
set logging on
# 输出所有线程堆栈到指定文件
thread apply all bt
# 或者简化命令
thr app all bt
```
直接输出所有线程堆栈信息到指定文件
```
gdb -ex "thread apply all bt" -batch -p pid > thread_stack.info
gdb -ex "thread apply all bt" -batch -p `ps -e | grep procname | awk '{print $1}'` > thread_stack.info
```
<!--more-->

# 调用函数
``` bash
gdb>call function();
```

# 网络资源
[《100个gdb小技巧》](https://gitlore.com/subject/15/src/print-threads.md)
