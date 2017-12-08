---
title: C++Primer5e第06章：函数
tags:
  - C++
toc: true
date: 2017-12-07 13:13:43
---
# 函数基础
## 局部对象
### 自动对象
只存在于块执行期间的对象称为自动对象，当块执行结束后，块中创建的自动对象的值就变成未定义的了。
形参是一种自动对象，函数开始时为形参申请存储空间，位于CPU寄存器或者栈上；函数执行结束后，形参销毁。
### 局部静态对象
如果需要函数的局部变量在函数调用以及之后时间有效，可以将局部变量定义成static类型。`局部静态对象(local static object)`在程序第一次经过对象时初始化，并且直到程序终止才被销毁。初始化的局部静态对象存储在.data段，如果局部静态变量没有显式的初始值，它将执行值初始化，内置类型的局部静态变量初始化为0。

# 参数传递
每次调用函数时都会重新创建它的形参，并用传入的实参对形参进行初始化。
形参初始化的机制和变量初始化一样。
- 当形参为引用类型时，它将绑定到对应的实参上，`也就是说引用形参是对应实参的别名`
- 传值调用时，形参和实参是两个相互独立的对象，实参的值被拷贝给形参，因此函数对形参做的修改不会影响实参

当拷贝大的类类型或者容器对象时效率低或者有些类型不支持拷贝，可以使用引用避免拷贝。<strong>如果函数不改变引用形参的值，最好将其声明为常量引用</strong>
一个函数只能返回一个值，但有时我们需要同时返回多个值，可以使用返回结构体形式或者使用引用形参。

## 数组形参
因为数组会被转换成指针，所以当我们为函数传递一个数组时，实际上传递的是指向数组首元素的指针。所以，调用者需要提供额外的信息：
- 使用标记指定数组的长度，例如C风格字符串结尾的空字符，空字符指ASCII值为0的控制字符，在程序代码中通常以转义序列'\0'表示
- 使用标准库规范，同时传递首元素和尾后元素指针 `void print(const int *beg, const int *end)`
- 显式传递一个表示数组大小的形参

<!--more-->

## 传递多维数组
数组第二维(以及后面所有维度)的大小是数组类型的一部分，不能省略：
``` cpp
//int (*matrix)[10] 指向含有10个整数的数组的指针
void print(int (*matrix)[10], int rowSize) { }
//因为编译器会忽略第一个维度，所以不要把它包含在形参列表中
void print(int matrix[][10], int rowSize) { }
```

## 可变形参的函数
C++11新标准提供两种处理不同数目实参的方法：
- 如果所有实参类型相同，可以传递一个名为initializer_list的标准库类型
- 如果实参的类型不同，使用可变参数模板
- 另外，C++还提供一种特殊的形参类型，即省略符

initializer_list类似vector，但initializer_list中的元素全部为常量，无法改变initializer_list中元素的值。如果想向initializer_list形参传递一个值的序列，则必须把序列放在一对花括号内：
``` cpp
void error_msg(initializer_list<string> il)
{
    for(auto beg = il.begin(); beg != il.end(); ++beg)
    {
        cout << *beg << " ";
    }
    cout << endl;
}
error_msg({"functionX", "expected", "actual"});
```

# 返回类型
返回一个值的方式和初始化一个变量或形参的方式完全一样：返回值用于初始化调用点的一个临时变量，该临时变量就是函数调用的结果。
同其他引用类型一样，如果函数返回引用，则该引用仅仅是它所引用对象的一个别名。例如：
``` cpp
const string& shorterString(const string &s1, const string &s2)
{
    return s1.size() <= s2.size() ? s1 : s2;
}
```
因为形参和返回类型都是const string的引用，所以不管调用函数还是返回结果都不会真正拷贝string对象。
因为函数完成后，它所占用的存储空间也被释放，局部变量的引用将指向不再有效的区域，所以`不要返回局部对象的指针或者引用`，否则导致未定义的行为。

