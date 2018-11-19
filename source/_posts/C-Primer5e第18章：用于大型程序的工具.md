---
title: C++Primer5e第18章：用于大型程序的工具
tags:
  - C++
toc: true
date: 2018-01-09 15:13:05
---
与小系统相比，大规模编程对程序设计语言的要求更高。大规模应用程序的特殊要求包括：
>在独立开发的子系统之间协同处理错误的能力
使用各种库(可能包含独立开发的库)进行协同开发的能力
对比较复杂的应用概念建模的能力

# 异常处理
异常处理机制允许程序中独立开发的部分能够在运行时就出现的问题进行通信并作出相应的处理。
异常将问题的检测与解决过程分离开来。程序的一部分负责检测问题的出现，解决问题的任务传递给程序的另一部分。
## 抛出异常
当抛出一个异常后，栈展开过程沿着嵌套函数的调用链不断查找，直到找到了与异常匹配的catch子句为止，如果没有找到匹配的catch，则它将终止当前程序。
<strong>栈展开过程中对象被自动销毁</strong>
如果在栈展开过程中退出了某个块，编译器将负责在这个块中创建的对象能被正确的销毁。如果某个局部对象是类类型，则该对象的析构将被自动调用。
如果异常发生在构造函数中，则当前对象可能只是构造了一部分。我们也要确保已构造的成员能被正确的销毁。
<strong>析构函数与异常</strong>
析构函数总是会被执行的，但是函数中负责释放资源的代码可能因为异常而被跳过。因此，如果我们使用类来控制资源的分配，就能确保无论函数正常结束还是遇到异常，资源都能被正确释放。
析构函数不应该抛出异常，如果析构函数的某个操作可能抛出异常，则该操作应该放置在一个try语句块中，并且在析构函数内部处理。
<!--more-->
<strong>异常对象</strong>
抛出一个指向局部对象的指针几乎肯定是一种错误行为。如果指针所指的对象位于某个块中，而该块在catch语句之前就已经退出了，则意味着在执行catch语句之前局部对象已经被销毁了。

## 捕获异常
通常情况下，如果catch接受的异常与某个继承体系相关，则最好将该catch的参数定义成引用类型。
如果多个catch语句的类型之间存在继承关系，则我们应该把继承中最特殊的类放在前面。
在执行了某些校正操作之后，我们可以使用空的throw语句重新抛出。空的throw语句只能出现在catch语句或catch语句调用的函数内。
``` cpp
catch(my_error &eObj) //引用类型
{
    eObj.status = errCodes::serverErr; //修改了异常对象
    throw;    //异常对象的status成员时serverErr
} catch(other_error eObj) //非引用类型
{
    eObj.status = errCodes::badErr;  //只修改了异常对象的局部副本
    throw;    //异常对象的status成员没有改变
}
```
为了一次`捕获所有异常`，我们使用catch(...)省略号作为异常声明。
catch(...)通常与重新抛出语句一起使用
``` cpp
try{
    //操作引发异常
}
catch(...)
{
    //处理异常的某些特殊操作
    throw;
}
```
如果在初始值列表抛出异常，但构造函数体内的try语句块还未生效，所以构造函数体内的catch语句无法处理构造函数初始值列表抛出的异常。
要想处理构造函数初始值列表抛出的异常，我们必须将构造函数写成`函数try语句块`：
``` cpp
template <typename T>
Blob<T>::Blob(std::initializer_list<T> il) try :
    data(std::make_shared<std::vector<T>(il))
{
    /* 空函数体 */
}
catch(const std::bad_alloc &e)
{
    handle_out_of_memory(e);
}
```

## noexcept异常说明
在C++11新标准中，我们可以通过noexcept说明指定某个函数不会抛出异常，其形式是关键字noexcept紧跟函数的参数列表后面。
noexcept说明要么出现在该函数的所有声明语句和定义语句中，要么一次也不要出现。
noexcept有两层含义：
>跟在函数参数列后面它是异常说明符
当作为noexcept异常说明的bool实参出现时，它是一个运算符

我们可以使用noexcept运算符得到如下的异常说明：
``` cpp
void f() noexcept(noexcept(g())); //f和g的异常说明一致
```
如果函数g承诺不会抛出异常，则f也不会抛出异常；如果g允许抛异常，则f也可能抛异常。

