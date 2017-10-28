---
title: CSAPP实验之BombLab
tags:
  - 技术点滴
toc: true
date: 2017-10-27 17:32:56
---
# 前言
“二进制炸弹”是作为目标代码文件提供给学生的程序。运行时，它提示用户键入6个不同的字符串。如果这些中的任何一个是不正确的，炸弹“爆炸”，打印一个错误消息。我们必须通过拆卸和反向工程来确定6个字符串应该是什么，来“消除”自己独特的炸弹。本实验的主要目的是熟悉汇编语言，并强制学习如何使用调试器。
本实验详细项目文件以及分析解决方案参见[BombLab](https://gitee.com/zhoulee/CSAPP/tree/master/bomb)，下载实验代码，解压，进入工作目录，下面进入惊险刺激的的破译之旅。
# 基本知识
## 寄存器基本使用规律
```
%rbp %rbx %r12~%15 被调用者保存寄存器
%r10 %r11 调用者保存寄存器
%rax 保存函数返回值
%rdi %rsi %rdx %rcx %r8 %r9 依次保存函数参数1~6
```
## 汇编指令阅读
指令编写方式：`指令名  源操作数  目的操作数`源操作数和目的操作数不能同时都为内存单元
对于源操作数和目的操作数中，除lea指令外，其他包含括号的指令均为访问对应的内存单元的值，在C语言中，可以将该变量视为指针变量。
cmp指令使用`目的操作数-源操作数`，test指令使用`目的操作数&源操作数`，接下来的条件指令根据命令规则进行跳转。
## gdb基本调试命令
<center>
![avatar](http://oyh38rhr2.bkt.clouddn.com/gdb_debug.png)
</center>
在gdb调式过程中，可以使用`help`命令查看对应的帮助信息，例如`help x`查看`x`命令的使用方式。
# phase_1
首先，在`solutions.txt`文件中输入任意字符串；然后使用gdb命令`gdb ./bomb`进入gdb调试，在gdb中使用`set args ./solutions`设置破解密码保存文件名；最后，使用`break phase_1`命令设置断点，运行程序。程序将在phase_1处进入断点，使用`disas phase_1`反汇编phase_1获得对应的汇编代码如下：
```
(gdb) disas phase_1
Dump of assembler code for function phase_1:
   0x0000000000400ee0 <+0>: sub    $0x8,%rsp
   0x0000000000400ee4 <+4>: mov    $0x402400,%esi                 将立即数0x402400复制到%esi
   0x0000000000400ee9 <+9>: callq  0x401338 <strings_not_equal>   比较从地址0x402400开始的字符串和读入的指向的字符串值%rdi是否相等
   0x0000000000400eee <+14>:    test   %eax,%eax
   0x0000000000400ef0 <+16>:    je     0x400ef7 <phase_1+23>
   0x0000000000400ef2 <+18>:    callq  0x40143a <explode_bomb>
   0x0000000000400ef7 <+23>:    add    $0x8,%rsp
   0x0000000000400efb <+27>:    retq
End of assembler dump.
```
根据寄存器使用规则，可知`%rdi`保存输入字符串，`%esi`保存`strings_not_equal`函数第二个参数内容。另外，从函数名称可以该函数比较两个字符串是否相等。
>在GDB中使用x/s 0x402400查看0x402400内存单元中字符串的内容,从而获得密码
```
(gdb) x/s 0x402400
0x402400:	"Border relations with Canada have never been better."
```
<!--more-->
# phase_2
