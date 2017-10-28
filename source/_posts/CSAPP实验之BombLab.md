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
`phase_4`函数的反汇编代码和详细注释如下：
```
Dump of assembler code for function phase_4:
   0x000000000040100c <+0>: sub    $0x18,%rsp                    分配栈空间
   0x0000000000401010 <+4>: lea    0xc(%rsp),%rcx                为调用scanf函数构造参数，对应scanf第二个参数
   0x0000000000401015 <+9>: lea    0x8(%rsp),%rdx                为调用scanf函数构造参数，对应scanf第一个参数
   0x000000000040101a <+14>:    mov    $0x4025cf,%esi            %esi存储scanf函数格式化字符串 gdb中使用 x /s 0x4025cf查看格式化字符串
   0x000000000040101f <+19>:    mov    $0x0,%eax                 %eax保存scanf函数返回值

   0x0000000000401024 <+24>:    callq  0x400bf0 <__isoc99_sscanf@plt>  调用scanf参数

   0x0000000000401029 <+29>:    cmp    $0x2,%eax                 判断输入参数个数是否等于2
   0x000000000040102c <+32>:    jne    0x401035 <phase_4+41>     输入参数个数不等于2,调用explode_bomb
   0x000000000040102e <+34>:    cmpl   $0xe,0x8(%rsp)            判断输入第一个参数与0xe的大小
   0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>     第一个参数小于等于0xe跳转
   0x0000000000401035 <+41>:    callq  0x40143a <explode_bomb>   第一个参数大于0xe调用explode_bomb

   0x000000000040103a <+46>:    mov    $0xe,%edx                 为调用func4构造参数c,参数值为0xe
   0x000000000040103f <+51>:    mov    $0x0,%esi                 为调用func4构造参数b,参数值为0x0
   0x0000000000401044 <+56>:    mov    0x8(%rsp),%edi            为调用func4构造参数a,参数值为输入第一个参数值
   0x0000000000401048 <+60>:    callq  0x400fce <func4>          调用func4

   0x000000000040104d <+65>:    test   %eax,%eax                 测试func4返回值是否等于0
   0x000000000040104f <+67>:    jne    0x401058 <phase_4+76>     等于0继续执行,否则调用explode_bomb
   0x0000000000401051 <+69>:    cmpl   $0x0,0xc(%rsp)            测试输入第二个参数是否等于0
   0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>     等于0,跳转;否则调用explode_bomb
   0x0000000000401058 <+76>:    callq  0x40143a <explode_bomb>
   0x000000000040105d <+81>:    add    $0x18,%rsp
   0x0000000000401061 <+85>:    retq
End of assembler dump.
```
在`phase_4`函数中要求`func4`函数的返回值等于0，并且`func4`函数的参数为：第一个输入数，0，14。`func4`函数反汇编代码如下所示：
```
Dump of assembler code for function func4:
   0x0000000000400fce <+0>: sub    $0x8,%rsp
   0x0000000000400fd2 <+4>: mov    %edx,%eax                   result = c
   0x0000000000400fd4 <+6>: sub    %esi,%eax                   result = result - b
   0x0000000000400fd6 <+8>: mov    %eax,%ecx                   tmp = result
   0x0000000000400fd8 <+10>:    shr    $0x1f,%ecx              tmp = (unsigned)tmp >> 31
   0x0000000000400fdb <+13>:    add    %ecx,%eax               result = result + tmp
   0x0000000000400fdd <+15>:    sar    %eax                    result = result / 2
   0x0000000000400fdf <+17>:    lea    (%rax,%rsi,1),%ecx      tmp = result + b
   0x0000000000400fe2 <+20>:    cmp    %edi,%ecx   tmp <= a    
   0x0000000000400fe4 <+22>:    jle    0x400ff2 <func4+36>     
   0x0000000000400fe6 <+24>:    lea    -0x1(%rcx),%edx         c =  tmp - 1
   0x0000000000400fe9 <+27>:    callq  0x400fce <func4>        
   0x0000000000400fee <+32>:    add    %eax,%eax               
   0x0000000000400ff0 <+34>:    jmp    0x401007 <func4+57>     
   0x0000000000400ff2 <+36>:    mov    $0x0,%eax               result = 0
   0x0000000000400ff7 <+41>:    cmp    %edi,%ecx               tmp >= a
   0x0000000000400ff9 <+43>:    jge    0x401007 <func4+57>     
   0x0000000000400ffb <+45>:    lea    0x1(%rcx),%esi          b = tmp + 1
   0x0000000000400ffe <+48>:    callq  0x400fce <func4>
   0x0000000000401003 <+53>:    lea    0x1(%rax,%rax,1),%eax
   0x0000000000401007 <+57>:    add    $0x8,%rsp
   0x000000000040100b <+61>:    retq
End of assembler dump.
```
其中，`%rdi %rsi %rdx`依次保存第1，2，3个参数，分别对应于a b c；`%eax`表示返回值。另外定义局部变量`int result`, 保存在`%rax`作为返回值，定义局部变量`int tmp`，保存在`%rcx`。按照上述定义，获得`func4`函数对应的C语言代码：
```
int func4(int a, int b, int c)
{
    int result;
    result = c;
    result = result - b;
    int tmp = result;
    tmp = (unsigned)tmp >> 31;
    result = result + tmp;
    result = result / 2;
    tmp = result + b;
    if(tmp > a)
    {
        c = tmp - 1;
        result = func4(a, b, c);
        return (2 * result);
    }
    result = 0;
    if(tmp < a)
    {
        b = tmp + 1;
        result = func4(a, b, c);
        return (1 + 2 * result);
    }
    return result;
}
```
使用如下的测试程序，获得所有满足`phase_4`函数的破解密码：
```
int main()
{
    for(int input = 0; input < 15; ++input)
    {
        int result = func4(input, 0, 14);
        if(result == 0)
        {
            printf("input = %d, func4 = %d\n", input, result);
        }
    }
    return 0;
}
```
>因此phase_4破译可能结果为：
```
0 0
1 0
3 0
7 0
```

