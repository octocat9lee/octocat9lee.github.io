---
title: C++Primer5e第13章：拷贝控制
tags:
  - C++
toc: true
date: 2017-12-15 10:09:25
---
拷贝控制决定着对象在拷贝、移动、赋值和销毁时的行为，包括：`拷贝构造函数(copy constructor)、拷贝赋值运算符(copy-assignment operator)、移动构造函数(move constructor)、移动赋值运算符(move-assignment operator)和析构函数(destructor)`。
>拷贝构造和移动构造函数定义当用同类型的另一个对象初始化本对象时做什么。
拷贝赋值运算符和移动赋值运算符定义了将一个对象赋予同类型对象时做什么。
析构函数定义了当此对象销毁时做什么。

# 拷贝、赋值和销毁

## 拷贝构造函数
拷贝构造函数特征：首先满足构造函数特征，没有返回值；其次操作函数的第一个参数必须是自身类型的引用，其他参数有默认值。
<strong>如果第一个参数不是引用类型，为了在调用拷贝构造函数时拷贝实参又需要调用拷贝构造函数，从而形成死循环。</strong>

## 拷贝赋值运算符
赋值运算符其实就是名为`operator=`的函数。赋值运算符通常应该返回一个指向其左侧对象的引用。

## 析构函数
如同构造函数有一个初始化部分和一个函数体，析构函数也有一个函数体和一个析构部分。在构造函数中，成员的初始化是在函数体执行之前完成的，而且按照它们在类中出现的顺序初始化。在析构函数中，首先执行函数体，然后按初始化的相反顺序销毁其成员。成员是在析构函数体之后隐含的析构阶段中被销毁的。
对于一个类，只有唯一的一个析构函数。
<!--more-->
## 三/五法则
当我们判断一个类是否需要自己版本的拷贝控制成员时，一个基本原则就是确定这个类是否需要一个析构函数。
<strong>如果这个类需要自定义析构函数，那么也需要自定义拷贝构造函数和拷贝赋值运算符。</strong>
如果上面HastPtr仅定义自己的析构函数，使用默认合成的拷贝构造函数和拷贝赋值运算符，因为默认的函数仅简单拷贝指针成员，也就意味着多个HasPtr对象可能指向相同的内存：
``` cpp
class HasPtr
{
public:
    HasPtr(const std::string &s = std::string()):
        ps(new std::string(s)), i(0) {}
    ~HasPtr() {delete ps;}
private:
    std::string *ps;
    int i;
};

HasPtr f(HasPtr hp) //HasPtr是传值参数，所以被拷贝
{
    HasPtr ret = hp;
    return ret;   //ret和hp被销毁，析构函数会delete ret和hp中的指针成员，会导致重复delete
}
```

## 使用=default
我们可以将拷贝控制成员定义为=default来显式地要求编译器生成合成的版本。但我们只能对具有合成版本的成员函数使用=default，即默认构造函数或拷贝控制成员。

## 阻止拷贝
有时，拷贝操作没有合理的意义。因此，我们在定义类时必须采用某种机制阻止拷贝和赋值。例如，iostream类阻止了拷贝，以避免多个对象写入或读取相同的IO缓冲。
在C++11新标准下，我们可以将拷贝构造函数和拷贝赋值运算符定义为`删除的函数(deleted function)`来阻止拷贝。
在函数的参数列表后面加上`=delete`来指出我们希望将它定义为删除的：
``` cpp
class NoCopy
{
public:
    NoCopy() = default;  //使用合成的默认构造函数
    NoCopy(const NoCopy&) = delete;  //阻止拷贝
    NoCopy& operator=(const NoCopy&) = delete; //阻止赋值
    ~NoCopy() = default; //使用合成的析构函数    
};
```
=delete必须出现在函数第一次声明的时候。另外，我们可以对任何函数使用=delete。
<strong>析构函数不能是删除的成员</strong>，如果析构函数被删除，就无法销毁此类型的对象了。
如果一个类有数据成员不能默认构造，拷贝，复制和销毁，则对应的成员函数将被定义为删除的。
通过声明(但不定义)private的拷贝构造函数和拷贝赋值运算符，我们可以阻止任何拷贝该类型对象的企图：试图拷贝对象的用户代码在编译阶段标记为错误，成员函数或友元函数中的拷贝操作将会导致链接时错误。
在C++11新标准中，希望阻止拷贝的类应该使用=delete来定义它们的拷贝构造函数和赋值运算符，而不应该将它们声明为private的。

