---
title: C++Primer5e第14章：重载运算与类型转换
tags:
  - C++
toc: true
date: 2017-12-22 13:30:48
---
当运算符作用于类类型的运算对象时，可以通过运算符重载重新定义该运算符的含义。合理使用运算符重载能令我们的程序更易于编写和阅读。
# 基本概念
重载的运算符是具有特殊名字的函数：它们的名字由关键字operator和其后要定义的运算符号共同组成。除了重载的函数调用运算符`operator()`之外，其他重载运算符不能含有默认实参。
当一个重载的运算符是成员函数时，则它的第一个(左侧)运算对象绑定到隐式的this指针上。
对于一个运算符函数来说，它或者是类的成员，或者至少包含有一个类类型的参数，因此我们不能改变内置类型的运算符的含义。
我们可以使用直接和间接两种方式调用重载的运算符函数：
``` cpp
//非成员运算符函数的等价调用
data1 + data2;  //普通的表达式
operator+(data1, data2); //等价的函数调用

//调用成员函数的operator+=运算符
data1 += data2; //this绑定到data1的地址、将data2作为实参传入函数
data1.operator+=(data2);  
```
<strong>通常情况下，不应该重载逗号、取地址、逻辑与和逻辑或运算符。</strong>
当内置的运算符和我们自己的操作之间存在逻辑映射关系时，使用重载的运算符更自然直观。但是，过分滥用运算符重载也会使我们的类变得难以理解。
<!--more-->
运算符定义为成员函数还是普通的非成员函数规则：
>赋值(=)、下标([])、调用(())和成员访问箭头(->)运算符必须是成员函数
复合赋值运算符一般来说应该是成员函数，但并非必须
改变对象状态的运算符或者与给定类型密切相关的运算符，如递增、递减和解引用运算符通常是成员函数
具有对称性的运算符可能转换任意一端的运算对象，例如算术、相等性、关系和位运算符，它们通常是普通的非成员函数

当我们把运算符定义成类成员函数时，它的左侧运算对象必须是运算符所属类的一个对象。

# 输入和输出运算符
IO标准库分别使用>>和<<执行输入和输出操作。类则需要自定义合适其对象的新版本以支持IO操作。
## 重载输出运算符
通常，输出运算符的第一个参数是一个非常量ostream对象的引用。ostream是非常量是因为向流写入会改变其状态，引用是因为我们无法复制ostream对象。第二个参数是我们需要打印的类类型的常量引用。为了与其他输出运算符保持一致，operator<<一般要返回它的ostream形参。
Sales_data的输出运算符：
``` cpp
ostream& operator<<(ostream &os, const Sales_data &item)
{
    os << item.isbn() << " " << item.units_sold;
    return os;
}
```
如果我们为类自定义IO运算符，则必须将其定义为非成员函数。因为成员函数要求左侧的运算对象必须我们的类的一个对象，但IO类属于标准库，我们无法给标准库添加任何成员。
`IO运算符通常需要读写类的非公有数据成员，所以IO运算符一般被声明为友元`。

## 重载输入运算符
第二个参数必须是个非常量是因为输入运算符本身的目的就是将数据读入到这对象中。
Sales_data的输入运算符：
``` cpp
istream &operator>>(istream &is, Sales_data &item)
{
    double price;
    is >> item.bookNo >> item.units_sold >> price;
    if(is) //检查输入是否成功
    {
        item.revenue = item.units_sold * price;
    }
    else
    {
        item = Sales_data(); //输入失败：对象被赋予默认状态
    }
    return is;
}
```
当读取操作发生错误时，输入运算符应该负责从错误中恢复。另外，在输入运算符中做数据校验工作时，输入运算符只设置failbit条件状态标示失败信息。

## 算术和关系运算符
通常情况下，把算术和关系运算符定义成非成员函数以允许对左侧或右侧的运算对象进行转换。因为这些运算符一般不需要改变运算对象的状态，所以形参都是常量引用。
如果类同时定义了算术运算符合相关的符合赋值运算符，则通常情况应该使用复合赋值来实现算术运算符。在并不损失性能的情况下，代码复用可以减少重复代码。
``` cpp
Sales_data operator+(const Sales_data &lhs, const Sales_data &rhs)
{
    Sales_data sum = lhs; //把lhs的数据成员拷贝给sum
    sum += rhs;
    return sum;
}
```

