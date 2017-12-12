---
title: C++Primer5e第10章：泛型算法
tags:
  - C++
toc: true
date: 2017-12-12 11:40:20
---
标准库并未给容器添加大量的功能，而是提供了一组独立于任何特定容器的通用的算法实现相关功能。算法是泛型的，也就是说可以用于不同类型的元素和多种容器类型，不仅包括标准库类型，还包括内置数组类型。
# 概述
大多数算法定义在头文件`algorithm`中。标准库还在头文件`numeric`中定义了一组数值泛型算法。算法并不直接操作容器，而是遍历由两个迭代器指定的一个元素范围进行操作。迭代器令算法不依赖于容器，但算法依赖于元素类型的操作，大多数算法提供了一种方法，允许我们使用自定义的操作来代替默认的运算符。<strong>STL组件间关系如下图所示：</strong>
<center>![STL组件关系](http://oyh38rhr2.bkt.clouddn.com/github/171212/JG7AjIb3dh.jpg)</center>
泛型算法使用迭代器作为访问容器元素的桥梁，对元素的元素可以通过函数对象传递到算法内部。为了实现与数据结构的分离，为了实现通用性，算法根本就不该知道容器的存在。算法访问数据的唯一通道是迭代器，是否改变容器大小，完全是迭代器的选择和责任。

# 初识泛型算法
算法使用元素的方式包括只读取元素、改变元素或者是重排元素顺序。算法不会执行容器操作，所以算法不会改变底层容器的大小。
## 只读算法
accumulate的第三个参数的类型决定了函数中使用哪个加法运算符以及返回值的类型。
``` cpp
//对vec中的int元素求和，和的初始值为0
int sum = accumulate(vec.cbegin(), vec.cend(), 0);
//将vector中所有的string元素连接起来
string sum = accumulate(v.cbegin(), v.cend(), std::string(""));
//错误：const char*上没有定义+运算符
string sum = accumulate(v.cbegin(), v.cend(), "");
```
对于只读取元素而不改变元素的算法，最好使用cbegin()和cend()。
使用equal判断两个序列是否保存相同的值，构成这两个序列的元素可以来自不同类型的容器。
``` cpp
vector<string> roster1 = {"Hello", "The", "a"};
list<const char*> roster2 = {"Hello", "The", "a"};
bool eq = equal(roster1.cbegin(), roster1.cend(), roster2.cbegin());
```
在上面equal中，如果roster中都是C风格字符串，用==比较两个char*对象，只是检查两个指针值是否相等。所以只有当两个序列中的指针都指向相同地址时，equal才会返回true。否则，即使字符串内容完全相同，也会返回false。
那些只接受一个单一迭代器来表示第二个序列的算法，都假定第二个序列至少与第一个序列一样长。
<!--more-->
## 写容器元素的算法
<strong>使用写容器算法时，必须注意确保序列有足够元素，而不是有足够的空间，因为算法不会执行容器操作，不可能改变容器的大小</strong>
一种保证算法有足够元素空间来容纳输出数据的方法是使用`插入迭代器(insert iterator)`。当我们通过一个插入迭代器赋值时，一个与赋值号右侧值相等的元素被添加到容器中。
back_inserter接受一个指向容器的引用，返回一个与该容器绑定的插入迭代器。当通过此迭代器赋值时，赋值运算符会调用push_back将给定值的元素添加到容器中
``` cpp
vector<int> vec; //空向量
//正确：back_inserter创建插入迭代器，可以用来向vec添加元素
fill_n(back_inserter(vec), 10, 0);  //添加10个元素到vec

```
replace算法将其中所有等于给定值的元素都修改为另一个值。如果我们希望保留原序列不变，可以调用replace_copy。此算法接受额外第三个迭代器参数，指出调整后序列的保存位置：
``` cpp
//将所有值为0的元素修改为42
replace(ilst.begin(), ilst.end(), 0, 42);
//使用back_inserter按需要增长目标序列
//ivec包含ilst的拷贝，不过ilst中值为0的元素在ivec中都变为42
replace(ilst.cbegin(), ilst.cend(), back_inserter(ivec), 0, 42);
```
标准库算法从来不直接操作容器，它们只操作迭代器，从而间接访问容器。能不能插入和删除元素，不在于算法，而在于传递给它们的迭代器是否具有这样的能力。

## 重排容器元素的算法
删除容器中重复元素，使得vector<string>中每个单词仅出现一次：
``` cpp
void elimDups(vector<string> &words)
{
    //sort按字典排序words，使重复元素相邻
    sort(words.begin(), words.end());
    //unique去除相邻的重复元素，返回指向不重复区域之后一个位置的迭代器
    auto end_unique = unique(words.begin(), words.end());
    //删除重复的单词
    words.erase(end_unique, words.end());
}
```
unique操作并不会删除重复值的元素，所以执行unique后，容器的元素数目并未改变。不重复元素之后位置上的元素的值是未定义的。在某些编译器上，unique操作是将重复元素交换到了容器的末尾。

# 定制操作
标准库为算法定义额外的版本，我们能够自己定义操作来代替默认运算符。
## 向算法传递函数
谓词是一个可调用的表达式，返回结果是一个能用作条件的值。标准库算法使用的谓词包括`一元谓词和二元谓词`。接受谓词参数的算法对输入序列中的元素调用谓词，因此元素的类型必须能够转换为谓词的参数类型。
使用isShorter将单词按照长度排序：
``` cpp
bool isShorter(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
sort(words.begin(), words.end(), isShorter);
```
稳定排序算法维持相等元素的原有顺序。通过调用stable_sort，可以保存等长元素的字典序：
``` cpp
elimDups(words); //将words按字典序重排，并消除重复单词
//按长度重新排序，长度相同的单词维持字典序
stable_sort(words.begin(), words.end(), isShorter);
for(const auto &s : words)
{
    cout << s << " ";
}
```

## lambda表达式
我们可以向算法传递任何类别的`可调用对象`。对于一个对象或一个表达式，如果可以对其使用调用运算符，则称为可调用的。可调用对象包括：
>函数和函数指针
重载了函数调用运算符的类
lambda表达式

lambda表达式可以看作是`一个未命名的内联函数`。lambda可以定义在函数内部。一个lambda表达式的形式：
``` cpp
[capture list] (parameter list) -> return type { function body }
```
capture list(捕获列表)是一个lambda所在函数中定义的局部变量的列表(通常为空)。我们可以忽略参数列表和返回类型，但必须永远包含捕获列表和函数体
``` cpp
auto f = [] { return 42; };
cout << f() << endl;
```
如果函数体只是一个return语句，则返回类型从返回的表达式类型推断而来。否则，返回类型为void。

### 向lambda传递参数
一个与isShorter函数相同功能的lambda：
``` cpp
[](const string &a, const string &b) { return a.size() < b.size(); }
//按长度排序，长度相同的单词维持字典序
stable_sort(words.begin(), words.end(),
            [](const string &a, const string &b)
                { return a.size() < b.size(); });
```

### 使用捕获列表
一个lambda通过将局部变量包含在捕获列表中指出将会使用这些变量。使用捕获列表的lambda：
``` cpp
[sz](const string &a) { return a.size() >= sz; };
void biggies(vector<string> &words, vector<string>::size_type sz)
{
    elimDups(words); //将words按字典序重排，并消除重复单词
    //按长度重新排序，长度相同的单词维持字典序
    stable_sort(words.begin(), words.end(), isShorter);
    //返回指向第一个满足size()>=sz的元素的迭代器
    auto wc = find_if(words.cbegin(), words.cend(),
        [sz](const string &a) { return a.size() >= sz; });
    //计算size>=sz的元素的数目
    auto count = words.end() - wc;
    //for_each对序列中每个元素调用可调用对象
    for_each(wc, words.cend(),
        [](const string &s) { cout << s << " "; });
    cout << endl;
}
```
捕获列表只用于局部非static变量，lambda可以直接使用局部static变量和在它所在函数之外声明的名字。

## lambda捕获和返回
当定义一个lambda时，编译器生成一个与lambda对应的新的(未命名的)类类型。当向一个函数传递一个lambda时，传递的参数就是编译器生成的类类型的未命名对象。
默认情况下，从lambda生成的类都包含一个对应该lambda所捕获的变量的数据成员。类似任何普通类的数据成员，lambda的数据成员也在lambda对象创建时被初始化。
### 值捕获
被捕获的值是在lambda创建时拷贝，而不是调用时拷贝：
``` cpp
void fcn1()
{
    size_t v1 = 42; //局部变量
    //将v1拷贝到名为f的可调用对象
    auto f = [v1] { return v1; }
    v1 = 0;
    auto j = f();   //j为42：f保存了我们创建它是v1的拷贝
}
```

### 引用捕获
lambda捕获的都是局部变量，如果lambda可能在函数结束后执行，捕获的引用指向的局部变量已经消失。如果函数返回一个lambda，此lambda也不能包含引用捕获，与函数不能返回局部变量的引用类似。
``` cpp
void fcn2()
{
    size_t v1 = 42; //局部变量
    //对象f2包含v1的引用
    auto f2 = [&v1] { return v1; }
    v1 = 0;
    auto j = f2();   //j为0：f2保存v1的引用
}

void biggies(vector<string> &words, vector<string>::size_type sz,
             ostream &os = cout, char c = ' ')
{
    //之前一样的重排words代码
    for_each(wc, words.cend(),
        [&os, c](const string &s) { os << s << c; }); //必须使用引用捕获，因为不能拷贝ostream对象
}
```
尽量保持lambda的变量捕获简单化，避免潜在的捕获导致的问题。而且，尽量避免捕获指针或引用。

### 隐式捕获
隐式捕获就是让编译器根据lambda中的代码来推断我们使用哪些变量。为了指示编译器推断捕获列表，应该在捕获列表中写一个`&或=`。&告诉编译器采用引用捕获，=则表示采用值捕获。
如果希望一部分采用值捕获，其他采用引用捕获，可以混合使用隐式捕获和显式捕获，但第一个元素必须是一个&或=。
``` cpp
void biggies(vector<string> &words, vector<string>::size_type sz,
             ostream &os = cout, char c = ' ')
{
    //之前一样的重排words代码
    //os隐式捕获，引用方式；c显式捕获，值捕获方式
    for_each(wc, words.cend(),
        [&, c](const string &s) { os << s << c; }); //必须使用引用捕获，因为不能拷贝ostream对象
    //os显式捕获，引用方式，c隐式捕获，值捕获方式
    for_each(wc, words.cend(),
        [=, &os](const string &s) { os << s << c; }); //必须使用引用捕获，因为不能拷贝ostream对象
}
```

### 可变lambda
默认情况下，对于一个值被拷贝的变量，lambda不会改变其值。如果我们希望能改变一个被捕获变量的值，就必须在参数列表首加上关键字mutable。
``` cpp
void fcn3()
{
    size_t v1 = 42; //局部变量
    //f可以改变它所捕获的变量的值
    auto f = [v1] () mutable {return ++v1;}
    v1 = 0;
    auto j = f(); //j为43
}
```
引用捕获的变量是否可以修改依赖于此引用指向的是一个const类型还是一个非const类型

### 指定lambda返回类型
当我么需要为lambda定义返回类型时，必须使用尾置返回类型。
``` cpp
transform(vi.begin(), vi.end(), vi.begin(),
    [](int i) -> int
    { if(i < 0) {return -i;} else {return i;});
```