# 拷贝控制和资源管理
类的行为像一个值，意味着它有自己的状态。当拷贝一个像值的对象时，副本和原对象是相互独立的。例如标准库容器和string类的行为像一个值。
行为像指针的类则共享状态，当我们拷贝一个这种对象时，副本和原对象使用相同的底层数据。改变副本也会改变原对象，反之亦然。例如，shared_ptr类提供类似指针的行为。

## 行为像值的类
赋值运算符通常组合了析构函数和构造函数的作用。类似析构函数，赋值操作会销毁左侧运算对象的资源。类似拷贝构造函数，赋值操作会从右侧运算对象拷贝数据。需要注意`异常安全和自我赋值`特殊情形。
当编写一个赋值运算符时，`一个好的模式是先将右侧运算对象拷贝到一个局部临时对象中`。当拷贝完成后，销毁左侧元素对象的现有成员就是安全的了。一旦左侧运算对象的资源被销毁，就只剩下就数据从临时对象拷贝到左侧运算对象的成员中了。
``` cpp
class HasPtr
{
public:
    HasPtr(const std::string &s = std::string()):
        ps(new std::string(s)), i(0) {}

    HasPtr(const HasPtr &rhs); //copy constructor

    HasPtr& operator=(const HasPtr &rhs); //copy-assignment operator

    ~HasPtr(); //destructor

private:
    std::string *ps;
    int i;
};

HasPtr::HasPtr(const HasPtr &rhs)
{
    ps = new string(*rhs.ps); //拷贝ps指向的对象，而不是拷贝指针本身
    i = rhs.i;
}

HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
    //异常安全并且同时处理了自我赋值
    auto newps = new string(*rhs.ps); //拷贝指针指向的对象，等价于构造函数工作
    delete ps;                        //销毁原string，等价于析构函数功能
    ps = newps;                       //指向新string
    i = rhs.i;                        //使用内置的int赋值
    return *this;                     //返回此对象的引用
}

HasPtr::~HasPtr()
{
    delete ps;
}
```

## 行为像指针的类
对于行为类似指针的类，我们需要为其定义拷贝构造函数和拷贝赋值运算符，来拷贝指针成员本身而不是它指向的string。我们的类仍然需要自己的析构函数来释放接受string参数的构造函数分配的内存。但是，析构函数不能简单释放关联的string，只有当最后一个指向string的HasPtr销毁时，它才可以释放string。
令一个类展现类似指针的行为的最好方法是使用shared_ptr来管理类中的资源。拷贝或赋值一个shared_ptr会拷贝或赋值shared_ptr所指向的指针。shared_ptr类自己记录有多少用户共享它所指向的对象。当没有用户使用对象时，shared_ptr类负责释放资源。
<strong>设计自己的引用计数</strong>
>每个构造函数（拷贝构造函数除外）创建一个引用计数，记录有多少对象与正在创建的对象共享状态。当新创建对象时，只有一个对象共享状态，因此计数器初始化为1
拷贝构造函数不分配新的计数器，而是拷贝给定对象的数据成员，包括计数器。递增共享的计数器，指出对象又被一个新用户共享
析构函数递减计数器，如果计数器为0，则析构函数释放状态
拷贝赋值运算符递增右侧运算对象的计数器，递减左侧运算对象的计数器。如果左侧运算对象计数器变为0，拷贝赋值运算符就必须销毁状态。

计数器不能直接作为HasPtr的数据成员，否则变化时，无法更新所有共享对象的计数器。解决此问题的一种方法是`将计数器保存在动态内存中`。当创建一个对象时，我们分配一个新的计数器。当拷贝或赋值对象时，我们拷贝指向计数器的指针。使用这种方法，副本和原对象都会指向相同的计数器。

