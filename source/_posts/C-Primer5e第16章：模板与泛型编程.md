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
twin<int> win_loss;   //win_loss是一个pair<int, int>
```
一个模板类型别名是一簇类的别名，像使用类模板一样，当我们使用twin时，需要指出希望使用哪种特定类型的twin。

### 类模板的static成员
``` cpp
template <typename T>
class Foo
{
public:
    static std::size_t count() { return ctr; }
private:
    static std::size_t ctr;
};

//static成员数据也定义为模板
template <typename T>
size_t Foo<T>::ctr = 0; //定义并初始化ctr
```
每个Foo的实例都有其自己的static成员实例。对任意给定类型X，都有一个Foo<X>::ctr和一个Foo<X>::count成员。所有Foo<X>类型的对象共享相同的ctr对象和count函数。
类似任何其他成员函数，一个static成员函数只有在使用时才会实例化。

## 模板参数
一个特定文件所需要的所有模板的声明通常一起放置在文件开始位置，出现于任何使用这些模板的代码之前。
假设T是一个模板类型参数，当编译器遇到T::mem这样的代码时，它不会知道mem是一个类型成员还是static数据成员，直至实例化时才会直到。但是为了处理模板，编译器必须知道名字是否表示一个类型。例如，假设T是一个类型参数的名字，当编译器遇到如下形式的语句时：
``` cpp
T::size_type * p; //需要分别是定义名为p的变量还是将名为size_type的static数据成员与名为p的变量相乘
```
默认情况下，C++假定通过作用域运算符访问的名字不是类型。我们必须使用typename关键字显式地告诉编译器该名字是一个类型。
``` cpp
template <typename T>
typename T::value_type top(const T &c)
{
    if(!c.empty())
    {
        return c.back();
    }
    else
    {
        return typename T::value_type();
    }
}
```

### 默认模板实参
在C++11新标准中，我们可以为函数和类模板提供默认实参。
函数模板中使用默认实参，包括模板实参和函数实参：
``` cpp
//compare有一个默认模板实参less<T>和一个默认函数实参F()
template <typename T, typename F = less<T>>
int compare(const T &v1, const T &v2, F f = F())
{
    if(f(v1, v2)) return -1;
    if(f(v2, v1)) return 1;
    return 0;
}
int r1 = compare(1, 2);  //使用默认的less<T>比较函数
Sales_data item1, item2;
int r2 = compare(item1, item2, ComaparseIsbn);
```

在类模板中使用默认模板实参：
``` cpp
template <class T = int> //T默认为int
class Numbers
{
public:
    Numbers(T v = 0): val(v) {}
private:
    T val;
};

Numbers<long double> lots_of_precision;
Numbers<> avarage_precision; //空<>表示希望使用默认类型
```

# 模板实参推断
为了获得元素类型，我们可以使用标准库的类型转换模板。类型转换模板定义在头文件`type_traits`中。
通过组合使用remove_reference、尾置返回以及decltype，我们在函数中返回元素值的拷贝：
``` cpp
//为了使用模板参数的成员，必须使用typename
template <typename It>
auto fcn2(It beg, It end) -> typename remove_reference<decltype(*beg)>::typename
{
    return *beg; //返回序列第一个元素的拷贝
}
```

## 转发
某些函数需要将其一个或多个实参连同类型不变地转发给其他函数。我们需要保持被转发实参的所有性质，包括实参类型是否是const的以及实参是左值还是右值。
当用于一个指向模板参数类型的右值引用参数(T &&)时，forward会保持实参类型的所有细节。
``` cpp
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

# 重载与模板
在定义任何函数之前，记得声明所有重载的函数版本，从而避免因为编译器由于未遇到你希望调用的函数而实例化一个并非你所需的版本。
重载与模板函数匹配规则：
>当有多个重载模板对一个调用提供同样好的匹配时，应选择最特例化的版本
对一个调用，如果一个非模板函数与一个模板函数提供同样好的匹配，则选择非模板的版本

# 可变参数模板
可变参数模板就是一个可以接受可变数目参数的模板函数或模板类。可变数目的参数被称为参数包，包括模板参数包，表示零个或多个模板参数；函数参数包，表示零个或多个函数参数。
```  cpp
//Args是一个模板参数包；rest是一个函数参数包
//Args表示零个或多个模板类型参数
//rest表示零个或多个函数参数
template <typename T, typename... Args>
void foo(const T &t, const Args& ... rest);

int i = 0; double d = 3.14; string s = "how now brwon cow";
foo(i, s, 42, d); //包中有三个参数
foo(d, s);        //包中有一个参数
foo("hi");        //空包

//当我们需要知道包中有多少元素时，可以使用`sizeof...`运算符，返回一个常量表达式，而且不会对实参求值
template <typename ... Args>
void g(Args ... args)
{
    cout << sizeof...(Args) << endl; //类型参数的数目
    cout << sizeof...(args) << endl; //函数参数的数目
}
```

