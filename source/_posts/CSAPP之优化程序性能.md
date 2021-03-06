---
title: CSAPP之优化程序性能
tags:
  - CSAPP
toc: true
date: 2017-11-03 10:31:39
---
# 前言
程序性能优化最重要的原则是在`保证程序正确性`的基础上，使用多种手段，反复尝试分析，有时甚至需要查看底层对应的汇编代码，从而找出性能的瓶颈，进行合理的优化。另外，我们需要注意的是在性能重要的环境中反复执行的代码，进行优化是合适的。为了提升程序的性能，我们做了大量的修改优化，但仍然要保持代码的简洁和可读性。如果我们盲目执行对低级别的优化，往往会降低程序的可读性，使得程序难以扩展和维护。因此，我们需要在`程序实现和维护的简单性与它们的运行速度`之间做出权衡。在性能优化过程中，我们经常遇到`很小的改动但产生性能巨大的差异，有时一些很有希望的技术却被证明无效`的场景，这是因为性能依赖于处理器许多特性的设计，而我们却所知甚少，因此，性能的优化往往是比较困难的，需要我们有足够的经验，采用各种综合手段去分析，挖掘性能的瓶颈。

# 优化方法
## 熟悉编译器优化选项
在大多数情况下，我们都是使用高级的程序语言，需要通过编译器将源码转变成机器识别执行的机器程序。因此，我们需要对使用的编译器的优化选项有大致的了解，其中[gcc optimize options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)编译优化选项在其使用手册文档中有详细说明。因为编译器优化能力的局限性和安全性，导致很多情形下编译器很无法执行理想的优化，因此，需要程序员编写编译器容易优化的代码，帮助编译器显式执行优化。
## 消除循环低效率
在循环过程中，重复计算没有改变的量应该使用局部变量进行保存，减少不必要的重复计算。例如，CSAPP作者举例的`将字符串由小写转成成大小`的示例中，在每次循环中求取保持不变的字符串长度量，导致函数的运行时间为二次方。如果使用局部变量保存长度，函数的时间开销为线性。作者特别提到，编程中更常见的问题是`一个无足轻重的代码隐藏着渐进的低效率`，使得当问题的规模较小时，根本无法发现问题。因此我们作为一名有经验合格的程序员应该竭力避免引入这种`渐进低效率`的问题。
<!--more-->
## 减少过程调用
过程调用会带来开销，并且妨碍程序的优化。
## 消除不必要的内存引用
对于指针变量来说，每次对指针变量的引用都会涉及内存的访问，相对寄存器访问速度来说，内存读写是低效的，因此，如果过多的内存医用会降低程序的效率。我们可以将中间的计算过程保存在局部变量中，利用寄存器访问的高效性提高程序的性能。
## 理解现代处理器
一致性：多条指令可以并行执行，但与多条指令简单顺序执行效果一致
乱序性：指令的执行顺序并不与机器程序的指令顺序一致
## 循环展开
循环展开就是增加每次迭代计算的元素的数量，减少循环迭代的次数。比如说，求数组元素和时，每次迭代计算两个元素的和，因此需要的迭代次数减半。`编译器很容易执行循环展开，只要优化级别设置得足够高，许多编译器都能做到这一点，用优化等级3或更高等级调用GCC，它就会执行情况展开。`
## 提高并行性
对于可以结合和可交换的合并运算来说，比如说整数加法或乘法，我们可以通过将一组合并运算分割成两个或更多的部分，在最后合并结果来提高性能。但是浮点乘法和加法是不可结合的，因此由于四舍五入或溢出，可能会产生不同的结果。所以，程序员应该与用户协商，看看是否有特殊的条件，会导致修改后的算法不能接受。大多数编译器并不会尝试对浮点数代码进行这种变换，因为他们没有办法判断这种变换所带来的风险。
## 重新结合变换
重新结合变换是另一种打破顺序相关提高程序性能的方法。累乘中重新结合的方式的每次迭代内第一个乘法都不需要等待前一次迭代的累积值就可以执行。例如：将累乘的`acc = acc * data[i] * data[i + 1]修改为 acc = acc * (data[i] * data[i + 1])`。重新结合变换能够减少关键路径上操作的数量，更好的利用流水线能力得到更好的性能。

# 限制因素
## 寄存器溢出
如果变形度超过了可用寄存器数量，那么编译器就会将某些临时值放到内存中，通常是在运行时堆栈上分配空间。一旦寄存器溢出，那么维护多个累积变量的优势就会消失。幸运的是，x86-64有足够多的寄存器，大多数循环在出现寄存器溢出之前将达到吞吐量的限制。
## 编写适合用条件传送实现的代码
分支预测只对有规律的模式可行，但是程序中有许多测试是完全不可预测的，依赖于数据的任意特性。对于本质上无法预测的情况，如果编译器能够使用条件传送，而不是使用条件转移，可以极大地提高程序的性能。
## 消除读写相关的内存访问
使用局部变量保存中间计算过程，避免下次的迭代依赖上一次的源。

# 性能分析工具

# 总结
总结程序优化的基本策略如下
- 高级设计，为问题选择适当的算法和数据结构，避免使用那些渐进降低性能的算法和编码技术
- 消除连续的函数调用，在可能时将计算移到循环外
- 消除不必要的内存应用，引入临时变量来保存中间结果
- 展开循环，降低开销，使得进一步优化成为可能
- 提高指令级的并行，可以使用多个变量累积，和重新结合等技术，
- 用功能性的风格重写条件操作，使用条件数据传送，而不是条件转移控制

在优化程序性能时，要警惕为了提高效率重写程序时，避免引入错误。一项有用的技术是在优化函数时，使用检查代码测试函数的每个版本，确保优化过程中没有引入错误。
