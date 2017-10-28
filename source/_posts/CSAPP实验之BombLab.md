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
<!--more-->
## 栈示意图
在一个函数栈帧中，首先将被调用者保存寄存器中的值入栈，因为在被调用函数中，会使用上述寄存器，因此需要保存，并在函数返回时，`按照与入栈相反顺序重新出栈`；然后再保存函数中局部变量；最后，如果调用函数的参数多于6个，还需要按照从右至左顺序构造调用函数剩余的参数，栈示意图如下图所示。
<center>
![avatar](http://oyh38rhr2.bkt.clouddn.com/github/171028/dcm2AfeBgc.png)
</center>
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

# phase_2
在phase_2函数处设置断点，然后反汇编`phase_2`函数：
```
(gdb) disas phase_2
Dump of assembler code for function phase_2:
   0x0000000000400efc <+0>: push   %rbp
   0x0000000000400efd <+1>: push   %rbx
   0x0000000000400efe <+2>: sub    $0x28,%rsp
   0x0000000000400f02 <+6>: mov    %rsp,%rsi
   0x0000000000400f05 <+9>: callq  0x40145c <read_six_numbers>  读入6个值,保存至从%rsi开始的地址
   0x0000000000400f0a <+14>:    cmpl   $0x1,(%rsp)              读入第一个值是否等于1
   0x0000000000400f0e <+18>:    je     0x400f30 <phase_2+52>
   0x0000000000400f10 <+20>:    callq  0x40143a <explode_bomb>
   0x0000000000400f15 <+25>:    jmp    0x400f30 <phase_2+52>
   0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax          %eax保存前一个值
   0x0000000000400f1a <+30>:    add    %eax,%eax                将前一个值乘以2
   0x0000000000400f1c <+32>:    cmp    %eax,(%rbx)              判断后一个的值是否为前一个值的2倍
   0x0000000000400f1e <+34>:    je     0x400f25 <phase_2+41>
   0x0000000000400f20 <+36>:    callq  0x40143a <explode_bomb>
   0x0000000000400f25 <+41>:    add    $0x4,%rbx                %rbx获取下一个值的地址
   0x0000000000400f29 <+45>:    cmp    %rbp,%rbx                比较是否为最后一个值
   0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27>
   0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>
   0x0000000000400f30 <+52>:    lea    0x4(%rsp),%rbx           %rbx获得第二个值地址开始地址
   0x0000000000400f35 <+57>:    lea    0x18(%rsp),%rbp          %rbp获得最后一个值地址结束地址
   0x0000000000400f3a <+62>:    jmp    0x400f17 <phase_2+27>
   0x0000000000400f3c <+64>:    add    $0x28,%rsp
   0x0000000000400f40 <+68>:    pop    %rbx
   0x0000000000400f41 <+69>:    pop    %rbp
   0x0000000000400f42 <+70>:    retq
End of assembler dump.
```
在`phase_2`函数中调用`read_six_numbers`函数从输入中读入输入，所以继续反汇编出`read_six_numbers`函数：
```
(gdb) disas read_six_numbers
Dump of assembler code for function read_six_numbers:
   0x000000000040145c <+0>: sub    $0x18,%rsp
   0x0000000000401460 <+4>: mov    %rsi,%rdx
   0x0000000000401463 <+7>: lea    0x4(%rsi),%rcx
   0x0000000000401467 <+11>:    lea    0x14(%rsi),%rax
   0x000000000040146b <+15>:    mov    %rax,0x8(%rsp)
   0x0000000000401470 <+20>:    lea    0x10(%rsi),%rax
   0x0000000000401474 <+24>:    mov    %rax,(%rsp)
   0x0000000000401478 <+28>:    lea    0xc(%rsi),%r9
   0x000000000040147c <+32>:    lea    0x8(%rsi),%r8
   0x0000000000401480 <+36>:    mov    $0x4025c3,%esi
   0x0000000000401485 <+41>:    mov    $0x0,%eax
   0x000000000040148a <+46>:    callq  0x400bf0 <__isoc99_sscanf@plt>
   0x000000000040148f <+51>:    cmp    $0x5,%eax
   0x0000000000401492 <+54>:    jg     0x401499 <read_six_numbers+61>
   0x0000000000401494 <+56>:    callq  0x40143a <explode_bomb>
   0x0000000000401499 <+61>:    add    $0x18,%rsp
   0x000000000040149d <+65>:    retq
End of assembler dump.
```
在gdb中使用`x/s 0x4025c3`查看输入格式化字符串为：
```
(gdb) x/s 0x4025c3
0x4025c3:	"%d %d %d %d %d %d"
```
因此，phase_2函数调用`read_six_numbers`函数读入6个整型数据，。另外，从`read_six_numbers`调用`sscanf`函数前构造函数参数代码可知
```
%rsi:保存phase_2栈帧的局部变量开始地址
%rdx:指向%rsi + 0，即第一个读入参数地址，以下依照4Byte递增，依次保存2~6个局部变量地址
%rcx:%rsi + 4
%r8:%rsi + 8
%r9:%rsi + 12
(%rsp):%rsi + 16
8(%rsp):%rsi + 20
```
>在`phase_2`函数反汇编代码中，详细注释了每一个汇编语句的含义，很容易知道，该函数循环判断读入的数字中后一个数是否为前一个数的2倍，并且读入的第1个数必须为1。因此`phase_2`函数的破解密码为`1 2 4 8 16 32`

# phase_3
`phase_3`函数的反汇编代码和详细注释如下：
```
(gdb) disas phase_3
Dump of assembler code for function phase_3:
   0x0000000000400f43 <+0>: sub    $0x18,%rsp
   0x0000000000400f47 <+4>: lea    0xc(%rsp),%rcx               %rcx第2个参数地址
   0x0000000000400f4c <+9>: lea    0x8(%rsp),%rdx               %rdx第1个参数地址
   0x0000000000400f51 <+14>:    mov    $0x4025cf,%esi
   0x0000000000400f56 <+19>:    mov    $0x0,%eax
   0x0000000000400f5b <+24>:    callq  0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000400f60 <+29>:    cmp    $0x1,%eax                输入参数个数是否大于1
   0x0000000000400f63 <+32>:    jg     0x400f6a <phase_3+39>
   0x0000000000400f65 <+34>:    callq  0x40143a <explode_bomb>  输入参数个数小于等于1,调用explode_bomb
   0x0000000000400f6a <+39>:    cmpl   $0x7,0x8(%rsp)           将第一个参数转换成无符号后,再判断是否大于7
   0x0000000000400f6f <+44>:    ja     0x400fad <phase_3+106>   第一个参数大于7,调用explode_bomb
   0x0000000000400f71 <+46>:    mov    0x8(%rsp),%eax           将第一个参数值复制到%eax
   0x0000000000400f75 <+50>:    jmpq   *0x402470(,%rax,8)       switch语句跳转表,使用x/1xg 0x402470命令查看当%rax的值为0时跳转地址为 0x0000000000400f7c
   0x0000000000400f7c <+57>:    mov    $0xcf,%eax               将%eax的值设置为207
   0x0000000000400f81 <+62>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f83 <+64>:    mov    $0x2c3,%eax
   0x0000000000400f88 <+69>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f8a <+71>:    mov    $0x100,%eax
   0x0000000000400f8f <+76>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f91 <+78>:    mov    $0x185,%eax
   0x0000000000400f96 <+83>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f98 <+85>:    mov    $0xce,%eax
   0x0000000000400f9d <+90>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f9f <+92>:    mov    $0x2aa,%eax
   0x0000000000400fa4 <+97>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400fa6 <+99>:    mov    $0x147,%eax
   0x0000000000400fab <+104>:   jmp    0x400fbe <phase_3+123>
   0x0000000000400fad <+106>:   callq  0x40143a <explode_bomb>
   0x0000000000400fb2 <+111>:   mov    $0x0,%eax
   0x0000000000400fb7 <+116>:   jmp    0x400fbe <phase_3+123>
   0x0000000000400fb9 <+118>:   mov    $0x137,%eax
   0x0000000000400fbe <+123>:   cmp    0xc(%rsp),%eax          比较%eax值是否和输入第二个参数相等
   0x0000000000400fc2 <+127>:   je     0x400fc9 <phase_3+134>  不相等,则调用explode_bomb
   0x0000000000400fc4 <+129>:   callq  0x40143a <explode_bomb>
   0x0000000000400fc9 <+134>:   add    $0x18,%rsp
   0x0000000000400fcd <+138>:   retq
End of assembler dump.
```
其中关键在于意识到`0x0000000000400f81`地址的代码为`switch`语句的跳转表，能否破解关卡的密码在于输入的两个参数中第一个参数作为`switch`语句的参数，第2输入个参数是否和`switch`语句的返回值相等。因此，使用gdb查看`0x402470`开始的地址的内存单元的内容获得`switch`跳转表如下所示：
```
(gdb) x/1xg 0x402470
0x402470:	0x0000000000400f7c
(gdb) x/1xg 0x402478
0x402478:	0x0000000000400fb9
(gdb) x/1xg 0x402480
0x402480:	0x0000000000400f83
(gdb) x/1xg 0x402488
0x402488:	0x0000000000400f8a
(gdb) x/1xg 0x402490
0x402490:	0x0000000000400f91
(gdb) x/1xg 0x402498
0x402498:	0x0000000000400f98
(gdb) x/1xg 0x4024a0
0x4024a0:	0x0000000000400f9f
(gdb) x/1xg 0x4024a8
0x4024a8:	0x0000000000400fa6
```
由跳转表从而获得`switch`语句返回值如下：
```
%rax(输入参数1)       跳转地址            0xc(%rsp)(输入参数2)
0               0x0000000000400f7c       0xcf  207
1               0x0000000000400fb9       0x137 311
2               0x0000000000400f83       0x2c3 707
3               0x0000000000400f8a       0x100 256
4               0x0000000000400f91       0x185 389
5               0x0000000000400f98       0xce  206
6               0x0000000000400f9f       0x2aa 682
7               0x0000000000400fa6       0x147 327
```
>所以，`phase_3`函数的破解密码存在上述多组。

# phase_4
