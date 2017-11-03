---
title: coredump
tags:
  - 调试技术
toc: true
date: 2017-10-28 23:13:11
---
# coredump介绍
应用程序有时会因为异常或者bug导致在运行过程中异常退出或者终止，为了方便问题的定位，我们往往需要获取程序运行时的内存，寄存器状态，堆栈指针，内存管理以及函数调用堆栈信息等，从而找到bug所在。在linux系统中，我们通常可以通过对系统进行一些配置，将上述的信息输出到ELF文件中，即core文件。core文件默认存储位置与对应的可执行程序位于同一路径下，文件名为core；具体信息可以通过如下命令查看：
``` bash
cat /proc/sys/kernel/core_pattern
```
如果程序中调用了chdir函数，则core文件生成在chdir指定路径下。另外，我们也可以通过相关设置设定core文件生成路径。在类unix系统中，core文件本身的主要格式也是ELF格式，因此可以使用readelf或者file命令查看core文件的具体信息。
<!--more-->

# coredump基本配置
调试coredump文件需要读取相应符号信息。因此，编译时需要加入`-g`选项。因为系统资源限制，所以默认情况下，core文件大小限制为0，不会生成core文件，需要进行设置。命令如下：
``` bash
ulimit -c unlimited  (coredump文件大小不受限制)
```
进程资源限制，有时即使系统设置了unlimited，但是当前进程启动时如果没有读取到，则仍然不会生成core文件。进程限制内容查看命令：
``` bash
egrep "Units|core" /proc/<pid>/limits
```
## 配置core文件位置和文件名
使core文件添加pid信息
``` bash
echo "1" > /proc/sys/kernel/core_uses_pid
```
生成的core文件与执行程序位于相同路径，文件名为core.pid.execute_name
``` bash
echo "./core.%p.%e" > /proc/sys/kernel/core_pattern
其他控制参数：
%p - insert pid into filename 添加pid
%u - insert current uid into filename 添加当前uid
%g - insert current gid into filename 添加当前gid
%s - insert signal that caused the coredump into the filename 添加导致产生core的信号
%t - insert UNIX time that the coredump occurred into filename 添加core文件生成时的unix时间
%h - insert hostname where the coredump happened into filename 添加主机名
%e - insert coredumping executable name into filename 添加命令名
```

# 未生成core文件定位
+ 检查core目录，core文件生成路径所在磁盘空间是否为0，或者小于进程运行时所占的虚拟内存大小，通过top命令查看VIRT。
+ 检查core目录是否有写权限
+ 如果发现core文件不知道属于哪个进程，执行如下命令：
``` bash
strings core.6440 | head
```
