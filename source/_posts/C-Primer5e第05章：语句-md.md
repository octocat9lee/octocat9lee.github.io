---
title: C++Primer5e第05章：语句
tags:
  - C++
toc: true
date: 2017-12-07 11:21:32
---
# 迭代语句
## 范围for语句
范围for语句语法形式：
``` cpp
for(declaration : expression)
    statement
```
expression表示一个序列，共同特点是要有能返回迭代器的begin和end成员。比如说花括号括起来的初始值列表、数组、vector和string等类型。
declaration定义一个变量，要求序列中每个元素能够转换成该变量类型。最简单的办法是使用auto类型说明符。如果需要对序列中的元素执行写操作，循环变量必须声明为引用类型。
<!--more-->
# try语句块和异常处理
在C++中，异常处理包括：
- throw表达式，使用throw表达式来表示遇到了无法处理的问题，也就是说`throw引发(raise)异常`
- try语句块，try语句块以关键字try开始，并以一个或多个catch子句结束。catch子句称为`异常处理代码`
- 一套异常类，用于在throw表达式和catch子句间传递异常的具体信息

## throw表达式
throw表达式包含关键字和紧随其后的一个表达式，其中表达式就是抛出的异常类型。
``` cpp
throw runtime_error("Run Error Tip Message");
```
类似runtime_error是标准库异常类型的一种，定义在stdexcept头文件中。

## try语句块
try语句块的通用语法形式：
``` cpp
try
{
    program-statements
}
catch(exception-declaration)
{
    handler-statements
}
catch(exception-declaration)
{
    handler-statements
}
```
`寻找异常处理代码与函数调用链相反`。当异常抛出时，首先搜索抛出该异常的函数，如果没有找到匹配的catch子句，终止该函数，并在调用函数中寻找。如果还是没有找到匹配的catch子句，新的函数也终止，继续搜索调用它的函数。以此类推，直到找到适当的catch子句为止。如果最终还是没有找到任何匹配的catch子句，程序执行terminate库函数非正常退出。
<strong>编写异常安全的代码非常困难</strong>
异常中断程序的正常流程，从而导致程序处于无效或未完成状态，或者资源没有清理释放等待。如果在异常发生期间正确执行了清理工作的程序称为异常安全代码。对于异常发生时，什么时候直接让程序退出，什么时候处理异常后继续执行，参见下面的讨论[对使用 C++ 异常处理应具有怎样的态度？](https://www.zhihu.com/question/22889420)

## 标准异常
C++标准定义了一组类，用户报告标准库函数遇到的问题，异常类定义在4个头文件中：
- exception头文件定义了最通用的异常类exception。它只报告异常的发生，不提供额外信息
- stdexcept头文件定义几种常用的异常类，比如说exception、runtime_error、range_error
- new头文件定义了bad_alloc异常类型
- type_info头文件定义了bad_cast异常类型

我们只能以默认方式初始化exception、bad_alloc和bad_cast对象，不允许为这些对象提供初始值。其他异常类型可以使用string或者C风格字符串初始化。
异常类型提供名为what的成员函数，返回指向C风格字符串的`const char*`，提供异常的文本信息。如果异常类型有一个字符串初始值，则what返回该字符串，对于其他没有初始值的异常类型来说，what返回的内容由编译器决定。