### 使用引用计数的类
``` cpp
class HasPtr
{
public:
    //构造函数分配新的string和新的计数器，将计数器置为1
    HasPtr(const std::string &s = std::string()):
        ps(new std::string(s)), i(0), use(new std::size_t(1)) {}

   //拷贝构造函数拷贝所有成员，并递增计数器
    HasPtr(const HasPtr &p):
        ps(p.ps), i(p.i), use(p.use) {++*use;}

    HasPtr& operator=(const HasPtr &rhs); //copy-assignment operator

    ~HasPtr(); //destructor

private:
    std::string *ps;
    int i;
    std::size_t *use; //引用计数器，用来记录有多少个对象共享*ps
};

HasPtr::~HasPtr()
{
    if(--*use == 0)
    {
        delete ps;
        delete use;
    }
}

//通过先递增rhs中计数然后再递减左侧运算对象中的计数来处理自我赋值
HasPtr& HasPtr::operator=(const HasPtr &rhs)
{
    ++*rhs.use; //递增右侧运算对象的引用计数
    if(--*use == 0) //然后递减本对象的引用计数
    {
        delete ps;  //如果没有其他用户，释放本对象分配的成员
        delete use;
    }
    ps = rhs.ps;  //将数据从rhs拷贝到本对象
    i = rhs.i;
    use = rhs.use;
    return *this;  //返回本对象
}
```

# 交换操作
对于分配了资源的类，定义swap可能是一种很重要的优化手段。如下，我们编写自己的swap函数：
``` cpp
class HasPtr
{
    friend void swap(HasPtr&, HasPtr&);
    //其他成员定义与值类型版本一致
}

inline void swap(HasPtr &lhs, HasPtr &rhs)
{
    using std::swap;
    swap(lhs.ps, rhs.ps); //交换指针，而不是string数据
    swap(lhs.i, rhs.i);   //交换int成员
}
```
<strong>swap函数应该调用swap，而不是std::swap</strong>。因为如果一个类有自己特定的swap函数，调用std::swap就是错误的。每个swap调用应该都是未加限定的。如果存在特定类型的swap版本，其匹配程度会优先于std中定义的版本。

## 赋值运算符中使用swap
定义了swap的类通常用swap来定义它们的赋值运算符。使用一种称为`拷贝并交换(copy and swap)`的技术，将左侧运算对象与右侧运算对象的一个`副本`进行交换：
``` cpp
//注意rhs是按值传递，意味调用HasPtr的拷贝构造函数将右侧运算对象中的string拷贝对rhs
//同样遵循先拷贝后交换的原则处理自我赋值，另外，由于内存分配发生在交换之前的拷贝阶段所以也是异常安全的
HasPtr& HasPtr::operator=(HasPtr rhs)
{
    //交换左侧运算对象和局部变量rhs的内容
    swap(*this, rhs); //rhs现在指向左侧对象曾经使用的内存，左侧对象指向赋值运算符右侧的副本
    return *this;     //rhs销毁，从而delete了rhs中指针，也就是左侧对象中原来的内存
}
```

# 对象移动
C++11新标准一个最主要的特性是可以移动而非拷贝对象的能力。在某些情况下，对象拷贝后就立即被销毁了，在这些情况下，移动而非拷贝对象将大幅度提升性能。
在C++11新标准中，容器可以保存不可拷贝的类型，只要它们能被移动即可。
标准库容器、string和shared_ptr类既支持拷贝也支持移动。IO类和unique_ptr类可以移动但不可以拷贝。

## 右值引用
所谓右值引用就是必须绑定到右值的引用。使用`&&`来获得右值引用。
>左值：能对表达式取地址、或具名对象。一般指表达式结束后依然存在的持久对象
右值：不能对表示取地址或匿名对象。一般指表达式结束就不再存在的临时对象

右值要么是字面常量，要么是在表达式求值过程中创建的临时对象。由于右值引用只能绑定到临时对象，因此：
>所引用的对象将要被销毁
该对象没有其他用户
使用右值引用的代码可以自由地接管所引用的对象的资源

变量是左值，因为变量是持久的，直至离开作用域时才被销毁。因此我们不能将一个右值引用直接绑定到一个变量上，即使这个变量是右值引用类型也不行。

