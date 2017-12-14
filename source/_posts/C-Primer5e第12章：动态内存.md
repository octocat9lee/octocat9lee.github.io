---
title: C++Primer5e第12章：动态内存
tags:
  - C++
toc: true
date: 2017-12-14 10:00:15
---
动态对象的生存期由程序来控制，当动态对象不再使用时，程序必须显式地销毁它们。
# 动态内存与智能指针
C++11新标准库提供两种`智能指针类型`管理动态对象。智能指针行为类似常规指针，重要区别是智能指针负责自动释放所指向的对象。两种智能指针的区别在于管理底层底层的方式：`shared_ptr`允许多个指针指向同一个对象；`unique_ptr`则独占所指向的对象。标准库还定义了一个名为`weak_ptr`的伴随类，它是一种弱引用，指向`shared_ptr`所管理的对象。三种类型都定义在`memory`头文件中。

## shared_ptr类
智能指针也是模板类，必须在尖括号内提供指针指向的类型参数。
<strong>shared_ptr和unique_ptr都支持的操作</strong>
``` cpp
shared_ptr<T>    空智能指针，可以指向类型为T的对象
unique_ptr<T>
p                将p用作一个条件判断，若p指向一个对象，则为true
*p               解引用p，获得它指向的对象
p->mem           等价于(*p).mem
p.get()          返回p中保存的指针。小心使用，若智能指针释放了其对象，返回的指针所指向的对象也就消失
swap(p, q)       交换p和q中的指针
p.swap(q)
```
<strong>shared_ptr独有的操作</strong>
``` cpp
make_shared<T>(args)   返回一个shared_ptr，指向一个动态分配的类型为T的对象。使用args初始化此对象
shared_ptr<T> p(q)     p是shared_ptr的拷贝；此操作会递增q中的计数器。q中的指针必须能转换为T*
p = q                  p和q都是shared_ptr，所保存的指针必须能相互转换。此操作会递减p的引用计数，
                       递增q的引用。若p的引用计数变为0，则将其管理的原内存释放。表达式含义：将p指向q的对象
p.unique()             若p.use_count()为1，则返回true；否则返回false
p.use_count()          返回与p共享对象的智能指针数量，可能很慢，主要用于调试
```
<!--more-->
### make_shared函数
最安全的分配和使用动态内存的方法是调用一个名为make_sharedde的标准库函数。在分配对象的同时就将shared_ptr与之绑定，从而避免无意中将同一块内存绑定到多个独立创建的shared_ptr上。此函数在动态内存中分配一个对象并初始化它，返回指向此对象的shared_ptr。
``` cpp
//p6指向一个动态分配的空vector<string>
auto p6 = make_shared<vector<string>>(); //p6为shared_ptr<vector<string>>类型
```

### shared_ptr的拷贝和赋值
每个shared_ptr都有一个引用计数值，表示有多少个shared_ptr指向它的对象。
当我们拷贝一个shared_ptr，例如用一个shared_ptr初始化另一个shared_ptr，作为参数传递给函数以及作为函数的返回值时，它所关联的计数器会递增。
当我们给shared_ptr赋予一个新值或是shared_ptr被销毁，例如局部的shared_ptr离开作用域时，计数器会递减。
一旦一个shared_ptr的计数器变为0，它就会自动释放自己所管理的对象，并释放它占用的内存。
当动态对象不再被使用时，shared_ptr类会自动释放动态对象。
如果将shared_ptr保存在一个容器中，记得使用erase删除那些不再需要的元素，从而释放相关内存。

## 直接内存管理
对动态分配的对象进行初始化通常是个好主意。
默认情况下，如果new不能分配所要求的内存空间，它会抛出一个类型为bad_alloc的异常。
使用new和delete管理动态内存存在三个常见问题：
>1 忘记delete内存，导致内存泄露
2 使用已经释放的内存
3 同一块内存重复释放。当两个指针指向相同的动态分配对象时，可能通过一个指针delete了，如果我们随后又delete第二个指针，自由空间就可能被破坏

当我们delete一个指针后，指针指向的内存就无效了，指针变成了`空悬指针(dangling pointer)`。虽然指针指向的内存无效，但指针仍有可能保存着(已经释放了的)动态内存的地址，访问空悬指针将操作未知的错误，并且难以调试。

## shared_ptr和new结合使用
定义和改变shared_ptr的其他方法：
``` cpp
shared_ptr<T> p(q)      p管理内置指针q所指向的对象；q必须指向new分配的内存，且能够转换为T*类型
shared_ptr<T> p(u)      p从unique_ptr u那里接管了对象的所有权；将u置为空
shared_ptr<T> p(q, d)   p接管内置指针q所指向对象的所有权，p将使用可调用对象d来代替delete
shared_ptr<T> p(p2, d)  p是shared_ptr p2的拷贝，区别是p将用可调用对象d来代替delete

p.reset()               若p是唯一指向其对象的shared_ptr，reset会释放对象。
p.reset(q)              若传递了可选的参数内置指针q，会令p指向q，否则将p置为空。
p.reset(q, d)           若还传递了参数d，将会调用d而不是delete来释放q
```

我们需要向不能使用智能指针的代码传递一个内置指针，我们可以使用get返回的指针传递。但需要确保代码不会delete指针的情况下，才可以使用get。特别是，永远不要用get初始化另一个智能指针或者为另一个智能指针赋值。
``` cpp
shared_ptr<int> p(new int(42)); //引用计数为1
int *q = p.get();  //正确：但使用q时要注意，不要让q管理的指针被释放
{//新程序块
    //未定义：两个独立的shared_ptr指向相同的内存
    shared_ptr<int>(q);
}//程序块结束，q被销毁，它指向的内存被释放
int foo = *p; //未定义：p指向的内存已经被释放了
```

