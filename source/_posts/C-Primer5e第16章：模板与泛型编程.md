---
title: C++Primer5e第16章：模板与泛型编程
tags:
  - C++
toc: true
date: 2017-12-28 17:08:00
---
模板是C++中泛型编程的基础。一个模板就是一个创建类或函数的蓝图或者说公式。
# 定义模板
## 函数模板
除了定义类型参数，还可以在模板中定义`非类型参数(nontype parameter)`。一个非类型参数表示一个值而非一个类型。我们通过特定的类型名而非关键字class或typename来指定非类型参数。
当一个模板被实例化时，非类型参数被一个用户提供的或编译器推断出的值所代替。这些值必须是常量表达式，从而允许编译器在编译时实例化模板。非类型参数可以是一个整型，或者是一个指向对象或函数类型的指针或引用。
比较不同长度的字符串字面常量，因此为模板定义了两个非类型的参数，分别表示数组的长度：
``` cpp
template<unsigned N, unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M])
{
    return strcmp(p1, p2);
}

compare("hi", "mom");
//编译器会实例化出如下版本
int compare(const char (&p1)[3], const char (&p2)[4])
```
<strong>模板程序应该尽量减少对实参类型的要求</strong>
当编译器遇到一个模板定义时，它并不生成代码。只有当我们实例化出模板的一个特定版本时，编译器才会生成代码。因此大多数编译错误在实例化期间报告。
<strong>模板的头文件通常既包括声明也包括定义</strong>
<!--more-->
## 类模板
一个类模板的每个实例都形成一个独立的类。
定义在类模板之外的成员函数就必须以关键字template开始，后接类模板参数列表：
``` cpp
template <typename T>
ret-type Blob<T>::member-name(param-list) {

}
```
由于模板不是一个类型，我们不能使用typedef定义模板的类型别名。C++11新标准允许我们为类模板定义一个类型别名：
``` cpp
template<typename T> using twin = pair<T, T>;
twin<string> authors; //authors是一个pair<string, string>
twin<int> win_loss; //win_loss是一个pair<int, int>
```
一个模板类型别名是一簇类的别名，像使用类模板一样，当我们使用twin时，需要指出希望使用哪种特定类型的twin。