C++11新标准中使用`move`标准库函数获得绑定到左值上的右值引用，函数定义在`utility`头文件中。
``` cpp
int &&rr1 = 42;    //正确：字面常量是右值
int &&rr2 = rr1;   //错误：表达式rr1是左值！
int &&rr3 = std::move(rr1); //ok
```
move告诉编译器：我们有一个左值，但我们希望像一个右值使用。
调用move后，`除了对rr1赋值或销毁它，我们不能再使用它`。
使用move的代码应该使用std::move而不是move，从而避免可能存在的名字冲突。
返回左值的表达式包括返回左值引用的函数及赋值、下标、解引用和前置递增/递减运算符
返回右值的包括返回非引用类型的函数及算术、关系、位和后置递增/递减运算符

## 移动构造函数和移动赋值运算符
移动构造函数和移动赋值运算符它们从给定对象“窃取”资源而不是拷贝资源。
移动构造函数的第一个参数是一个`右值引用`。一旦完成资源移动，源对象必须不再指向被移动的资源，因为资源的所有权已经归属新创建的对象。另外，移动构造函数还必须确保移动对象可以被销毁，执行析构函数是安全的。
``` cpp
StrVec::StrVec(StrVec &&s) noexcept: //移动构造函数不应该抛出异常
elements(s.elements), first_free(s.first_free), cap(s.cap)
{
    //令s可以被安全析构，如果不设置，则销毁源对象就会释放我们刚刚移动的内存
    s.elements = s.first_free = s.cap = nullptr;
}

StrVec& StrVec::operator=(StrVec &&rhs) noexcept
{
     //检测自我赋值
    if(this != &rhs)
    {
        free(); //释放已有元素
        elements = rhs.elements; //从rhs接管资源
        first_free = rhs.first_free;
        cap = rhs.cap;
        //将rhs置为可析构状态
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}
```
移动构造函数通常不会分配任何资源，因此移动操作通常不会抛出异常。不抛出异常的移动构造函数和移动赋值运算符必须标记为`noexcept`。在类头文件的声明和定义中都指定`noexcept`。

从一个对象移动数据并不会销毁此对象，但有时在移动操作完成后，源对象会被销毁。因此，当我们编写一个移动操作时，必须确保移后源对象进入一个可析构的状态。

如果一个类定义了自己的拷贝构造函数、拷贝赋值运算符和析构函数，编译器就不会为它合成移动构造函数和移动赋值运算符了。
只有当一个类没有定义任何自己的拷贝控制成员，且它的所有数据成员都能移动构造或移动赋值时，编译器才会为它合成移动构造函数或移动赋值运算符。
如果类定义了移动构造函数或移动赋值运算符，则该类的合成拷贝构造函数和拷贝赋值运算符则被定义为删除的。

<strong>所有的五个拷贝控制成员应该看作一个整体：如果类定义了任何一个拷贝操作，它就应该定义所有五个操作。</strong>

## 移动迭代器
C++11新标准库定义了一种`移动迭代器(move iterator)`适配器。移动迭代器通过改变迭代器的解引用运算符的行为来适配此迭代器。移动迭代器的解引用运算符生成一个右值引用。
标准库的`make_move_iterator`函数将一个普通迭代器转换为一个移动迭代器。
由于移动迭代器支持正常的迭代器操作，我们可以将移动迭代器传递给算法。
``` cpp
void StrVec::reallocate(size_t newcapacity)
{
    auto first = alloc.allocate(newcapacity);
    auto last = uninitialized_copy(make_move_iterator(begin()),
        make_move_iterator(end()), first);

    free(); //一旦完成元素的移动就释放旧的内存空间
    elements = first;
    first_free = last;
    cap = elements + newcapacity;
}
```
由于我们传递给它的是移动迭代器，因此解引用生成的是一个右值引用，这意味着construct将使用移动构造函数来构造元素。
在移动构造函数和移动赋值运算符这些类实现代码之外的地方，只有当你确信需要进行移动操作是安全的，才可以使用std::move。

