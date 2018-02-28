---
title: Linux性能分析工具：ftrace
tags:
  - 性能分析
toc: true
date: 2018-02-28 13:50:15
---
本文主要介绍ftrace的基本功能和用法。ftrace用于跟踪时延和行为，它让我们可以很深入地了解系统的运行行为，是进行Linux性能调优必须掌握的基本工具。
<!--more-->
# 基本使用
文件set_ftrace_pid的值要更新为你想跟踪的进程的PID。当跟踪完成后，你需要清除set_ftrace_pid文件，用如下命令：
``` bash
root@tracing$ echo 2576 > set_ftrace_pid #设置pid
root@tracing$> set_ftrace_pid #清除pid
```

# ftrace脚本

# 参考资料
[ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/)
[在Linux下做性能分析2：ftrace](https://zhuanlan.zhihu.com/p/22130013)
[使用 ftrace 调试 Linux 内核，第 1 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace1/)
[使用 ftrace 调试 Linux 内核，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace2/)
[使用 ftrace 调试 Linux 内核，第 3 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace3/)
[Linux内核调试工具 Ftrace 进阶使用手册](http://blog.csdn.net/longerzone/article/details/16884703)