## 异常类层次
标准库异常类的继承体系：
<center>
![标准库异常继承体系](https://github.com/octocat9lee/blog-images/raw/master/%E6%A0%87%E5%87%86%E5%BA%93%E5%BC%82%E5%B8%B8%E7%B1%BB%E7%BB%A7%E6%89%BF%E4%BD%93%E7%B3%BB.jpg)

</center>

自定义书店应用程序的异常类，我们面向应用的异常类继承自标准异常类：
``` cpp
class out_of_stock : public std::runtime_error
{
public:
    explicit out_of_stock(const std::string &s):
        std::runtime_error(s) {}
};

class isbn_mismatch : public std::logic_error
{
public:
    explicit isbn_mismatch(const std::string &s):
        std::logic_error(s) {}
    isbn_mismatch(const std::string &s, const std::string &lhs, const std::string &rhs):
        std::logic_error(s), left(lhs), right(rhs) {}
    const std::string left, right;
};
```

# 命名空间
## 定义命名空间
命名空间为防止名字冲突提供了更加可控的机制。命名空间分割了全局命名空间，其中每个命名空间时一个作用域。
定义多个类型不相关的命名空间应该使用单独的文件分别表示每个类型(或关联类型构成的集合)。
在通常情况下，我们不把`#include`放在命名空间内部。如果我们这么做了，隐含的意思是把头文件中所有的名字定义成该命名空间的成员。
模板特例化必须定义在原始模板所属的命名空间中。和其他命名空间名字类似，只要我们在命名空间中声明了特例化，就能在命名空间外部定义它了。
``` cpp
//我们必须将模板特例化声明成std成员
namespace std {
    template <> struct hash<Sales_data>;
}

//在std中添加了模板特例化的声明后，就可以在命名空间std的外部定义它了
template <> struct std::hash<Sales_data>
{
    size_t operator()(const Sales_data &s) const
    {
        return hash<string>()(s.bookNo) ^ hash<unsigned>()(s.units_sold) ^ hash<double>()(s.revenue);
    }
}
```
C++11新标准引入`内联命名空间`，内联命名空间中的名字可以被外层命名空间直接使用。定义内联命名空间的方式是在关键字namespace前添加关键字inline。当应用程序的代码在一次发布和另一次发布之间发生了改变时，常常会用到内联命名空间。
`未命名的命名空间`是指关键字namespace后紧跟花括号的一系列声明语句。未命名的命名空间仅在特定的文件内部有效，其作用范围不会横跨多个不同的文件。
在文件中进行静态声明的做法已经被C++标准取消了，现在的做法是使用未命名的命名空间。

## 使用命名空间成员
命名空间的别名使得我们可以为命名空间的名字设定一个短得多的同义词。
``` cpp
namespace primer = cplusplus_primer;
namespace Qlib = cplusplus_primer::QueryLib;
```
using声明语句一次只引入命名空间的一个成员，例如`using std::vector;`。using声明语句可以出现在全局作用域、局部作用域、命名空间作用域域以及类的作用域中。
using指示以`关键字using开始，后面是关键字namespace以及命名空间的名字`。using指示可以出现在全局作用域、局部作用域和命名空间作用域中，但是不能出现在类的作用域中。
对于using声明来说，我们只是简单地令名字在局部作用域内有效。但using声明是令整个命名空间的所有内容变得有效。
<strong>在头文件中避免使用using指示。</strong>因为using指示一次性注入某个命名空间的所有名字，则全局命名空间污染的问题将重新出现。另外就是using指示引发的二义性错误只有在使用了冲突名字的地方才能被发现。

## 类、命名空间与作用域
对于位于命名空间中的类来说，查找规则：当成员函数石油给你某个名字时，首先在该成员中进行查找，然后在类中查找(包括基类)，接着在外层作用域中查找。

当我们给函数传递一个类类型的对象时，除了在常规的作用域查找外还会查找实参类所属的命名空间。
<strong>查找与std::move和std::forward</strong>
在标准库中move和forward都是模板函数，它们都接受一个右值引用的形参。如我们所知，在函数模板中，右值引用形参可以匹配任何类型。如果我们的应用程序也定义了一个接受单一形参的move函数，则不管该形参是什么类型，应用程序的move函数都将与标准库的版本冲突。另外move和forward执行的是非常特殊的类型操作，所以应用程序修改函数原有行为的概率非常小。
通过使用std::move和std::forward而非move，我们就能明确地知道想要使用的是标准库版本的函数。

## 重载与命名空间
using声明语句声明的是一个名字，而非一个特定的函数。当我们为函数书写using声明时，该函数的所有版本都被引入到当前作用域中。
如果using声明所在的作用域中已经有一个函数与新引入的函数同名而且形参列表相同，则将引发错误。
对于using指示来说，引入一个和已有函数形参列表完成相同的函数并不会产生错误。此时，只要我们指明我们调用的是命名空间中的版本还是当前作用域中的版本。

# 多重继承与虚继承