## 右值引用和成员函数
定义了push_back的标准库容器提供两个版本：一个版本有一个const的左值引用，一个版本指向非const的右值引用。
``` cpp
//拷贝：绑定到任意类型的x，拷贝操作不应该改变源对象，因此通常不需要非const的左值引用版本
void push_back(const X&);
//移动：只能绑定到类型X的可修改的右值，因为移动操作从实参窃取数据，实参不能是const的
void push_back(X &&);
```
StrVec类push_back的令一个版本：
``` cpp
void StrVec::push_back(const std::string& s)
{
    chk_n_alloc(); //确保有空间容纳元素
    alloc.construct(first_free++, s); //在first_free指向的元素构造s的副本
}

void StrVec::push_back(std::string &&s)
{
    chk_n_alloc(); //确保有空间容纳元素
    alloc.construct(first_free++, std::move(s)); //move返回一个右值引用，因此使用string的移动构造函数构造新元素
}

StrVec vec;
string s = "sone string or another";
vec.push_back(s); //调用push_back(const std::string& s)版本
vec.push_back("done"); //调用push_back(std::string &&s)版本，因为实参是一个临时对象是一个右值
```

### 右值和左值引用成员函数
无论对象是左值还是右值，我们都可以在一个对象上调用成员函数。更差异的是，我们可以对一个右值进行赋值：
``` cpp
string s1 = "a value";
string s2 = "another";
s1 + s2 = "wow!";
```
如果我们希望在自己的类中阻止这种用法，我们可以使用`引用限定符(reference qualifier)&或者&&`分别指出this可以指向一个左值或右值。限定符必须同时出现在声明和定义中。
``` cpp
class Foo()
{
public:
    Foo &operator=(const Foo&) &; //只能向左值赋值
    Foo someMem() & const; //错误：const限定符必须在前
    Foo someMem() const &; //正确：const限定符在前
};

Foo& Foo::operator=(const Foo &rhs)
{
    return *this;
}
```

### 重载和引用函数
如果一个成员函数有引用限定符，则具有相同参数列表的所有版本都必须有引用限定符。
成员函数可以根据是否有const来区分其重载版本，也可以使用引用限定符区分重载版本：
``` cpp
class Foo()
{
public:
    Foo sorted() &&; //可用于可改变的右值
    Foo sorted() const &; //可用于const右值或左值
private:
    std::vector<int> data;
};

//本对象为右值，因此可以原址排序
Foo Foo::sorted() &&
{
    sort(data.begin(), data.end());
    return *this;
}

//本对象为const或者左值，不能进行原址排序
Fool Foo::sorted() const &
{
    Foo ret(*this); //拷贝一个副本
    sort(ret.data.begin(), ret.data.end()); //排序副本
    return ret; //返回副本
};

Fool Foo::sorted() const &
{
    Foo ret(*this); //拷贝一个副本，是一个左值
    return ret.sorted(); //仍然调用左值引用版本，产生递归循环
};

Fool Foo::sorted() const &
{
    return Foo(*this).sorted(); //编译器认为Foo(*this)是一个临时对象，因此为右值，从而匹配到右值引用版本
};
```

