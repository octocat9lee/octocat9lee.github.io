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

# 拷贝控制示例
