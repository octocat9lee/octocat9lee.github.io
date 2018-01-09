---
title: C++Primer5e第19章：特殊工具与技术（二）
tags:
  - C++
toc: true
date: 2018-01-08 09:41:14
---
# 类成员指针
成员指针(pointer to member)是可以指向类的非静态成员的指针。成员指针指向类的成员，而非类的对象。当初始化一个成员指针时，我们令其指向类的某个成员，但是不指定该成员所属的对象；直到使用成员指针时，才提供成员所属的对象。
``` cpp
class Screen
{
public:
    typedef std::string::size_type pos;
    char get_cursor() const { return contents[cursor]; }
    char get() const;
    char get(pos ht, pos wd) const;
private:
    std::string contents;
    pos cursor;
    pos height, width;
};
```
## 数据成员指针
数据成员指针必须包含所属的类，因此，我们必须在*之前添加classname::以表示当前定义的指针可以指向包含classname的成员。我们将取地址运算符作用于Screen类的成员而非内存中的一个该类对象。定义数据成员指针方式：
``` cpp
const string Screen::*pdata = &Screen::contents;
//C++11中声明成员指针的简单方法是使用auto或decltype：
auto pdata = &Screen::contents;
```
<strong>当我们初始化一个成员指针或为成员指针赋值时，该指针并没有指向任何数据。</strong>
`.*和->*`运算符解引用数据成员指针获得该对象的成员：
``` cpp
Screen myScreen, *pScreen = &myScreen;
auto s = myScreen.*pdata;
s = pScreen->*pdata;
```
<!--more-->
### 返回数据成员指针的函数
因为数据成员一般情况下是私有的，所以我们通常不能直接获得数据成员的指针。最好定义一个函数，令其返回值是指向该成员的指针：
``` cpp
class Screen {
public:
    //data是一个静态成员，返回一个成员指针
    static const std::string Screen::*data()
    { return &Screen::contents; }
};

const string Screen::*pdata = Screen::data();
auto s = myScreen.*pdata;
```

## 成员函数指针
我们同样可以定义指向类的成员函数的指针。最简单的方法是使用autuo类推断类型：
``` cpp
//pmf是一个指针，它可以指向Screen的某个常量成员函数
//但前提是该函数不接受任何实参，并且返回一个char
auto pmf = &Screen::get_cursor;
```
如果成员函数存在重载，我们必须显式声明函数类型以明确指出我们想要使用的是哪个函数。例如，我们可以声明一个指针，令其指向含有两个形参的get：
``` cpp
char (Screen::*pmf2)(Screen::pos, Screen::pos) const; //Screen::*两端的括号必不可少
pmf2 = &Screen::get; //必须显式地使用取地址运算符

//使用成员函数指针
Screen myScreen, *pScreen = &myScreen;
char c1 = (pScreen->*pmf)();
char c2 = (myScreen.*pmf2)(0, 0);
```
因为函数调用运算符的优先级较高，所有在声明指向成员函数的指针并使用这样的指针进行函数调用时，括号必不可少：`(C::*p)(parms)和(obj.*p)(args)`。

### 使用成员指针的类型别名
使用类型别名或typedef可以让成员指针更容易理解。
``` cpp
//Action是一种可以指向Screen成员函数的指针，它接受两个pos实参，返回一个char
using Action = char (Screen::*)(Screen::pos, Screen::pos) const;
Action get = &Screen::get; //get指向Screen的get成员
//我们可以将成员函数的指针作为函数的参数或者返回值
Screen& action(Screen&, Action = &Screen::get);
//当我们调用action时，只需要将Screen的一个符合要求的函数指针或地址传入即可
Screen myScreen;
action(myScreen);       //使用默认实参
action(myScreen, get);  //使用我们之前定义的变量get
action(myScreen, &Screen::get); //显式地传入地址
```

