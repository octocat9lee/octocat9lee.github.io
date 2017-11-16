---
title: CSAPP实验之archlab
tags:
  - CSAPP
toc: true
date: 2017-11-02 17:11:57
---
# 前言
archlab是CSAPP3e第4章`处理器体系结构`对应的实验，该实验主要目的是学习Y86-64指令集体系结构，HCL控制语言以及Y86-64的流水线实现。因此，对应的实验也分为3个部分一一与之对应，现在暂时仅完成该实验的`Part A`部分。因为课程主页上的项目不断更新，很难维持博文的内容与课程主页上的实验内容保持一致，所以将本文章对应的程序源码保存在个人项目中[archlab](https://gitee.com/zhoulee/CSAPP/tree/master/archlab)，。

# Part A
该部分内容比较简单，就是使用Y86-64指令集体系实现`example.c`文件中对应的3个函数，然后使用`yas`编译器编译对应的汇编文件，最后使用`yis`模拟器模拟程序的运行过程。如有其它的疑问，可以参见项目文件中的`archlab.pdf`。

## 循环求链表和函数
`example.c`文件中第一个函数的功能就是求链表所有元素的和，数据段在内容在`archlab.pdf`文件中指出。参照CSAPP的`4.1.5`节求数组元素的和，很容易实现对应的求链表元素的和的汇编程序。
<!--more-->
``` x86asm
# Execution begins at address 0
    .pos 0
    irmovq stack, %rsp      # Set up stack pointer
    call main               # Execute main program
    halt                    # Terminate program

# Sample linked list
.align 8
ele1:
    .quad 0x00a
    .quad ele2
ele2:
    .quad 0x0b0
    .quad ele3
ele3:
    .quad 0xc00
    .quad 0

main:
    irmovq ele1,%rdi
    call sum_list
    ret

# long sum_list(list_ptr ls)
# ls in %rdi
sum_list:
    xorq   %rax,%rax     # val = 0
    andq   %rdi,%rdi     # Set CC
    jmp    test
loop:
    mrmovq (%rdi),%r10
    addq   %r10,%rax
    mrmovq 8(%rdi),%rdi
    andq   %rdi,%rdi
test:
    jne    loop
    ret

# Stack starts here and grows to lower addresses
    .pos 0x200
stack:
```
>在shell中执行如下命令，验证汇编代码的正确性。我们看到`%rax`寄存器的内容为`cba`，等于链表所有元素的和。
``` bash
root@misc$ ./yas sum.ys
root@misc$ ./yis sum.yo
Stopped in 26 steps at PC = 0x13.  Status 'HLT', CC Z=1 S=0 O=0
Changes to registers:
%rax:   0x0000000000000000  0x0000000000000cba
%rsp:   0x0000000000000000  0x0000000000000200
%r10:   0x0000000000000000  0x0000000000000c00

Changes to memory:
0x01f0: 0x0000000000000000  0x000000000000005b
0x01f8: 0x0000000000000000  0x0000000000000013
```

## 递归求链表元素的和
函数2和函数1实现相同的功能，不过函数2使用递归形式，因此程序的调用和栈与函数1基本保持一致，仅函数2本身的具体实现发生变化。对应的汇编程序如下：
``` x86asm
# Execution begins at address 0
    .pos 0
    irmovq stack, %rsp      # Set up stack pointer
    call main               # Execute main program
    halt                    # Terminate program

# Sample linked list
.align 8
ele1:
    .quad 0x00a
    .quad ele2
ele2:
    .quad 0x0b0
    .quad ele3
ele3:
    .quad 0xc00
    .quad 0

main:
    irmovq ele1,%rdi
    call rsum_list
    ret

# long rsum_list(list_ptr ls)
# ls in %rdi
rsum_list:
    xorq   %rax,%rax   # Set return value to 0
    andq   %rdi,%rdi
    je     return
    pushq  %rbx        #Save callee-saved register
    mrmovq (%rdi),%rbx
    mrmovq 8(%rdi),%rdi
    call   rsum_list
    addq   %rbx,%rax
    popq   %rbx        # Restore callee-saved register
return:
    ret

# Stack starts here and grows to lower addresses
    .pos 0x200
stack:
```
>同样的，我们看到`%rax`寄存器的内容为`cba`，等于链表所有元素的和。
``` bash
root@misc$ ./yas rsum.ys
root@misc$ ./yis rsum.yo
Stopped in 40 steps at PC = 0x13.  Status 'HLT', CC Z=0 S=0 O=0
Changes to registers:
%rax:   0x0000000000000000  0x0000000000000cba
%rsp:   0x0000000000000000  0x0000000000000200

Changes to memory:
0x01c0: 0x0000000000000000  0x0000000000000088
0x01c8: 0x0000000000000000  0x00000000000000b0
0x01d0: 0x0000000000000000  0x0000000000000088
0x01d8: 0x0000000000000000  0x000000000000000a
0x01e0: 0x0000000000000000  0x0000000000000088
0x01f0: 0x0000000000000000  0x000000000000005b
0x01f8: 0x0000000000000000  0x0000000000000013
```

## 数组复制
第3个函数将数组元素从一个数组复制到另一个数组中，并且返回数据元素的检验值。功能比较简单，与链表的对应的操作差异不大，唯一需要注意的就是Y86-64指令集体系没有从寄存器与立即数算术运算的指令，因此首先需要将立即数复制到寄存器中再执行相应的算术运算。对应的汇编代码如下：
```
# Execution begins at address 0
    .pos 0
    irmovq stack, %rsp      # Set up stack pointer
    call main               # Execute main program
    halt                    # Terminate program

.align 8
# Source block
src:
    .quad 0x00a
    .quad 0x0b0
    .quad 0xc00

# Destination block
dest:
    .quad 0x111
    .quad 0x222
    .quad 0x333

main:
    irmovq src,%rdi
    irmovq dest,%rsi
    irmovq $3,%rdx
    call copy_block
    ret

#long copy_block(long *src, long *dest, long len)
# src in %rdi, dest in %rsi, len in %rdx
copy_block:
    irmovq $8,%r8      # Constant 8
    irmovq $1,%r9      # Constant 1
    xorq   %rax,%rax   # Set result = 0
    andq   %rdx,%rdx   # Set CC
    jmp    test
loop:
    mrmovq (%rdi),%r10 # Get val = *src
    addq   %r8,%rdi    # src++
    rmmovq %r10,(%rsi) # *dst = *src
    addq   %r8,%rsi    # dst++
    xorq   %r10,%rax   # result ^= val
    subq   %r9,%rdx    # len--, Set CC
test:
    jne    loop        # Stop when 0
    ret

# Stack starts here and grows to lower addresses
    .pos 0x200
stack:
```
>同样的，我们看到`%rax`寄存器的内容为`cba`，等于数组元素异或的值。
``` bash
root@misc$ ./yis copy.yo
Stopped in 36 steps at PC = 0x13.  Status 'HLT', CC Z=1 S=0 O=0
Changes to registers:
%rax:   0x0000000000000000  0x0000000000000cba
%rsp:   0x0000000000000000  0x0000000000000200
%rsi:   0x0000000000000000  0x0000000000000048
%rdi:   0x0000000000000000  0x0000000000000030
%r8:    0x0000000000000000  0x0000000000000008
%r9:    0x0000000000000000  0x0000000000000001
%r10:   0x0000000000000000  0x0000000000000c00

Changes to memory:
0x0030: 0x0000000000000111  0x000000000000000a
0x0038: 0x0000000000000222  0x00000000000000b0
0x0040: 0x0000000000000333  0x0000000000000c00
0x01f0: 0x0000000000000000  0x000000000000006f
0x01f8: 0x0000000000000000  0x0000000000000013
```