# phase_5
`phase_5`函数反汇编代码和详细注释如下所示：
```
(gdb) disassemble phase_5
Dump of assembler code for function phase_5:
   0x0000000000401062 <+0>: push   %rbx
   0x0000000000401063 <+1>: sub    $0x20,%rsp
   0x0000000000401067 <+5>: mov    %rdi,%rbx                             %rdi保存输入的字符串指针
   0x000000000040106a <+8>: mov    %fs:0x28,%rax
   0x0000000000401073 <+17>:    mov    %rax,0x18(%rsp)                   将%rax存储到0x18(%rsp)

   0x0000000000401078 <+22>:    xor    %eax,%eax                         清零%eax
   0x000000000040107a <+24>:    callq  0x40131b <string_length>          计算输入字符串长度
   0x000000000040107f <+29>:    cmp    $0x6,%eax                         判断输入字符串长度是否等于6
   0x0000000000401082 <+32>:    je     0x4010d2 <phase_5+112>
   0x0000000000401084 <+34>:    callq  0x40143a <explode_bomb>

   0x0000000000401089 <+39>:    jmp    0x4010d2 <phase_5+112>
   0x000000000040108b <+41>:    movzbl (%rbx,%rax,1),%ecx                复制输入字符串的第%rax个字符到%ecx中
   0x000000000040108f <+45>:    mov    %cl,(%rsp)                        将第%rax字符保存至(%rsp)中
   0x0000000000401092 <+48>:    mov    (%rsp),%rdx                       将第%rax字符复制到%rdx中
   0x0000000000401096 <+52>:    and    $0xf,%edx                         将第%rax字符最低4bit复制到%rdx最低4bit
   0x0000000000401099 <+55>:    movzbl 0x4024b0(%rdx),%edx               将与0x4024b0偏移量为%rdx的一个字节数据复制到%edx
   0x00000000004010a0 <+62>:    mov    %dl,0x10(%rsp,%rax,1)             将%edx最低字节复制到与%rsp偏移量为(0x10 + %rax)的栈地址中
   0x00000000004010a4 <+66>:    add    $0x1,%rax                         %rax值加1,指向下一个输入字符
   0x00000000004010a8 <+70>:    cmp    $0x6,%rax                         判断%rax是否等于6,不等于6继续循环
   0x00000000004010ac <+74>:    jne    0x40108b <phase_5+41>
   0x00000000004010ae <+76>:    movb   $0x0,0x16(%rsp)
   0x00000000004010b3 <+81>:    mov    $0x40245e,%esi                    %esi指向从0x40245e内存单元读入的字符串
   0x00000000004010b8 <+86>:    lea    0x10(%rsp),%rdi                   %rdi指向前面循环中构造好的长度为6的字符串
   0x00000000004010bd <+91>:    callq  0x401338 <strings_not_equal>      判断%esi和%rdi指向的字符串是否相等
   0x00000000004010c2 <+96>:    test   %eax,%eax
   0x00000000004010c4 <+98>:    je     0x4010d9 <phase_5+119>
   0x00000000004010c6 <+100>:   callq  0x40143a <explode_bomb>
   0x00000000004010cb <+105>:   nopl   0x0(%rax,%rax,1)
   0x00000000004010d0 <+110>:   jmp    0x4010d9 <phase_5+119>
   0x00000000004010d2 <+112>:   mov    $0x0,%eax
   0x00000000004010d7 <+117>:   jmp    0x40108b <phase_5+41>
   0x00000000004010d9 <+119>:   mov    0x18(%rsp),%rax
   0x00000000004010de <+124>:   xor    %fs:0x28,%rax
   0x00000000004010e7 <+133>:   je     0x4010ee <phase_5+140>
   0x00000000004010e9 <+135>:   callq  0x400b30 <__stack_chk_fail@plt>
   0x00000000004010ee <+140>:   add    $0x20,%rsp
   0x00000000004010f2 <+144>:   pop    %rbx
   0x00000000004010f3 <+145>:   retq
End of assembler dump.
```
使用gdb查看`0x4024b0`和`0x40245e`开始的内存单元的内容：
```
(gdb) x/32xb 0x4024b0
0x4024b0 <array.3449>:      0x6d    0x61    0x64    0x75    0x69    0x65    0x72    0x73
0x4024b8 <array.3449+8>:    0x6e    0x66    0x6f    0x74    0x76    0x62    0x79    0x6c
(gdb) x/s 0x40245e
0x40245e:   "flyers"
```
>`flyers`串对应的ascii值为`0x66 0x6c 0x79 0x65 0x72 0x73`，与`0x4024b0`内存地址开始的查找表比较获得偏移量为`0x9 0xF 0xE 0x5 0x6 0x72`。因此输入长度为6的字符串中每个字符的低4bit的值分别为`0x9 0xF 0xE 0x5 0x6 0x72`。所以，`phase_5`函数的破解密码存在两种情形：若输入为大写字母,将低4bit的值加上`0x40`,获得输入字符串`IONEFG`，若输入为小写字母,将低4bit的值加上`0x60`，获得输入字符串`ionefg`。

# phase_6