### 成员指针函数表
将函数指针存入一个函数表中，如果一个类含有几个相同类型的成员，则成员指针函数表可以帮助我们从这些成员中选择一个。
``` cpp
#include <string>

class Screen
{
public:
    typedef std::string::size_type pos;
    //Action是一个指针，可以用任意一个光标移动函数赋值
    using Action = Screen& (Screen::*)();
    //指定具体移动方向
    enum Directions {HOME, FORWARD, BACK, UP, DOWN, };

    Screen() = default;
    Screen(pos ht, pos wd, char c)
        : contents(ht*wd, c), height(ht), width(wd) {}
    char get_cursor() const { return contents[cursor]; }
    char get() const;
    char get(pos ht, pos wd) const;

    Screen& move(Directions direction);

    //data是一个静态成员，返回一个成员指针
    static const std::string Screen::*data()
    {
        return &Screen::contents;
    }

private:
    //光标移动函数
    Screen& home();
    Screen& forward();
    Screen& back();
    Screen& up();
    Screen& down();

private:
    static Action Menu[]; //函数表
    std::string contents;
    pos cursor = 0;
    pos height = 0, width = 0;
};

//定义并初始化函数表
Screen::Action Screen::Menu[] = {&Screen::home, &Screen::forward, &Screen::back, &Screen::up, &Screen::down};

Screen& Screen::move(Directions direction)
{
    //运行this对象中索引值为direction的元素，必须利用.*或->*将指针绑定到特定的对象上
    return (this->*Menu[direction])();
}

//使用枚举成员调用move函数
Screen myScreen;
myScreen.move(Screen::HOME);
myScreen.move(Screen::DOWN);
```
## 将成员函数用作可调用对象
成员函数的指针与普通函数指针不同，成员指针不是一个可调用对象，这样的指针不支持函数调用运算符。所以我们不能直接将一个指向成员函数的指针传递给算法。
<strong>使用标准库模板function生成一个可调用对象</strong>
``` cpp
function<bool (const string&)> fcn = &string::empty;
find_if(svec.begin(), svec.end(), fcn);
```
通常情况下，执行成员函数的对象将被传给隐式的this形参。
当一个function对象包含有一个指向成员函数的指针时，function类知道它必须使用正确的指向类成员的指针运算符来执行函数调用。也就是说function将如下的函数调用：
``` cpp
//假设it是find_if内部的迭代器，则*it是给定范围内的一个对象
if(fcn(*it)) //假设fcn是find_if内部的一个可调用对象的名字
```
转换成如下形式：
``` cpp
if(((*it).*p)()) //假设p是fcn内部的一个指向成员函数的指针
```
<strong>使用mem_fn生成一个可调用对象</strong>
通过使用标准库功能`mem_fn`来让编译器负责推断成员的类型，它也可以从成员指针生成一个可调用对象。mem_fn也定义在functional头文件中。和function不同的是，mem_fn可以根据成员指针的类型推断可调用对象的类型，而无须用户显式地指定：
``` cpp
find_if(svec.begin(), svec.end(), mem_fn(&string::empty));
```
mem_fn生成的可调用对象可以通过对象调用，也可以通过指针调用：
``` cpp
auto f = mem_fn(&string.empty); //f接受一个string或一个string*
f(*svec.begin());  //正确：传入一个string对象，f使用.*调用empty
f(&svec[0]);       //正确：传入一个string的指针，f使用->*调用empty
```
<strong>使用bind生成一个可调用对象</strong>
``` cpp
//选择范围中的每个string，并将其bind到empty的第一个隐式实参上
auto it = find_if(svec.begin(), svec.end(), bind(&string::empty, _1));

//bind生成的可调用对象的第一个实参既可以是指针，也可以是引用
auto f = bind(&string::empty, _1);
f(*svec.begin());  //正确：传入一个string对象，f使用.*调用empty
f(&svec[0]);       //正确：传入一个string的指针，f使用->*调用empty
```

# union:一种节省空间的类
联合(union)是一种特殊的类。一个union可以有多个数据成员，但是在任意时刻只有一个数据成员可以有值。当我们给union的某个成员赋值之后，该union的其他成员就变成未定义的状态了。
分配一个union对象的存储空间至少要能容纳它的最大的数据成员。
union提供了一种有效的途径使得我们可以方便地表示一组类型不同的互斥值。
<strong>使用类管理union成员</strong>
对于union来说，要想构造或销毁类类型的成员必须执行非常复杂的操作，因此我们通常把含有类类型成员的union嵌套在另一个类中。这个类管理并控制union的类型成员有关的状态转换。
为了追踪union中到底存储了什么类型的值，我们通常会定义一个独立的对象，该对象称为union的判别式。
<strong>带有string类型的union管理类Token实现：</strong>
``` cpp
//token.h
#include <string>

class Token
{
public:
    Token(): tok(INT), ival{0} { }
    Token(const Token &t): tok(t.tok) { copyUnion(t); }
    Token& operator=(const Token &t);
    Token(Token &&other) noexcept;
    Token &operator=(Token &&other) noexcept;

    ~Token()
    {
        //如果union含有一个string成员，则我们必须销毁它
        if(tok == STR)
        {
            sval.std::string::~string();
        }
    }

    //下面的赋值运算符负责设置union的不同成员
    Token& operator=(const std::string &s);
    Token& operator=(char c);
    Token& operator=(int i);
    Token& operator=(double d);

private:
    //检查判别式，然后酌情拷贝union成员
    void copyUnion(const Token &t);

private:
    enum {INT, CHAR, DBL, STR} tok; //判别式
    union //匿名union
    {
        char cval;
        int  ival;
        double dval;
        std::string sval;
    }; //每个Token对象含有一个该未命名union类型的未命名成员
};


//token.cpp
#include "token.h"

using namespace std;

//赋值运算符必须处理string成员的三种可能情况：
//都是string，都不是string，只有一个是string
Token& Token::operator=(const Token &t)
{
    //本身是string，而t不是string，则必须释放原来的string
    if(tok == STR && t.tok != STR)
    {
        sval.~string();
    }
    //都是string
    if(tok == STR && t.tok == STR)
    {
        sval = t.sval; //无须构造新的string
    }
    else
    {
        copyUnion(t);
    }
    tok = t.tok;
    return *this;
}

Token& Token::operator=(const std::string& s)
{
    if(tok == STR) //如果当前存储的是string，可以直接赋值
    {
        sval = s;
    }
    else //否则需要先构造一个string
    {
        new(&sval) string(s); //placement new
    }
    tok = STR; //更新判别式
    return *this;
}

Token& Token::operator=(char c)
{
    if(tok == STR) //如果当前存储的是string，释放它
    {
        sval.~string();
    }
    cval = c; //为成员赋值
    tok = CHAR; //更新判别式
    return *this;
}

Token& Token::operator=(int i)
{
    if(tok == STR) //如果当前存储的是string，释放它
    {
        sval.~string();
    }
    ival = i; //为成员赋值
    tok = INT; //更新判别式
    return *this;
}

Token& Token::operator=(double d)
{
    if(tok == STR) //如果当前存储的是string，释放它
    {
        sval.~string();
    }
    dval = d; //为成员赋值
    tok = DBL; //更新判别式
    return *this;
}

Token::Token(Token &&other) noexcept
    :tok(INT), ival(0)
{
    tok = other.tok;
    ival = other.ival;
    other.tok = INT;
    other.ival = 0;
}

Token& Token::operator=(Token &&other) noexcept
{
    if(this != &other)
    {
        tok = other.tok;
        ival = other.ival;
        other.tok = INT;
        other.ival = 0;
    }
    return *this;
}

void Token::copyUnion(const Token &t)
{
    switch(t.tok)
    {
        case Token::INT: ival = t.ival; break;
        case Token::CHAR: cval = t.cval; break;
        case Token::DBL: dval = t.dval; break;
        //使用placement new表达式构造string
        case Token::STR: new(&sval) string(t.sval); break;
        default: break;
    }
}

//main.cpp
#include "token.h"

int main()
{
    Token t1;
    t1 = "hello";
    t1 = 1;
    t1 = 1.234;
    t1 = 'a';

    Token t2;
    t2 = t1;
    Token t3(std::move(t2));
    return 0;
}
```