### 相等运算符
相等运算符和不相等运算符中的一个应该把工作委托给另外一个，即其中一个负责实际比较对象工作，而另一个只是调用那个真正工作的运算符。
如果某个类在逻辑上有相等性的含义，则该类应该定义operator==，这样做可以使得用户更容易使用标准库算法来处理这个类。

### 关系运算符
定义了相等运算符的类也常常包含关系运算符。特别是，因为关联容器和一些算法要用到小于运算符，所以operator<会比较有用。
如果存在唯一一种逻辑可靠的<定义，则应该考虑为这个类定义<运算符。
如果类同时还包含==，则当且仅当<的定义和==产生的结果一致时才定义<运算符。

### 赋值运算符
除了拷贝赋值和移动赋值运算符，类还可以定义其他赋值运算符以使用别的类型作为右侧运算对象。
为StrVec类定义接受花括号的参数列表的赋值运算符：
``` cpp
StrVec& StrVec::operator=(std::initializer_list<std::string> il)
{
    auto newdata = alloc_n_copy(il.begin(), il.end()); // 分配空间并拷贝元素
    free(); //销毁对象中的元素并释放空间
    elements = newdata.first; //更新数据成员
    first_free = newdata.second;
    cap = newdata.second;
    return *this;
}
StrVec sVec4;
sVec4 = {"a", "an", "the"};
```
为了与内置类型的复合赋值保持一致，类中的复合赋值运算符也要返回其左侧运算对象的引用。
复合赋值运算符一般定义在类的内部。Sales类中的复合赋值运算符：
``` cpp
//左侧运算对象绑定到隐式的this指针
Sales_data& Sales_data::operator+=(const Sales_data &rhs)
{
    units_sold += rhs.units_sold;
    revenue += rhs.revenue;
    return *this;
}
```

# 下标运算符
为了与下标的原始定义兼容，下标运算符通常以运算的引用作为返回值，这样下标可以出现在赋值运算符的左右两侧。
如果一个类包含下标运算符，则它通常会定义两个版本：一个返回普通引用；另一个是类的常量成员并返回常量引用，当作用于一个常量对象时，下标运算符返回常量引用确保我们不会给返回的对象赋值。
``` cpp
class StrVec
{
public:
    std::string& operator[](std::size_t n) { return elements[n]; }

    const std::string& operator[](std::size_t n) const
    {
        return elements[n];
    }
    //其他成员函数
private:
    std::string *elements;
};
```

# 递增和递减运算符
定义递增和递减运算符的类应该同时定义前置版本和后置版本，并且通常定义为类的成员函数。
## 前置递增/递减运算符
StrBlobPtr类中定义的前置递增和递减运算符：
``` cpp
class StrBlobPtr
{
public:
    StrBlobPtr& operator++();
    StrBlobPtr& operator--();
};

StrBlobPtr& StrBlobPtr::operator++()
{
    check(curr, "increment past end of StrBlobPtr");
    ++curr;
    return *this;
}

StrBlobPtr& StrBlobPtr::operator--()
{
    --curr;
    check(curr, "decrement past begin of StrBlobPtr");
    return *this;
}
```

## 区分前置和后置运算符
因为前置和后置版本使用的是同一个符号，为了解决这个问题，后置版本接受一个额外不使用的int类型参数。当我们使用后置运算符时，编译器为这个形参提供一个值为0的实参。但该参数并不参与运算，仅仅作为区分前置和后置版本标识。
<strong>为了与内置版本保持一致，后置运算符应该返回递增或递减之前的值，返回的形式是一个值而非引用</strong>
StrBlobPtr类后置的递增和递减运算符，我们的后置运算符调用各自的前置版本完成实际的工作：
``` cpp
class StrBlobPtr
{
public:
    StrBlobPtr operator++(int);
    StrBlobPtr operator--(int);
};

StrBlobPtr StrBlobPtr::operator++(int)
{
    StrBlobPtr ret = *this; //记录当前的值
    ++*this; //向前移动一个元素
    return ret; //返回之前记录的状态
}

StrBlobPtr StrBlobPtr::operator--(int)
{
    StrBlobPtr ret = *this; //记录当前的值
    --*this; //向后移动一个元素
    return ret; //返回之前记录的状态
}
```