## 智能指针和异常
智能指针使用的基本规范：
>不使用相同的内置指针值初始化(或reset)多个智能指针
不delete get()返回的指针
不使用get()初始化或reset另外一个智能指针
如果你使用了get()返回的指针，记住当最后一个对应的智能指针销毁后，你的指针也无效了
如果你使用智能指针管理的资源不是new分配的内存，记住传递给它一个删除器

## unique_ptr
某个时刻只能有一个unique_ptr指向一个给定对象，因此unique_ptr不支持普通的拷贝或赋值操作。
标准库禁用了unique_ptr的拷贝和赋值，将其拷贝构造函数和赋值函数声明为delete的。因此，当调用unique_ptr的拷贝构造函数时，提示如下错误的编译信息：
``` cpp
error: use of deleted function ‘std::unique_ptr<_Tp, _Dp>::unique_ptr(const std::unique_ptr<_Tp, _Dp>&) [with _Tp = int; _Dp = std::default_delete<int>]’
```
当unique_ptr被销毁时，它所指向的对象也被销毁。
<strong>unique_ptr特有的操作：</strong>
``` cpp
unique_ptr<T> u1        空unique_ptr，可以指向类型为T的对象。u1会使用delete来释放它的指针
unique_ptr<T, D> u2     u2会使用一个类型为D的可调用对象来释放它的指针

unique_ptr<T, D> u(d)   空unique_ptr，指向类型为T的对象，用类型为D的对象d代替delete
u = nullptr             释放u指向的对象，将u置为空
u.release()             u放弃对指针的控制器，返回指针，并将u置为空
u.reset()               释放u指向的对象
u.reset(q)              如果提供了内置指针q，令u指向这个对象，否则将u置为空
```
调用release会切断unique_ptr和它原来管理的对象间的联系。release返回的指针通常用来初始化另外一个智能指针或给另外一个智能指针赋值。但是，如果我们不用另外一个智能指针来保存release返回的指针，我们程序就要负责释放资源。
``` cpp
//将所有权从p1(指向string Stegosaurus)转移给p2
unique_ptr<string> p2(p1.release()); //release将p1置为空，p2指向string Stegousaurus

unique_ptr<string> p3(new string("Trex"));
//将所有权从p3转移给p2
p2.reset(p3.release()); //p2指向string Trex，reset释放了p2原来指向的内存，并且p3置为空

p2.release(); //错误：p2不会释放内存，而且我们丢失了指针
auto p = p2.release(); //正确，但我们必须记得delete(p)
```

### 传递unique_ptr参数和返回unique_ptr
我们可以拷贝和赋值一个将要被销毁的unique_ptr。例如可以从函数返回一个unique_ptr或者返回一个局部对象的拷贝：
``` cpp
unique_ptr<int> clone(int p)
{
    return unique_ptr<int>(new int(p));
}

unique_ptr<int> clone(int p)
{
    unique_ptr<int> ret(new int(p));
    //...
    return ret;
}
```

## weak_ptr
weak_ptr表示一种“弱共享对象”的智能指针，将一个weak_ptr绑定到一个shared_ptr不会改变shared_ptr的引用计数。即使有weak_ptr指向对象，对象也还是被释放。
<strong>weak_ptr操作：</strong>
``` cpp
weak_ptr<T> w       空weak_ptr 可以指向类型为T的对象
weak_ptr<T> w(sp)   与shared_ptr sp指向相同对象的weak_ptr。T必须能够转换为sp指向的类型
w = p               p可以是一个shared_ptr或weak_ptr。赋值后w与p共享对象
w.reset()           将w置为空
w.use_count()       与w共享对象的shared_ptr的数量
w.expired()         若w.use_count()为0，返回true，否则返回false
w.lock()            如果expired为true，返回一个空的shared_ptr；否则返回一个指向w的对象的shared_ptr
```

# 动态数组
new动态分配数组得到的是一个数组元素类型的指针，因此不能对动态数组调用begin或end，也不能用范围for语句来处理动态数组中的元素。
在C++11新标准中，我们可以使用元素初始化器的花括号列表初始化动态分配的数组：
``` cpp
int *pia3 = new int[10]{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
delete [] pia3;
```
释放动态数组时，数组中的元素按逆序销毁。
如果我们在delete一个指向数组的指针时忽略了方括号，或者在delete一个指向单一对象的指针时使用了方括号，其行为是未定义的。

为了用unique_ptr管理动态数组，必须在对象类型后面跟一对方括号：
``` cpp
//up指向一个包含10个未初始化int的数组
unique_ptr<int[]> up(new int[10]);
for(size_t i = 0; i != 10; ++i)
{
    up[i] = i; //为每个元素赋予一个新值
}
up.release(); //自动调用delete[]销毁其指针
```
shared_ptr不支持管理动态数组，如果希望使用shared_ptr管理一个动态数组，必须提供自己定义的删除器。
``` cpp
//为了使用shared_ptr，必须提供一个删除器
shared_ptr<int> sp(new int[10], [](int *p) { delete [] p;});

for(size_t i = 0; i != 10; ++i)
{
    *(sp.get() + i) = i; //shared_ptr未定义下标运算符，并且不支持指针的算术运算，使用get获取内置指针
}

sp.reset(); //使用我们提供的lambda释放数组，使用delete[]
```

## allocator类
allocator类提供一种类型感知的内存分配方法，它分配的内存是原始的、未构造的。