# 固有的不可移植特性
所谓的不可移植的特性是指因机器而异的特性，当我们将含有不可移植特性的程序从一台机器转移到另一台机器时，通常需要重新编写该程序。<strong>算术类型的大小在不同机器上不一样</strong>

## 位域
类可以将其数据成员定义成位域，在一个位域中含有一定数量的二进制位。<strong>位域在内存中的布局是与机器相关的</strong>。
通常情况下最好将位域设为无符号类型，存储在带符号类型中的位域的行为将因具体实现而定。
如果一个类定义了位域成员，则它通常也会定义一组内联的成员函数以检验或设置位域的值。

## 链接指示：extern “C”
C++程序有时需要调用C语言编写的函数。C++使用`链接指示(linkage directive)`指出任意非C++函数所用的语言。
<strong>声明一个非C++的函数</strong>
链接指示可以有两种形式：单个的或者复合的。链接指示不能出现在类定义或函数定义的内部，另外，链接指示必须在函数的每个声明中都出现。
cstring头文件中的某些C函数声明：
``` cpp
//可能出现在C++文件<cstring>中的链接指示
//单语句链接指示
extern "C" size_t strlen(const char *);

//复合语句链接指示
extern "C" {
    int strcmp(const char*, const char*);
    char* strcat(char*, const char*);
}

//当一个#include指示被放置在复合链接指示的花括号中时，头文件中的所有普通函数声明都被认为是由链接指示的语言编写的
extern "C" {
    #include <string.h> //操作C风格字符串的C函数
}

//指向extern “C”函数的指针
//pf指向一个C函数，该函数接受一个int返回void
extern "C" void (*pf)(int); //指向C函数的指针与指向C++函数的指针是不一样的类型
```

当使用链接指示时，它不仅对函数有效，而且对返回类型和形参类型的函数指针也有效：
``` cpp
//f1是一个C函数，它的形参是一个指向C函数的指针
extern "C" void f1(void(*)(int

//如果希望C++函数传入C函数的指针，则必须使用类型别名
//FC是一个指向C函数的指针
extern "C" typedef void FC(int);
//f2是一个C++函数，该函数的形参是指向C函数的指针
void f2(FC *);
```
链接指示不仅对f1有效，对函数指针也有效。当调用f1时，必须传给它一个C函数的名字或者指向C函数的指针。

<strong>导出C++函数到其他语言</strong>
通过使用链接指示对函数进行定义，我们可以令一个C++函数在其他语言编写的程序中可用：
``` cpp
//calc函数可以被C程序调用
extern "C" double calc(double dparm) { /* ... */ }
```
有时需要在C和C++中编译同一个源文件，为了实现这一目的，在编译C++版本的程序时预处理定义__cplusplus。利用该变量，我们可以在编译C++程序时有条件包含进来一些代码：
``` cpp
#ifdef __cplusplus
//正确：正在编译C++程序
extern "C"
{
#endif
int strcmp(const char*, const char*);
#ifdef __cplusplus
}
#endif
```