# 附录
本章代码中完整的StrVec类，具体头文件如下：
``` cpp
//StrVec.h
#ifndef _STR_VEC_H
#define _STR_VEC_H

#include <string>
#include <memory>

class StrVec
{
public:
    StrVec(): //默认构造函数
        elements(nullptr), first_free(nullptr), cap(nullptr) { }

    StrVec(std::initializer_list<std::string> il); //构造函数

    StrVec(const StrVec &s); //拷贝构造函数

    StrVec(StrVec &&s) noexcept; //移动构造函数

    StrVec& operator=(const StrVec &rhs); //拷贝赋值运算符

    StrVec& operator=(StrVec &&rhs) noexcept; //移动赋值运算符

    ~StrVec(); //析构函数

    void push_back(const std::string& s); //拷贝元素

    void push_back(std::string &&s); //移动元素

    size_t size() const { return first_free - elements; }

    size_t capacity() const { return cap - elements; }

    std::string* begin() const { return elements; }

    std::string* end() const { return first_free; }

    void reserve(size_t n) { if(n > capacity()) { reallocate(n); } }

    void resize(size_t n);

private:
    static std::allocator<std::string> alloc;

    void chk_n_alloc()
    {
        if(size() == capacity()) //当没有空间添加新元素
        {
            //分配当前大小两倍的内存空间
            auto newcapacity = size() ? 2 * size() : 1;
            reallocate(newcapacity); //重新分配内存
        }
    }

    //拷贝一个给定范围中的元素
    std::pair<std::string*, std::string*> alloc_n_copy
        (const std::string*, const std::string*);

    void free();  //销毁元素并释放内存

    void reallocate(size_t newcapacity); //分配更多内存并拷贝已有元素

private:
    std::string *elements;   //指向数组首元素的指针
    std::string *first_free; //指向数组最后元素之后的指针
    std::string *cap;        //指向数组空间尾后位置的指针
};

#endif
```
StrVec类的源码文件如下：
``` cpp
//StrVec.cpp
#include "StrVec.h"
#include <algorithm>

using namespace std;

std::allocator<std::string> StrVec::alloc;

StrVec::StrVec(const StrVec &s)
{
    auto data = alloc_n_copy(s.begin(), s.end());
    elements = data.first;
    first_free = data.second;
    cap = data.second;
}

inline StrVec::StrVec(std::initializer_list<std::string> il)
{
    auto newdata = alloc_n_copy(il.begin(), il.end());
    elements = newdata.first;
    first_free = newdata.second;
    cap = newdata.second;
}

StrVec::StrVec(StrVec &&s) noexcept: //移动构造函数不应该抛出异常
elements(s.elements),
first_free(s.first_free),
cap(s.cap)
{
    //令s可以被安全析构，如果不设置，则销毁源对象就会释放我们刚刚移动的内存
    s.elements = s.first_free = s.cap = nullptr;
}

StrVec& StrVec::operator=(StrVec &&rhs) noexcept
{
    //检测自我赋值
    if(this != &rhs)
    {
        free(); //释放已有元素
        elements = rhs.elements; //从rhs接管资源
        first_free = rhs.first_free;
        cap = rhs.cap;
        //将rhs置为可析构状态
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}

StrVec::~StrVec()
{
    free();
}

void StrVec::push_back(const std::string& s)
{
    chk_n_alloc(); //确保有空间容纳元素
    alloc.construct(first_free++, s); //在first_free指向的元素构造s的副本
}

void StrVec::push_back(std::string &&s)
{
    chk_n_alloc(); //确保有空间容纳元素
    alloc.construct(first_free++, std::move(s)); //move返回一个右值引用，因此使用string的移动构造函数构造新元素
}

void StrVec::resize(size_t n)
{
    if(n > size())
    {
        while(size() < n)
        {
            push_back("");
        }
    }
    else if(n < size())
    {
        while(size() > n)
        {
            alloc.destroy(--first_free);
        }
    }
}

std::pair<std::string*, std::string*>
StrVec::alloc_n_copy(const std::string* b, const std::string* e)
{
    //指向同一个数组的指针相减获得间隔的元素数目
    auto data = alloc.allocate(e - b);
    return{data, uninitialized_copy(b, e, data)};
}

//void StrVec::free()
//{
//    if(elements)
//    {
//        for(auto p = first_free; p != elements; )
//        {
//            alloc.destroy(--p);
//        }
//        alloc.deallocate(elements, cap - elements);
//    }
//}

void StrVec::free()
{
    for_each(elements, first_free, [](string &p) { alloc.destroy(&p); });
    alloc.deallocate(elements, cap - elements);
}

//void StrVec::reallocate(size_t newcapacity)
//{
//    //分配新内存
//    auto newdata = alloc.allocate(newcapacity);
//    //将数据从旧内存移动到新内存
//    auto dest = newdata;  //指向新数组中下一个空闲位置
//    auto elem = elements; //指向旧数组下一个元素
//    for(size_t i = 0; i != size(); ++i)
//    {
//        alloc.construct(dest++, std::move(*elem++));
//    }
//    free(); //一旦完成元素的移动就释放旧的内存空间
//    elements = newdata;
//    first_free = dest;
//    cap = elements + newcapacity;
//}

void StrVec::reallocate(size_t newcapacity)
{
    auto first = alloc.allocate(newcapacity);
    auto last = uninitialized_copy(make_move_iterator(begin()),
        make_move_iterator(end()), first);

    free(); //一旦完成元素的移动就释放旧的内存空间
    elements = first;
    first_free = last;
    cap = elements + newcapacity;
}


StrVec& StrVec::operator=(const StrVec &rhs)
{
    auto data = alloc_n_copy(rhs.begin(), rhs.end());
    free();
    elements = data.first;
    first_free = data.second;
    cap = data.second;
    return *this;
}

```