# 成员访问运算符
我们将解引用和箭头运算符定义成const成员函数，因为获取一个元素不会改变对象的状态，返回非常量的引用或者指针。
StrBlobPtr对应的成员访问运算符：
``` cpp
class StrBlobPtr
{
public:
    std::string& operator*() const
    {
        auto p = check(curr, "dereference past end");
        return (*p)[curr]; //(*p)是对象所指的vector
    }

    std::string* operator->() const
    {
        return &this->operator*(); //将实际工作委托给解引用运算符
    }
};
```
重载的箭头运算符必须返回类的指针或者自定义了箭头运算符的某个类的对象。

# 函数调用运算符
如果类定义了调用运算符，则该类的对象称作`函数对象(function object)`。一个类可以定义多个不同版本的调用运算符，相互间在参数数量或类型上有所区别。
和其他类一样，函数对象除了operator()之外也可以包含其他成员。函数对象类通常含有一些数据成员，这些成员被用于定制调用运算符中的操作。
函数对象常常作为泛型算法的实参
``` cpp
class IfElseThen
{
public:
    IfElseThen() = default;
    IfElseThen(int i, int j, int k): a(i), b(j), c(k) {}
    int operator()(int i1, int i2, int i3)
    {
        return i1 ? i2 : i3;
    }
private:
    int a, b, c;
};

int main()
{
    IfElseThen ifElseThen(1, 2, 3);
    cout << ifElseThen(4, 5, 6) << endl; //5
    return 0;
}
```

## lambda是函数对象
在lambda表达式产生的类中包含有一个重载的函数调用运算符，例如，我们根据单词长度排序中的lambda表达式：
``` cpp
stable_sort(words.begin(), words.end(), [](const string &a, const string &b) {return a.size() < b.size();});
//使用函数对象重写上面的lambda表达式
class ShorterString
{
public:
    bool operator()(const string &a, const string &b) const
    {
        return a.size() < b.size();
    }
};

stable_sort(words.begin(), words.end(), ShorterString());
```
当一个lambda表达式通过引用捕获变量时，由程序保证引用对象的存在，因此编译器可以直接使用引用而不需要在lambda产生的类中将其存储为数据成员。
但通过值捕获的变量被拷贝到lambda中，这种lambda产生的类必须为每个值捕获的变量建立对应的数据成员，同时创建构造函数，用捕获的变量的值来初始化数据成员。
``` cpp
auto wc = find_if(words.begin(), words.end(), [sz](const string &a) {return a.size() >= sz;});
//上述lambda表达式产生的类等价于
class SizeComp
{
public:
    SizeComp(size_ n):sz(n) {}
    bool operator()(const string &a) const
    {
        return a.size() >= sz;
    }
private:
    size_t sz;    
};
auto wc = find_if(words.begin(), words.end(), SizeComp(sz));
```
lambda表达式产生的类不包含默认构造函数，赋值运算符以及析构函数。是否包含拷贝/移动构造函数需要根据捕获的数据成员类型而定。
<strong>什么情况使用lambda表达式？什么情况使用函数对象？</strong>
在C++11中，lambda是通过匿名的函数对象来实现的，因此可以把lambda看作是函数对象在使用方式上的简化。当代码需要一个简单的函数，并且这个函数并不会在其他地方使用，就可以使用lambda来实现。但如果这个函数需要多次使用，并且它需要保存状态的话，使用函数对象更合适。

