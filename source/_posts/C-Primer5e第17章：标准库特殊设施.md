---
title: C++Primer5e第17章：标准库特殊设施
tags:
  - C++
toc: true
date: 2018-01-03 08:54:00
---
# tuple类型
当我们希望将一些数据组合成单一对象，但又不想麻烦地定义一个新数据结构来表示这些数据时，tuple是非常有用的。我们可以将tuple看作一个“快速而随意”的数据结构。
tuple类型及其伴随类型和函数都定义在tuple头文件中。
## 定义和初始化tuple
``` cpp
tuple<size_t, size_t, size_t> threeD{1, 2, 3};
tuple<string, vector<double>, int, list<int>> somVal("constants", {0.01, 3.14}, 100, {1, 2, 3, 4, 5});
auto item = make_tuple("0990-0990-0990", 3, 20.00); //使用初始值的类型推断tuple类型，tuple<const char *, int, double>
//访问tuple的成员
auto book = get<0>(item);  //返回第一个数据成员的引用
auto cnt = get<1>(item);
get<2>(item) *= 0.8;       //打8折
auto price = get<2>(item);
cout << book << " " << cnt << " " << price << endl;
//使用辅助模板类获取tuple成员数量和类型
typedef decltype(item) trans;  //使用decltype为item定义类型别名
size_t sz = tuple_size<trans>::value; //sz = 3
cout << "tuple size " << sz << endl;
tuple_element<1, trans>::type bookCnt = get<1>(item); //bookCnt的类型与item中第二个成员的类型相同
cout << "book count " << bookCnt << endl;
```
[C++11::遍历tuple中的元素](http://blog.csdn.net/lanchunhui/article/details/49868077)
<!--more-->
## 使用tuple返回多个值
tuple的一个常见用途是从一个函数返回多个值。
``` cpp
//match包含三个成员：书店的索引和两个指向书店vector中元素的迭代器
typedef tuple<vector<Sales_data>::size_type,
              vector<Sales_data>::const_iterator,
              vector<Sales_data>const_iterator> matches;

//定义类替代tuple方式
class matches
{
public:
    matches(vector<Sales_data>::size_type n, pair<vector<Sales_data>::const_iterator, vector<Sales_data>::const_iterator> f)
    :num(n), first(f.first), last(f.second) {}

private:
    vector<Sales_data>::size_type num;
    vector<Sales_data>::const_iterator first;
    vector<Sales_data>::const_iterator last;    
};
```
如果只是简单使用搜索结果，pair和tuple都是简单直接的实现方式。若搜索结果还需要进行复杂的计算、处理，定义一个类对其进行封装更好。

# bitset类型
标准库定义bitset类，使得位运算的使用更为容易，并且能够处理超过最长整型类型大小的位集合。
bitset类定义在头文件bitset中。
## 定义和初始化bitset
当我们使用一个整型值来初始化bitset时，此值将被转换为unsigned long long类型并被当作位模式来处理。
``` cpp
//bitvec1比初始值小：初始值中的高位被丢弃
bitset<13> bitvec1(0xbeef);  //二进制序列为1111011101111
//bitvec2比初始值大：它的高位被置为0
bitset<20> bitvec2(0xbeef);  //二进制序列为00001011111011101111
//在64位机器中，unsigned long long 0ULL是64个0比特，因此~0ULL是64比特1
bitset<128> bitvec3(~0ULL);  //0~63bit为1，64~127bit为0

//当我们使用字符串表示数时，字符串中下标最小的字符对应高位，反之亦然。
bitset<32> bitvec4("1100"); //2，3两位为1，其余为0

//使用子串作为初始值
string str("1111111000000011001101");
bitset<32> bitvec5(str, 5, 4); //从str[5]开始的四个二进制位，1100
bitset<32> bitvec6(str, str.size() - 4); //使用最后四个字符
cout << bitvec6.to_string() << endl;

bitset<32> bitvec7(0b1010100); //第2,4,6bit为1，其余为0-
```
## bitset操作
to_ulong和to_ullong操作都返回一个值，保存了与bitset对象相同的位模式。只有当bitset的大小小于等于对应的大小时，我们才能使用这两个操作。
如果bitset中的值不能放入指定的类型中，则这两个操作会抛出一个std::overflow_error异常。
输入运算符从一个输入流读取字符，保存到一个临时string对象中。直到读取的字符数达到对应bitset的大小或遇到不是1或0的字符时，或遇到文件尾或输入错误时，读取停止。随后使用临时的string对象来初始化bitset。
输出运算符打印bitset对象中的位模式：
``` cpp
bitset<16> bits;
cin >> bits; //从cin读取最多16个0或1
cout << "bits: " << bits << endl; //打印刚刚读入的内容
```

# 正则表达式
正则表达式(regular expression, RE)是一种描述字符串序列的方法，是一种强大的计算工具。RE库定义在头文件regex中，它包含多个组件。
## 使用正则表达式库
``` cpp
//查找不在字符c之后的字符ei
string pattern("[^c]ei");
//pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
pattern = "[a-zA-Z]*" + pattern + "[a-zA-Z]*";
regex r(pattern); //构造用于查找模式的regex
smatch results;   //定义保存搜索结果的对象
//待搜索的文本
string test_str = "receipt freind theif receive";
//用r在test_str中查找与pattern匹配的
if(regex_search(test_str, results, r))
{
    cout << results.str() << endl; //输出freind
}

//匹配一个或多个字母或数字字符后接一个‘.’，再接cpp或cxx或cc的文件名
regex r("[[:alnum:]]+\\.(cpp|cxx|cc)$", regex::icase); //regex::icase忽略大小写
smatch results;
string filename = "hello.cxx word.CC";
if(regex_search(filename, results, r))
{
    cout << results.str() << endl; //输出word.CC 因为$表示匹配行尾的单词
}
```
正则表达式是在运行时，当一个regex对象被初始化或被赋予一个新模式时，才被“编译的”。正则表达式的编译是一个非常慢的过程。
一个正则表达式的语法是否正确是在运行时解析的。
如果我们编写的正则表达式存在错误，则在运行时标准库会抛出一个类型为regex_error的异常。regex_error的what操作描述发生了什么错误，code成员用来返回某个错误类型对应的数值编码。
因为构造一个regex对象以及向一个已存在的regex赋予一个新的正则表达式可能非常耗时，为了最小化这种开销，应该努力避免创建不必要的regex。例如，如果在一个循环中使用了正则表达式，应该在循环外创建它，而不是在每步迭代时都编译它。
``` cpp
try
{
     regex r("[[:alnum:]+\\.(cpp|cxx|cc)$", regex::icase); //regex::icase忽略大小写
         smatch results;
    string filename = "hello.cxx word.CC";
    if(regex_search(filename, results, r))
    {
        cout << results.str() << endl;
    }
}
catch(regex_error e)
{
    cout << e.what() << "\ncode: " << e.code() << endl;
}
```
[C++ regex 正则表达式的使用](http://blog.csdn.net/mycwq/article/details/18838151)

## 匹配与Regex迭代器类型
``` cpp
string pattern = string("[[:alnum:]]+\\.(cpp|cxx|cc)");
regex r(pattern, regex::icase); //regex::icase忽略大小写
string file = "hello.cxx word.cc hi.cpp hello.txt m.CPP 1.log 1.CXX";
//sregex_iterator的构造函数调用regex_search(b, e, r)将it定位到输入中第一个匹配的位置
//end_it是一个空的sregex_iterator，起到尾后迭代器的作用
//该循环不断从一个匹配位置跳到下一个匹配位置
for(sregex_iterator it(file.begin(), file.end(), r), end_it; it != end_it; ++it)
{
    cout << it->str() << endl;
}
```
匹配类型smatch包含prefix和suffix两个成员，返回ssub_match对象，分别表示当前匹配之前和匹配之后的序列。ssub_match对象包含str和length的成员，分别返回匹配的string和该string的大小。
在输出匹配单词的同时打印上下文：
``` cpp
//查找不在字符c之后的字符ei
string pattern("[^c]ei");
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
regex r(pattern); //构造用于查找模式的regex
//待搜索的文本
string test_str = "receipt freind hello theif receive";
//用r在test_str中查找与pattern匹配的    
for(sregex_iterator it(test_str.begin(), test_str.end(), r), end_it; it != end_it; ++it)
{
    auto pos = it->prefix().length(); //前缀的大小
    pos = pos > 40 ? pos - 40 : 0;    //我们想要最多40个字符
    cout << it->prefix().str().substr(pos) //前缀的最后一部分
         << "\n\t\t>>> " << it->str() << " <<<\n" //匹配的单词
         << it->suffix().str().substr(0, 40) //后缀的第一个部分
         << endl;
}
```

## 使用子表达式
正则表达式中的模式通常包含一个或多个子表达式。一个子表达式是模式的一部分，本身也具有意义。正则表达式语法通常用括号表示子表达式。
将模式中点之前表示文件名的部分也形成子表达式，如下所示：
``` cpp
string pattern = string("([[:alnum:]]+)\\.(cpp|cxx|cc)");
//r包含两个子表达式：第一个是点之前的文件名部分，第二个是点之后的扩展名部分
regex r(pattern, regex::icase); //regex::icase忽略大小写
string file = "hello.cxx word.cc hi.cpp hello.txt m.CPP 1.log 1.CXX";
for(sregex_iterator it(file.begin(), file.end(), r), end_it; it != end_it; ++it)
{
    cout <<  "full name [" << it->str(0) << "], file name [" << it->str(1) << "], file type [" << it->str(2) << "]" << endl;
}
```
### 子表达式用于数据验证
电话号码校验的正则表达式解析：
``` cpp
string phone = "(\\()?(\\d{3})(\\))?([-. ])?(\\d{3})([-. ]?)(\\d{4})";
整个正则表达式包含七个子表达式：
1 (\\()?   表示可选的左括号
2 (\\d{3}) 表示区号
3 (\\))?   表示可选的右括号
4 ([-. ])? 表示区号部分可选的分隔符
5 (\\d{3}) 表示号码的下三位数字
6 ([-. ])? 表示可选的分隔符
7 (\\d{4}) 表示号码最后四位数字
```

## 使用regex_replace
当我们希望在输入序列中查找并替换一个正则表达式时，可以使用regex_replace。在电话号码匹配中，如果我们希望替换字符串中使用第二个，第五个和第七个子表达式。而忽略其他的子表达式，我们用一个符号`$`后跟子表达式的索引号来表示一个特定的子表达式：
``` cpp
string phone = "(\\()?(\\d{3})(\\))?([-. ])?(\\d{3})([-. ]?)(\\d{4})";
string fmt = "$2.$5.$7"; //将号码格式修改为ddd.ddd.dddd
regex r(phone);
string number = "morgan (908) 555-1800";
cout << regex_replace(number, r, fmt) << endl; //morgan 908.555.1800
//默认情况下，regex_replace输出整个输入序列，使用format_no_copy匹配标志从而不输出序列中未匹配的部分
using namespace std::regex_constants;
cout << regex_replace(number, r, fmt, format_no_copy) << endl; //908.555.1800
```

# 随机数
在新标准之前，C和C++都依赖于一个简单的C库函数rand来生成随机数。但存在如下问题：不能指定随机数范围以及随机数的概率分布特性。
在C++11新标准中，定义在头文件random中的随机数库通过一组协作类来解决问题：随机数引擎类和随机数分布类。随机数引擎类可以生成unsigned随机数序列，一个分布类使用一个引擎类生成指定类型的，给定范围的，服从特定概率分布的随机数。
<strong>C++程序不应该使用库函数rand，而应使用default_random_engine类和恰当的分布类对象。</strong>
## 随机数引擎和分布
随机数发生器特性：对一个给定的发生器，每次运行程序它都会返回相同的数值序列。序列不变这一事实在调试时非常有用，但另一方面，使用随机数发生器的程序必须考虑这一特性。
假设我们需要一个函数生成一个vector，包含100个均匀分布在0到9之间的随机数：
``` cpp
//几乎肯定是生成随机整数vector的错误方法
//每次调用这个函数都会生成相同的100个数！
vector<unsigned> bad_randVec()
{
    default_random_engine e;
    uniform_int_distribution<unsigned> u(0, 9);
    vector<unsigned> ret;
    for(size_t i = 0; i < 100; ++i)
    {
        ret.push_back(u(e));
    }
    return ret;
}
```
一个函数如果定义类局部的随机数发生器，应该将其(包括引擎和分布对象)定义为static的，从而在函数调用之间会保持住状态。否则，每次调用函数都会生成相同的序列。
``` cpp
vector<unsigned> bad_randVec()
{
    //定义为static的，保持引擎和分布对象的状态，从而每次调用生成新的数
    static default_random_engine e;
    static uniform_int_distribution<unsigned> u(0, 9);
    vector<unsigned> ret;
    for(size_t i = 0; i < 100; ++i)
    {
        ret.push_back(u(e));
    }
    return ret;
}
```
随机数发生器会生成相同的随机数这一特性在调试时很有用。当我们调试完毕，我们希望每次运行程序都会生成不同的随机结果，可以通过提供种子来达到这一目的。
为引擎设置种子有两种方式：创建引擎对象时提供种子，或者调用引擎的seed成员。
``` cpp
default_random_engine e1; //使用默认种子
default_random_engine e2(214789562); //使用给定的种子值

//e3和e4将生成相同的序列，因为它们使用了相同的种子
default_random_engine e3;
e3.seed(32767); //调用seed设置一个新种子值
default_random_engine e4(32767);

default_random_engine es(time(0)); //适用于生成间隔为秒级或更长的应用
```

## 其他随机数分布
``` cpp
default_random_engine e;
//0到1(包含)的均匀分布
uniform_real_distribution<double> u(0, 1);
for(size_t i = 0; i < 10; ++i)
{
    cout << u(e) << " ";
}
//空<>表示我们希望使用默认结果类型
uniform_real_distribution<> u(0, 1); //默认生成double值

//生成均值为4，标准差为1.5的正态分布
normal_distribution<> n(4, 1.5);

```

### bernoulli_distribution类
bernoulli_distribution是一个普通的类，该分布总是返回一个bool值，返回true的概率是一个常数，次概率的默认值是0.5。
决定用户和程序先行的游戏：
``` cpp
string resp;
default_random_engine e; //e应该保持状态，所以必须在循环体外定义
bernoulli_distribution b; //默认是50/50的机会
do {
    bool first = b(e); //如果为true，则程序先行
} while(cin >> resp && resp[0] == 'y');

bernoulli_distribution b(0.55); //则程序有55/45的机会先行
```

# IO库再探
## 格式化输入与输出
标准库定义类一组操纵符来修改流的格式状态。
操作符也返回它所处理的流对象，因此我们可以在一条语句中组合操纵符和数据。
大多数改变格式状态的操纵符都是设置/复原成对的：一个用来设置新值，一个用来恢复为默认格式。
整型类型常用的格式操纵符：
>使用`boolalpha`和`noboolalpha`操纵符控制布尔值的格式
使用`hex oct dec`将整型值输出修改为十六进制、八进制或是改回十进制
使用`showbase`和`noshowbase`显式与关闭进制的前导字符
使用`uppercase`和`nouppercase`打开或关闭进制前导字符或数字中a-f大小写

浮点数常用操纵符：
>使用`setprecision`或IO对象的`precision`成员控制打印精度
使用`scientific`表示科学记数法
使用`fixed`表示定点十进制
使用`hexfloat`强制浮点数使用十六进制格式，C++11新增
使用`defaultfloat`将流回复到默认状态
使用`showpoint`强制打印小数点，默认情况下，浮点数的小数部分为0时，不显示小数点

setprecision和其他接受参数的操纵符都定义在头文件`iomainip`中。
除非你需要控制浮点数的表示形式（如按列打印数据或打印表示金额或百分比的数据），否则由标准库选择记数法是最好的方式。
在执行scientific、fixed或hexfloat后，精度控制的是小数点后面的数字位数，而默认情况下精度指定的是数字的总位数。

当按列打印数据时，需要非常精细控制数据格式。标准库提供了一些操纵符完成控制：
>setw  指定下一个数字或字符串值的最小空间
left  表示左对齐输出
right 表示右对齐输出，默认格式
internal 控制负数的符号位置，左对齐括号，右对齐值，用空格填满中间空间
setfill  使用指定字符代替默认的空格补白输出

setw类似endl，不改变输出流的内部状态。它只决定下一个输出的大小。
默认情况下，输入运算符会忽略空白符(空格符、制表符、换行符、换纸符和回车符)。操纵符`noskipws`会令输入运算符读取空白符而不是跳过它们。使用`skipws`操纵符恢复默认行为。

## 未格式化的输入/输出操作
之前我们使用的都是格式化的IO操作。输入和输出运算符根据读写的数据类型来格式化它们。
标准库还提供一组低层操作，支持未格式IO。这些操作允许我们将一个流当作一个无解释的字节序列来处理。
头文件`cstdio`定义了一个名为EOF的const，我们可以使用它来检测从get返回的值是否是文件尾，而不必记忆表示文件尾的实际数值。
``` cpp
int ch; //使用int而不是char来保存get()的返回值
//循环读取并输出读入的字符
while ((ch = cin.get()) != EOF)
{
    cout.put(ch);
}
```
## 流随机访问
由于istream和ostream类型通常都不支持随机访问，所以随机访问只适用于fstream和sstream类型。
标准库提供`tell和seek`两组函数支持随机访问：seek将流设置到指定的位置；tell标记当前位置。输入和输出的版本的差别在于名字的后缀是g还是p。g版本表示我们正在“获得”(读取)数据，而p版本表示我们正在“放置”(写入)数据。
虽然标准库区分seek和tell的“放置”和“获得”版本，但同时可以读写的流只维护一个标记，并不存在独立的读标记和写标记。因此只要我们在读写操作间切换，就必须进行seek操作来重定位标记。
在文件的末尾写入新的一行，表示文件中每行的相对起始位置：
``` cpp
#include <fstream>
#include <cstdlib>
#include <iostream>

using namespace std;

int main()
{
    //以读写方式打开文件，并定位到文件尾
    fstream inOut("seek.in", fstream::ate | fstream::in | fstream::out);
    if(!inOut)
    {
        cerr << "Unable open file!" << endl;
        return EXIT_FAILURE;
    }
    //以ate模式打开inOut，因此一开始就定位到文件末尾
    auto end_mark = inOut.tellg();  //记住原文件尾的位置
    inOut.seekg(0, fstream::beg);   //重新定位到文件开始处
    size_t cnt = 0;
    string line;
    while(inOut && (inOut.tellg() != end_mark) && getline(inOut, line))
    {
        cnt += line.size() + 1; //+1表示统计换行符
        auto read_mark = inOut.tellg(); //记住读取位置
        inOut.seekp(0, fstream::end);   //将写标记移动到文件尾
        inOut << cnt;                   //输出累计长度
        //如果不是最后一行，打印分隔符
        if(read_mark != end_mark)
        {
            inOut << " ";
        }
        inOut.seekg(read_mark);         //恢复读位置
    }
    inOut.seekp(0, fstream::end);       //定位到文件尾
    inOut << "\n";                      //输出换行符
    inOut.close();
    return EXIT_SUCCESS;
}
```