## 引用返回左值
调用一个返回引用的函数得到左值，其他返回类型得到右值。可以像使用其他左值那样使用返回引用的函数调用，特别是，我们能为返回类型是非常量引用的函数的结果赋值。

## 列表初始化返回值
C++11新标准规定，函数可以返回花括号包围的值的列表。
``` cpp
vector<string> process()
{
    //expected和actual都是string对象
    if(expected.empty())
    {
        return {};  //返回一个空的vector对象
    }
    else if(expected == actual)
    {
        return {"funcitonX", "ok"}; //列表初始化的vector对象
    }
    else
    {
        return {"funcitonX", expected, actual};
    }
}
```

## 主函数main的返回值
如果控制到达了main函数的结尾处而且没有return语句，编译器将隐式插入一条返回0的return语句。返回0表示执行成功，返回其他表示执行失败，其中非0值的具体含义根据机器而定。为了使返回值与机器无关，cstdlib头文件定义了`EXIT_SUCCESS、EXIT_FAILURE`两个预处理变量。

## 返回数组指针
因为数组不能被拷贝，所以函数不能返回数组，不过可以返回数组的指针或者引用。最直接的方法是使用类型别名：
``` cpp
typedef int arrT[10];  //arrT是一个类型别名，表示含有10个整数的数组
arrT* func(int i) //func返回一个指向含有10个整数数组的指针

//返回数组指针的函数形式如下：
Type (*function(parameter_list))[dimension]

//下面func函数的声明没有使用类型别名：如果没有(*func(int i))两边的括号，函数返回类型为指针的数组
int (*func(int i))[10];
```
上述声明含义的理解：
``` cpp
func(int i)表示调用func需要一个int实参
(*func(int i))表示返回值是指针，可以对函数的结果进行解引用
(*func(int i))[10]表示解引用func的调用得到一个大小为10的数组
int (*func(int i))[10]表示数组中的元素是int类型
```

<strong>使用类型别名让函数的声明更清晰，也更容易理解</strong>
在C++11新标准中，还可以使用`尾置返回类型`，简化上述func声明。尾置返回类型在形参列表后面以一个->符号开头，为了表示函数真正的返回类型跟在形参列表后，在本该出现返回类型的地方放置一个auto：
``` cpp
auto func(int i) -> int (*)[10]
```

# 函数重载
如果同一作用域内的几个函数名字相同但形参列表不同，就称为`函数重载(overloaded)`。
如果形参是指针或者引用类型，可以通过区分指向的是常量对象还是非常量对象实现函数重载。因为非常量对象可以转换成const，所以非常量对象也可以调用常量对象的函数，但编译器会优先选用非常量版本的函数。

## const_cast和重载
``` cpp
const string& shorterString(const string &s1, const string &s2)
{
    return s1.size() <= s2.size() ? s1 : s2;
}
```
函数的参数和返回类型都是const string引用，我们能够对两个非常量的string实参调用这个函数，但返回结果仍然是const string。如果我们希望当实参不是常量时，返回的结果是一个普通的引用，使用使用const_cast可以做到：
``` cpp
string& shorterString(string &s1, string &s2)
{
    auto &r = shorterString(const_cast<const string&>(s1),
                            const_cast<const string&>(s2));
    return const_cast<string&>(r);
}
```

# 特殊用途语言特性
## 默认实参
<strong>通常应该在函数声明中指定默认实参，并且将声明放在合适的头文件中</strong>
因为声明是用户可以看到的部分，客户非常信任地使用这个特性，希望得到确定的结果，但如果在实现里使用了不同的默认值，那么将是灾难性的。`因此编译器禁止声明和定义同时定义默认实参`。
默认实参的参数必须位于形参列表的右边，如`int func(int a,int b=0, int c=5)`，但在定义的时候则不能加默认值，只能写成`int func(int a, int b, int c)`。