## 编写可变参数函数模板
可变参数函数通常是递归的。第一步调用处理包中的第一个实参，然后用剩余实参调用自身。
在我们的print函数中，每次递归调用将第二个实参打印到第一个实参表示的流中。为了终止递归，我们还需要定义一个非可变参数的print函数，接受一个流和一个对象。
``` cpp
//用来终止递归并打印最后一个元素的函数
//该函数必须定义在可变参数版本的print定义之前声明，否则可变参数版本会无限递归
template <typename T>
ostream &print(ostream &os, const T &t)
{
    return os << t;
}

//包中除了最后一个元素之外的其他元素都会调用这个版本的print
template <typename T, typename... Args>
ostream &print(ostream &os, const T &t, const Args&... rest)  //扩展Args
{
    os << t << ", ";    //打印第一个实参
    return print(os, rest...); //递归调用，打印其他实参，扩展rest
}
```

### 包扩展
扩展一个包就是将它分解为构成的元素，对每个元素应用模式，获得扩展后的列表。我们通过在模式右边放一个省略号(...)来触发扩展操作。
print中的函数参数包扩展仅仅将包扩展为其构成元素，C++语言还允许更复杂的扩展模式。例如，我们可以编写第二个可变参数函数，对其每个实参调用debug_rep，然后调用print打印结果string：
``` cpp
template <typename T> string debug_rep(const T &t)
{
    ostringstream ret;
    ret << t; //使用T的输出运算符打印t的一个表示形式
    return ret.str(); //返回ret绑定的string的一个副本
}

template <typename... Args>
ostream &errorMsg(ostream &os, const Args&... rest)
{
    //print(os, debug_rep(a1), debug_rep(a2), ..., debug_rep(an))
    return print(os, debug_rep(rest)...); //对参数包rest中的每个元素调用debug_rep
}
```
扩展中的模式会独立地应用于包中的每个元素。

### 转发参数包
可变参数函数通常将它们的参数转发给其他函数，例如我们希望将fun的所有实参转发给另一个名为work的函数，假定由它完成函数的实际工作。由于fun的参数是右值引用，因此我们可以传递给它任意类型的实参，由于我们使用std::forward传递实参，因此他们的所有类型信息在调用work时都会得到保持。work调用中的扩展即扩展了模板参数包也扩展了函数参数包：
``` cpp
//fun有零个或多个参数，每个参数都是一个模板参数类型的右值引用
template<typename... Args>
void fun(Args&&... args) //将Args扩展为一个右值引用的列表
{
    //work的实参即扩展Args又扩展args
    work(std::forward<Args>(args)...);
}
```

# 模板特例化
## 定义函数模板特例化
为了处理字符串指针而不是数组，可以为第一个版本的compare定义一个模板特例化版本，一个特例化版本就是模板的一个独立的定义，在其中一个或多个模板参数被指定为特定的类型。
``` cpp
//第一个版本：可以比较任意两个类型
template <typename T> int compare(const T&, const T&);
//第二个版本处理字符串字面常量
template<size_t N, size_t M>
int compare(const char (&)[N], const char (&)[M]);

const char *p1 = "hi", *p2 = "mom";
compare(p1, p2);      //调用第一个版本
compare("hi", "mom"); //调用有两个非类型参数的版本

//compare的特殊版本，处理字符数组的指针
template<> //空尖括号指出我们将为原模板的所有模板参数提供实参
int compare(const char* const &p1, const char* const &p2)
{
    return strcmp(p1, p2);
}
```
特例化的本质是实例化一个模板，而非重载它。因此，特例化不影响函数匹配。
模板及其特例化版本应该声明在同一个头文件中。所有同名模板的声明应该放在前面，然后是这些模板的特例化版本。

## 类模板特例化
我们将为标准库hash模板定义一个特例化版本，可以用它来将Sales_data保存在无序容器中。一个特例化的hash类必须定义：
>一个重载的调用运算符，接受一个容器关键字类型的对象，返回一个size_t
两个类型成员，result_type和argument_type，分别表示调用运算符的返回类型和参数类型
默认构造函数和拷贝赋值运算符

为了让Sales_data的用户能使用hash的特例化版本，我们应该在Sales_data的头文件中定义该特例化版本。
``` cpp
//打开std命名空间，以便特例化std::hash
namespace std {
template <> //我们正在定义一个特例化版本，模板参数为Sales_data
struct hash<Sales_data>
{
    //用来散列一个无序容器的类型必须定义下列类型
    typedef size_t result_type;
    typedef Sales_data argument_type; //默认情况下，此类型需要==
    size_t operator()(const Sales_data &s) const;
    //我们的类使用合成的拷贝控制和默认构造函数
};

size_t hash<Sales_data>::operator()(const Sales_data &s) const
{
    return hash<string>()(s.bookNo) ^
           hash<unsigned>()(s.units_sold) ^
           hash<double>()(s.revenue);
}
} //关闭std命名空间


//由于hash使用Sales_data的私有成员，我们必须将它声明为Sales_data的友元：
template <typename T> class std::hash; //友元声明所需要的
class Sales_data {
    friend class std::hash<Sales_data>; //由于hash<Sales_data>声明在std命名空间中
    //其他定义如前
};

//当将Sales_data作为容器的关键字类型时，编译器会自动使用此特例化的版本：
//使用hash<Sales_data>和Sales_data的operator==
unordered_multiset<Sales_data> SDset;
```
