---
title: grep
tags:
  - 调试技术
toc: true
date: 2017-10-30 15:29:36
---
# grep常用选项
对于基本常用的命令，如果每次去百度或者Google命令的基本使用方式，因为信息的杂乱，我们很难快速检索到自己需要的信息。因此，作为一名专业的程序员，我们应该逐步建立自己的知识地图，不断总结完善自己的知识体系，当以后再遇到类似问题时，我们只需要回忆类似的场景获取对应的关键字，然后查阅自己的笔记，那么我们解决问题也会快速很多，并且也再次加深对之前知识的理解。`好记性不如烂笔头`，下面将在log定位检索时常用的`grep命令`选项总结如下：
查看指定内容对应的行号并显示改行
``` bash
grep -rn "search-content" < file.log
```
查看指定内容对应的行号但忽略大小写
``` bash
grep -irn "search-content" < file.log
```
在指定的文件类型中查找指定的内容
``` bash
find /path/to/ -regex '.*\.h\|.*\.cpp' | xargs grep -rn "search-content"
```
<!--more-->
查找匹配行前后内容
``` bash
grep -A 20 -B 10 -irn "search-content" < file.log
```
匹配单词的全部内容
``` bash
grep -wrn "search-content" < file.log
```
匹配指定内容之外的行
``` bash
grep -vrn "search-content" < file.log
```
匹配范本文件中每一行，如果在范本文件中指定多个匹配的串，可以同时匹配任意的一行
``` bash
grep -f fromat.pat < file.log
```
在程序调试时，需要查找多个关键之时，该选项特别有用！
