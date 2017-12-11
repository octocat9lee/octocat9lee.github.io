---
title: C++Primer5e第08章：IO库
tags:
  - C++
toc: true
date: 2017-12-11 10:37:44
---
# IO类
标准库定义的IO类型包括基础istream，ostream，iostream类型，ifstream，ofstream，fstream读写文件类型以及istringstream，ostringstream，stringstream读写内存string对象的类型。
另外，为了支持使用宽字符，标准库还定义了一组wchar_t类型的数据。
宽字符版本的类型和函数的名字以一个w开始。例如：wiostream，wfstream，wstringstream，wcin，wcout，wcerr。

## IO类型间的关系
继承机制让我们可以将派生类对象当作基类对象来使用。类型ifstream和istringstream都继续自istream。因此，我们可以在传递istream对象的地方传递ifstream和istringstream。
例如：对ifstream和istringstream对象调用getline，也可以用>>从ifstream和istringstream读取数据。类的ofstream和ostringstream也都继承自ostream。

## IO对象无拷贝和赋值
不能拷贝IO对象，因此不能将形参和返回类型设置为流类型。所以，进行IO操作的函数通常以引用方式传递和返回流，并且读写一个IO对象会改变其状态，因此传递和返回的引用不能是const的。

## 条件状态
IO类定义了一些函数和标志，帮助我们访问和控制流的状态条件。IO标准库内部条件状态：
``` cpp
strm::iostate  由各个iostream 类定义，用于定义条件状态
strm::badbit   strm::iostate类型的值，用于指出被破坏的流
strm::failbit  strm::iostate类型的值，用于指出失败的流
strm::eofbit   strm::iostate类型的值，用于指出流已经到达文件的结束符
strm::goodbit　strm::iostate类型的值，流未出现错误状态，此值保证为0
```

IO标准库条件状态获取以及设置函数：
``` cpp
s.eof()           若流s的eofbit置位，则返回true
s.fail()          若流s的failbit或badbit置位，则返回true
s.bad()           若流s的badbit置位，则返回true
s.good()          若流s处于有效状态，则返回true
s.clear()         将流s的所有条件状态位复位，将流的状态设置为有效
s.clear(flag)     根据指定的flag，将流s中对应条件状态复位。flag类型是strm::iostate
s.setstate(flag)  根据指定的flag，将流s中对应条件状态置位
s.rdstate()       返回流的当前条件状态，返回值类型是strm::iostate
```

使用位操作按需复位生成新的状态，例如将failbit和badbit复位，但保持eofbit不变：
``` cpp
cin.clear(cin.rdstate() & ~cin.failbit & ~cin.badbit)
```
<!--more-->
## 管理输出缓冲
因为IO设备写操作耗时，所以操作系统会将多个输出组合为单一的设备写操作从而提升性能。每个输出流都管理一个缓冲区，用来保存程序读写的数据。导致缓冲刷新的原因：
>1）程序正常结束，缓冲刷新被执行
2）缓冲区满时，需要刷新缓冲
3）使用操纵符endl、flush和ends来显式刷新缓冲区
4）可以用操纵符unitbuf设置流的内部状态，来清空缓冲区。默认情况下，cerr是设置unitbuf的
5）一个输出流被关联到另一个流。在这种情况下，当读写被关联的流时，关联到的流的缓冲区会被刷新，cin和cerr都关联到cout。因此读cin或写cerr会导致cout的缓冲区被刷新

<strong>如要程序异常终止，输出缓冲区是不会被刷新的。当一个程序崩溃后，它所输出的数据很可能停留在输出缓冲区中等待打印</strong>

# 文件输入和输出
创建一个文件流对象时，我们可以提供文件名，也可不提供文件名，后面用open成员函数来打开文件。
``` cpp
ifstream in(infile);  //构造一个ifstream并打开文件
ofstream out;         //输出流并未关联到任何文件
out.open(outfile);    //用open打开文件
```
如果调用open失败，failbit会被置位，所以调用open时进行检测通常是一个好习惯。
如果用读文件流ifstream去打开一个不存在的文件，将导致读取失败；而如果用写文件流ofstream去打开文件，如果文件不存在，则会创建这个文件。
一旦一个文件流已经被打开，它就保持与对应文件的关联。对一个已经打开的文件流调用open会失败，并会导致failbit被置位，随后的试图使用文件流的操作都会失败。为了将文件流关联到另外一个文件，必须首先关闭已经关联的文件。
<strong>当一个fstream对象被销毁时，close会自动被调用</strong>

## 文件模式
每个文件流都有一个关联的文件模式。文件模式名称位于std::ofstream类型中，具体如下：
``` cpp
in     以读方式打开
out    以写方式打开
app    每次写操作均定位到文件末尾，append
ate    打开文件后立即定位到文件末尾，at the end
trunc  截断文件
binary 以二进制方式进行IO
```
保留被ofstream打开的文件中已有数据的唯一方法是显式指定app或in模式

# string流
string流特有的操作：
``` cpp
sstream strm;    strm是一个未绑定的stringstream对象。sstream是头文件sstream中定义的一个类型
sstream strm(s); strm是一个stringstream对象，保存string s的拷贝。此构造函数是explicit
strm.str();      返回strm保存的字符串
strm.str(s);     将strm保存的字符串拷贝到string s中
```
## 使用istringstream
istringstream将string转换为int：
``` cpp
int string2int(string str)
{
    int ia = 0;
    istringstream istr(str);
    istr >> ia;
    return ia;
}
```
读取文件中的人名和电话号码记录：
``` cpp
struct PersonInfo{
    string name;
    vector<string> phones;
};

string line, word;
vector<PersonInfo> people;
while(getline(cin, line))
{
    PersonInfo info;
    istringstream record(line);
    record >> info.name;
    while(record >> word)
    {
        info.phones.push_back(word);
    }
    people.push_back(info);
}
```