## 标准库定义的函数对象
标准库定义了一组表示算术运算符、关系运算符和逻辑运算符的类，每个类分别定义了一个执行命名操作的调用运算符。所有类型定义在functional头文件中。
``` cpp
plus<int> intAdd; //可执行int加法的函数
int sum = intAdd(10, 20); //sum = 30
//在算法中使用标准库函数对象
sort(svec.begin(), svec.end(), greater<string>()); //sort默认使用升序排列，使用greater函数对象实现降序排列
```
另外，标准库规定其函数对象对于指针同样适用。
``` cpp
vector<string *> nameTable; //指针的vector
//错误：nameTable中的指针彼此之间没有关系，所以<将产生未定义的行为
sort(nameTable.begin(), nameTable.end(), [](string *a, string *b) { return a < b; });
//正确：标准库规定指针的less是定义良好的
sort(nameTable.begin(), nameTable.end(), less<string*>());
```
关联容器使用`less<key_type>`对元素排序，因此我们可以定义一个指针的set或者在map中使用指针作为关键值而不需要声明less。
标准库函数对象以及适配器使用示例：
``` cpp
//std::bind1st和std::bind2nd将二元函数转换为一元函数
//统计大于1024的值有多少
using namespace std::placeholders;
vector<int> ivec = {2000, 3, 1024, 7, 1025, 2046, 1, 2};
count_if(ivec.begin(), ivec.end(), bind2nd(greater<int>(), 1024));
count_if(ivec.begin(), ivec.end(), bind(greater<int>(), _1, 1024));
//找到第一个不等于pooh的字符串
find_if(vec.begin(), vec.end(), bind2nd(not_equal_to<string>(), "pooh"));
//将所有值乘以2
transform(vec.begin(), vec.end(), vec.begin(), bind2nd(multiplies<int>(), 2));
//判断一个int值能否被int容器中的所有元素整除
bool dividedByAll(vector<int> &ivec, int dividend)
{
    return count_if(ivec.begin(), ivec.end(), bind1st(modulus<int>(), dividend)) == 0;
}
//使用count_if每次会遍历容器中所有的元素，改用find_if当遍历到不能整除的数时退出
bool dividedByAll(vector<int> &ivec, int dividend)
{
    return find_if(ivec.begin(), ivec.end(), bind1st(modulus<int>(), dividend)) == ivec.end();
}
```

## 可调用对象与function
C++语言中可调用对象包括：
- 函数
- 函数指针
- lambda表达式
- bind创建的对象
- 重载了函数调用运算符的类

两个不同类型的可调用对象却可能共享同一种`调用形式(call signature)`。调用形式指明了调用返回类型以及传递给调用的实参类型。
function是一个类模板，模板中的参数信息是该function类型能够表示的对象的调用形式。
``` cpp
function<int(int, int)> f1 = add;      //函数指针
function<int(int, int)> f2 = divide(); //函数对象类的对象
function<int(int, int)> f3 = [](int i, int j) {return i * j; }; //lambda

std::map<string, funciton<int(int, int)>> binops = {
    {"+", add},  //函数指针
    {"-", std::minus<int>()}, //标准库函数对象
    {"/", divide()}, //用户定义的函数对象
    {"*", [](int i, int j) { return i * j; }}, //未命名的lambda
    {"%d", mod}}; //命名的lambda对象
```
我们不能直接将重载函数的名字存入function类型的对象中：
``` cpp
int add(int i, int j) {return i + j;}
Sales_data add(const Sales_data &, const Sales_data &);
map<string, function<int(int, int)>> binops;
binops.insert({"+", add}); //错误：哪个add？
```
解决二义性问题的途径是使用函数指针而非函数名字，或者lambda来消除二义性：
``` cpp
int (*fp)(int, int) = add; //指针指向接受两个int的版本
binops.insert({"+", fp});  //正确：fp指向一个正确的add版本

binops.insert({"+", [](int i, int j) { return add(a, b); } });
```

# 重载、类型转换与运算符
类型转换运算符负责将一个类类型的值转换成其他类型。类型转换函数的一般形式如下：
``` cpp
operator type() const; //type表示某种类型

//定义含有类型转换运算符的类
class SmallInt
{
public:
    SmallInt(int i = 0): val(i)
    {
        if(i < 0 || i > 255)
        {
            throw std::out_of_range("Bad SmallInt value");
        }
    }

    operator int() const { return val; }
private:
    std::size_t val;
};

SmallInt si;
si = 4; //首先将4隐式地转换成SmallInt，然后调用SmallInt::operator=
si + 3; //首先将si隐式地转换成int，然后执行整数的加法
```
为了避免隐式类型转换运算符产生意外结果，C++11新标准引入`显式的类型转换运算符`。
``` cpp
class SmallInt
{
public:
    //编译器不会自动执行这一类型转换
    explicit operator int() const { return val; }
};
SmallInt si = 3; //正确：SmallInt的构造函数不是显式的
si + 3; //错误：此处需要隐式的类型转换，但类的转换运算符是显式的
static_cast<int>(si) + 3; //正确：显式地请求类型转换
```
如果表达式被用作条件，显式的类型转换将被隐式地执行。
<strong>向bool的类型转换通常用在条件部分，因此operator bool一般定义成explicit的。</strong>