## 内联函数
inline关键字放在函数返回类型前，函数实现处必须写inline关键字。关键字inline在函数声明部分可以加也可以不加，建议不加，因为用户不需要知道一个函数是否是内联函数。假如在声明处加了inline，但是在实现处没有加inline，那么此函数被当做普通函数处理。与普通成员函数不同的是，inline成员函数的实现在头文件中，因为内联函数必须在调用该函数的每个文本文件中定义。假如，内联函数的实现写在了源文件中并且在这个源文件以外的文本文件中调用了此内联函数，那么编译可以通过，但是链接器会报“无法解析的外部符号”的错误。
<strong>内联只是想编译器发出一个请求，编译器可以忽略这个请求</strong>，所以内联函数应当是规模小、调用频繁的函数。

## constexpr函数
为了能在编译过程中随时展开，constexpr函数被隐式地指定为内联函数。constexpr函数必须满足以下条件：
- 返回值和参数必须是字面值类型
- 函数体必须只包含一个return语句
- 函数提可以包含其他的语句，但是这些语句在运行时不执行任何操作，比如空语句，类型别名以及using声明

``` cpp
constexpr int new_sz()
{
    return 42;
}
constexpr int foo = new_sz();
```
<strong>constexpr修饰的函数，返回值不一定是编译期常量</strong>，如果其传入的参数可以在编译时期计算出来，那么这个函数就会产生编译时期的值。但是，传入的参数如果不能在编译时期计算出来，那么constexpr修饰的函数就和普通函数一样了。不过，我们不必因此而写两个版本，所以如果函数体适用于constexpr函数的条件，可以尽量加上constexpr。
<strong>把内联函数和constexpr函数定义放在头文件</strong>

## assert预处理宏
assert是一种预处理宏，由预处理器管理而非编译器管理，定义在cassert头文件中。很多头文件都包含了cassert，所以即使我们没有直接包含cassert，也很有可能通过其他途径包含在我们程序中。
assert宏通常用于检测“不能发生”的条件，比如说指针无效等。

## NDEBUG预处理宏
assert的行为依赖一个名为NDEBUG的预处理变量。如果定义了NDEBUG，则assert什么也不做。默认状态没有定义NDEBUG，所以assert将执行运行时检查。
除了使用assert外，也可以使用NDEBUG编写自己的条件调试代码：
``` cpp
void print(const int ia[], size_t size)
{
#ifndef NDEBUG
    cerr << __func__ << ": array size is " << size << endl;
#endif
}
```
预处理器定义的其他名字：
``` cpp
__FILE__    存放文件名的字符串字面值
__LINE__    存放当前行号的整型字面值
__TIME__    存放文件编译时间的字符串字面值
__DATE__    存放文件编译日期的字符串字面值
```

# 函数匹配
函数匹配过程如下：
1. 在函数调用点可见的声明内查找与被调用的函数同名的候选函数
2. 从候选函数选出可行函数，可选特征包括形参与实参数量匹配以及类型相同，或者实参可以转换成形参类型
Note：如果函数有默认实参，调用时实参数量可以少于形参数量。
3. 寻找最佳匹配，精确匹配比需要类型转换的匹配更好
调用重载函数应该尽量避免强制类型转换，如果在实际应用中确实需要强制类型转换，则说明我们设计的形参集合不合理。

# 函数指针
``` cpp
//使用typedef定义函数指针别名
typedef bool(*FuncP)(const string&, const string&);
//使用using定义函数指针别名
using PF = int(*)(int*, int);
```
如果我们明确知道返回的是哪一个函数，可以使用decltype简化返回函数指针类型的声明方式。例如：
``` cpp
string::size_type sumLength(const string&, const string&);
string::size_type largeLength(const string&, const string&);
//根据形参值，getFcn返回指向sumLength或者largeLength的指针
decltype(sumLength) *getFcn(const string&);
```
当将decltype作用于某个函数时，返回函数类型而非指针类型，因此，我们显式加上`*`表明我们需要返回指针，而非函数本身。
