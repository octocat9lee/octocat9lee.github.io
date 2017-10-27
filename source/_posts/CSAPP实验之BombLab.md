---
title: CSAPP实验之BombLab
tags:
  - 技术点滴
toc: true
date: 2017-10-27 17:32:56
---
本实验主要目的熟悉汇编指令以及gdb调试命令，掌握过程调用中栈的变化
# phase_1
下载实验代码，解压，进入工作目录。首先，在`solutions.txt`文件中输入任意字符串；然后使用gdb命令`gdb ./bomb`进入gdb调试，在gdb中使用`set args ./solutions`设置破解密码保存文件名；最后，使用`break phase_1`命令设置断点，运行程序。程序将在phase_1处进入断点，使用`disas phase_1`反汇编phase_1获得对应的汇编代码如下：
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
>在GDB中使用x/s 0x402400查看0x402400内存单元中字符串的内容,从而获得密码

<!--more-->
