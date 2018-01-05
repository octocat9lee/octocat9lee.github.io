---
title: C++Primer5e第19章：特殊工具与技术（一）
tags:
  - C++
toc: true
date: 2018-01-05 09:25:57
---
# 内存管理
内存管理的层级以及对应的特性如下：
<center>
![C++内存管理层级](http://oyh38rhr2.bkt.clouddn.com/C++%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%B1%82%E7%BA%A7.jpg)
</center>
我们在程序中经常使用的new和delete是C++标准中的表达式，有时也称为new运算符。但new与delete运算符和其他运算符最大的不同在于new和delete运算符不可以重载。
各层次内存分配与释放方式的使用示例：
``` cpp
void *p1 = malloc(512); //512 bytes
free(p1);

complex<int> *p2 = new complex<int>; //one object
delete p2;

void *p3 = ::operator new(512); //512 bytes
::operator delete(p3);

//以下函数都是non-static，要通过object调用。以下分配7个ints
void *p4 = allocator<int>().allocate(7);
allocator<int>().deallocate((int*)p4,7);
```
<!--more-->
我们虽然不能直接重载new和delete表达式，但由于当我们使用一条new表达式时，实际上执行了三个步骤：
>new表达式调用名为`operator new或operator new[]`的标准库函数，分配一块足够大、原始的、未命名的内存空间
编译器运行相应的构造函数构造对象
返回一个指向该对象的指针

执行一条delete表达式时，实际上执行了如下两个步骤：
>执行指针所指向对象的析构函数
编译器调用`operator delete或operator delete[]`的标准库函数释放内存空间

如果应用程序希望控制内存的分配过程，则需要自己定义operator new函数和operator delete函数。如果我们定义了自己的版本，则编译器将使用我们自定义版本替换标准库定义的版本。

new以及delete对应的具体细节：
``` cpp
try
{
    void *mem = ::operator new(sizeof(complex<int>));   //分配空间
    complex<int> *pc = static_cast<complex<int>*>(mem); //转型
    //pc->complex::complex(1, 2); 编译器才可以这样直接调用构造函数 调用构造函数
    new (pc) complex<int>(1, 2); //application可以使用placenment new调用构造函数
    cout << "real: " << pc->real() << " imag: " << pc->imag() << endl;
    pc->~complex();        //先执行析构
    ::operator delete(pc); //然后释放内存
}
catch(bad_alloc e)
{

}
```
placement new允许我们将对象构建于已分配的内存中，没有所谓的placement delete，因为placement new根本没有分配内存。

## 类作域中重载operator new和operator delete
当我们将重载的operator new和operator delete定义为类的成员时，它们是隐式静态的。我们无须显式地声明static，但显式声明也不会引发错误。因为operator new用在对象构造之前，而operator delete用在对象销毁之后，所以这两个成员必须是静态的，而且它们不能操纵类的任何数据成员。
对于operator new或者operator new[]函数来说，它的返回类型必须是`void*`，第一个参数的类型必须是size_t且该形参不能含有默认实参。
对于operator delete或operator delete[]函数，它们的返回类型必须是void，第一个形参类型必须是`void*`。
与析构函数类似，operator delete也不允许抛出异常，必须使用noexcept异常说明符指定其不抛出异常。
当将operator delete定义成类的成员时，该函数还可以包含一个额外的形参，形参的初始值是第一个形参所指对象的字节数。
自定义的operator new和operator delete函数的目的在于改变内存的分配方式，但不管怎么样，我们都不能改变new运算符和delete运算符的基本含义。
``` cpp
#include <iostream>

using namespace std;

class Foo
{
public:
    Foo()
    {
        //cout << "Foo::Foo()" << endl;
    }
    Foo(int)
    {
        //cout << "Foo::Foo(int)" << endl;
    }

    //(1) 這個就是一般的 operator new() 的重載
    void* operator new(size_t size) {
        cout << "operator new(size_t size), size= " << size << endl;
        return malloc(size);
    }

    //(2) 這個就是標準庫已經提供的 placement new() 的重載 (形式)
    //    (所以我也模擬 standard placement new 的動作, just return ptr)
    void* operator new(size_t size, void *start) {
        cout << "operator new(size_t size, void* start), size= " << size << "  start= " << start << endl;
        return start;
    }

    //(3) 這個才是嶄新的 placement new
    void* operator new(size_t size, long extra) {
        cout << "operator new(size_t size, long extra)  " << size << ' ' << extra << endl;
        return malloc(size+extra);
    }

    //(4) 這又是一個 placement new
    void* operator new(size_t size, long extra, char init) {
        cout << "operator new(size_t size, long extra, char init)  " << size << ' ' << extra << ' ' << init << endl;
        return malloc(size+extra);
    }

    //(1) 這個就是一般的 operator delete() 的重載
    void operator delete(void *ptr, size_t) noexcept {
        cout << "operator delete(void*,size_t)  " << endl;
        free(ptr);
    }

    //(2) 這是對應上述的 (2)
    void operator delete(void*, void*) noexcept {
        cout << "operator delete(void*,void*)  " << endl;
    }

    //(3) 這是對應上述的 (3)
    void operator delete(void *ptr, long) noexcept {
        cout << "operator delete(void*,long)  " << endl;
        free(ptr);
    }

    //(4) 這是對應上述的 (4)
    //如果沒有一一對應, 也不會有任何編譯報錯
    void operator delete(void *ptr, long, char) noexcept {
        cout << "operator delete(void*,long,char)  " << endl;
        free(ptr);
    }

private:
    int m_i;
};

int main()
{
    Foo* p1 = new Foo;           //operator new(size_t size)
    delete p1;                   //operator delete(void*,size_t)

    Foo start;  //Foo::Foo
    Foo* p2 = new (&start) Foo;  //operator new(size_t size, void* start)
    //delete p2; p2 is placement new, so don't release memory
    (void)p2;

    Foo* p3 = new (100) Foo;     //operator new(size_t size, long extra)
    delete p3;                   //operator delete(void*,size_t)

    Foo* p4 = new (100,'a') Foo; //operator new(size_t size, long extra, char init)
    delete p4;                   //operator delete(void*,size_t)

    Foo* p5 = new (100) Foo(1);  //operator new(size_t size, long extra)
    delete p5;                   //operator delete(void*,size_t)

    Foo* p6 = new (100,'a') Foo(1); //operator new(size_t size, long extra, char init)
    delete p6;                      //operator delete(void*,size_t)

    Foo* p7 = new (&start) Foo(1);  //operator new(size_t size, void* start)
    (void)p7;

    Foo* p8 = new Foo(1);           //operator new(size_t size)
    delete p8;                      //operator delete(void*,size_t)

    return 0;
}
```

## 全局作域中重载operator new和operator delete
我们也可以在全局作用域中重载operator new和opeator delete函数，但在全局作用中重载将导致C++标准库，new、delete表达式都将使用全局重载的operator new和operator delete函数，其影响无远弗届。
``` cpp
#include <cstdlib>
#include <string>
#include <iostream>
#include <memory>
#include <list>

using namespace std;

//除非 user 直接使用 malloc/free, 否則都避不開它們, 這樣就可以累積總量.
static long long countNew = 0;
static long long countDel = 0;
static long long countArrayNew = 0;
static long long countArrayDel = 0;

void* myAlloc(size_t size)
{  return malloc(size); }

void myFree(void* ptr)
{  return free(ptr); }

//小心，這影響無遠弗屆
//它們不可被聲明於一個 namespace 內
inline void* operator new(size_t size)
{
    cout << "jjhou global new(), \t" << size << endl;
    countNew += size;
    return myAlloc( size );
}

inline void* operator new[](size_t size)
{
    cout << "jjhou global new[](), \t" << size << endl;
    countArrayNew += size;
    void* p = myAlloc( size );
    return p;
}

//天啊, 以下(1)(2)可以並存並由(2)抓住流程 (但它對我這兒的測試無用).
//當只存在 (1) 時, 抓不住流程.
//在 class members 中二者只能擇一 (任一均可)
//(1)
inline void  operator delete(void* ptr, size_t size)
{
    cout << "jjhou global delete(ptr,size), \t" << ptr << "\t" << size << endl;
    countDel += size;
    myFree(ptr);
}
//(2)
inline void  operator delete(void* ptr)
{
    cout << "jjhou global delete(ptr), \t" << ptr << endl;
    myFree(ptr);
}

//(1)
inline void  operator delete[](void* ptr, size_t size)
{
    cout << "jjhou global delete[](ptr,size), \t" << ptr << "\t" << size << endl;
    countArrayDel += size;
    myFree(ptr);
}
//(2)
inline void  operator delete[](void* ptr)
{
    cout << "jjhou global delete[](ptr), \t" << ptr << endl;
    myFree(ptr);
}

void test_operator()
{
    string *p = new string("My name is Ace");
    delete p;
    cout << "::countNew= " << ::countNew << endl;   //32
    cout << "::countDel= " << ::countDel << endl;   //32
}

void test_operator_array()
{
    string *p = new string[3];
    delete[] p;
    cout << "::countArrayNew= " << ::countArrayNew << endl; //104
    cout << "::countArrayDel= " << ::countArrayDel << endl; //104
}

void test_allocator()
{
    void *p4 = allocator<int>().allocate(7);  //jjhou global new(),     28
    allocator<int>().deallocate((int*)p4, 7); //jjhou global delete(ptr)
}

void test_list()
{
    list<int> lst;      //註：list object 本身不是 dynamic allocated.
    lst.push_back(1);   //jjhou global new(),   24  (註：每個 node是 24 bytes)
    lst.push_back(1);   //jjhou global new(),   24
    lst.push_back(1);   //jjhou global new(),   24
}

int main()
{
    test_operator();
    test_operator_array();
    test_allocator();
    test_list();
}

```

# 运行时类型识别

## dynamic_cast运算符
与static_cast一样，dynamic_cast的转换也需要目标类型和源对象有一定的关系：继承关系。 更准确的说，dynamic_cast是用来检查两者是否有继承关系。因此该运算符实际上只接受基于类对象的指针和引用的类转换。
所以，dynamic_cast的使用形式如下：
>dynamic_cast<type*>(e)
dynamic_cast<type&>(e)
dynamic_cast<type&&>(e)
其中，type必须是一个类类型，并且通常情况下该类型应该包含虚函数。如果类型中没有虚函数，则编译报错：`source type is not polymorphic`，可能是因为当类没有虚函数表的时候，dynamic_cast就不能用RTTI来确定类的具体类型

对于从子类到基类的指针转换，static_cast和dynamic_cast都是成功并且正确的（所谓成功是说转换没有编译错误或者运行异常；所谓正确是指方法的调用和数据的访问输出是期望的结果），这是面向对象多态性的完美体现。
而从基类到子类的转换，static_cast和dynamic_cast都是成功的，但是正确性方面：dynamic_cast的结果是空指针，而static_cast则是非空指针。但很显然，static_cast的结果应该算是错误的，
对于没有关系的两个类之间的转换，输出结果表明，dynamic_cast依然是返回一个空指针以表示转换是不成立的；static_cast直接在编译期就拒绝了这种转换。
``` cpp
#include <iostream>
#include <typeinfo>

using namespace std;

class Base
{
public:
    void print()
    {
        cout << "Base::print()" << endl;
    }

    virtual void trivial() //for compile
    { }

    virtual ~Base()
    {
    }
};

class Derived : public Base
{
public:
    Derived(): i(123456) {}

    void print()
    {
        cout << "Derived::print()" << endl;
    }

    void dosomething()
    {
        cout << "Derived::dosomething() " << i << endl;
    }
private:
    int i;
};

class Other
{
public:
    void print()
    {
        cout << "Ohter::print()" << endl;
    }
};

int main()
{
    //从子类转换到基类
    Derived *pd = new Derived;

    Base *static_cast_pb = static_cast<Base*>(pd);
    static_cast_pb->print(); //Base::print()
    //static_cast_pb->dosomething(); ‘class Base’ has no member named ‘dosomething’

    Base *dynamic_cast_pb = dynamic_cast<Base*>(pd);
    dynamic_cast_pb->print(); //Base::print()
    //dynamic_cast_pb->dosomething(); ‘class Base’ has no member named ‘dosomething’

    //从基类到子类的转换
    Base *pb = new Base();
    Derived *static_cast_pd = static_cast<Derived*>(pb);
    if(static_cast_pd)
    {
        static_cast_pd->print(); //static_cast_pd->print();
        static_cast_pd->dosomething(); //Derived::dosomething() 0 但是不安全，如果该类调用了子类本身的数据成员
    }
    else
    {
        cout << "static_cast null" << endl;
    }

    Derived *dynamic_cast_pd = dynamic_cast<Derived*>(pb); //dynamic_cast from base to derived is null
    if(dynamic_cast_pd)
    {
        dynamic_cast_pd->print();
        dynamic_cast_pd->dosomething();
    }
    else
    {
        cout << "dynamic_cast from base to derived is null" << endl;
    }

    //无关类型的转换
    //Other *static_cast_other = static_cast<Other*>(pb); //invalid static_cast from type ‘Base*’ to type ‘Other*’

    Other *dynamic_cast_other = dynamic_cast<Other*>(pb); //dynamic_cast from non-related is null
    if(dynamic_cast_other)
    {
        dynamic_cast_other->print();
    }
    else
    {
        cout << "dynamic_cast from non-related is null" << endl;
    }

    delete pd;

    delete pb;

    return 0;
}
```
