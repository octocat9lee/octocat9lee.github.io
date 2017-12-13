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

## 参数绑定
例如，在单词查找程序中，传递长度参数的函数check_size不能直接作为find_if的谓词，因为find_if的可调用对象必须接受单一参数。在前面的程序中，我们使用捕获列表的lambda表达式解决。另外，如果我们为了解决使用函数替换捕获列表的lambda的问题，必须解决如何向形参传递参数的问题？
``` cpp
bool check_size(const string &s, std::string::size_type sz)
{
    return s.size() >= sz;
}
```

### 标准库bind函数
在C++11新标准中，我们可以使用名为`bind`的标准库函数，它接受一个可调用对象，生成一个新的可调用对象来“适配”原对象的参数列表。bind函数定义在functional头文件中。调用bind的一般形式：
``` cpp
auto newCallable = bind(callable, arg_list);
```
其中，newCallable本身是一个可调用对象，arg_list是一个逗号分隔的参数列表，对应给定的callable的参数。也就是说当我们调用newCallable时，newCallable会调用callable，并传递给它arg_list中的参数。
arg_list中的参数可能包含形如`_n`的名字，n是一个整数，这些参数是“占位符”，表示newCallable的参数，它们占据了传递给newCallable的参数的位置。`_1表示newCallable的第一个参数，_2为第二个参数，以此类推`。
<strong>bind的具体使用实例：</strong>
``` cpp
//check6是一个可调用对象，接受一个string类型参数，并用此string和值6来调用check_size
//bind调用只有一个占位符，表示check6只接受单一参数，参数类型const string&
using namespace std::placeholders; //声明占位符的命名空间
auto check6 = bind(check_size, _1, 6);
string s = "hello";
bool b1 = check6(s); //check6(s)会调用check_size(s, 6)
//单词查找程序中lambda表达式替换成bind版本
auto wc = find_if(words.begin(), words.end(),
            bind(check_size, _1, sz));
```

<strong>可以使用bind重排参数顺序</strong>
``` cpp
using namespace std::placeholders;
//按单词长度由短至长排序，相当于比较A和B时，调用isShorter(A, B)
sort(words.begin(), words.end(), isShorter);
////按单词长度由长至短排序，相当于比较A和B时，调用isShorter(B, A)
sort(words.begin(), words.end(), bind(isShorter, _2, _1);
```

<strong>绑定引用参数</strong>
默认情况下，`bind的那些不是占位符的参数被拷贝到返回的可调用对象中`。但有时我们希望传递引用或者甚至绑定的参数无法拷贝，必须使用引用。例如，替换前面使用引用方式捕获ostream的lambda：
``` cpp
ostream &print(ostream &os, const string &s, char c)
{
    return os << s << c;
}
//错误：不能拷贝os
for_each(words.begin(), words.end(), bind(print, os, _1, ' '));
//使用标准库ref函数，生成给定对象的引用，cref生成const引用，均定义在functional头文件中
for_each(words.begin(), words.end(), bind(print, ref(os), _1, ' '));
```

# 再探迭代器
除了为每个容器定义的迭代器外，标准库在头文件iterator中还定义了额外几种迭代器，包括`插入迭代器、流迭代器、反向迭代器和移动迭代器`。

## 插入迭代器
插入迭代器接受一个容器，生成一个迭代器，能实现向给定容器添加元素。也就是说当我们通过一个插入迭代器赋值时，该迭代器调用容器操作，例如`c.push_back(t)、c.push_front(t)、c.insert(t,p)，p为传递给inserter的迭代器位置`来向给定容器插入一个元素。
``` cpp
//假设it是由inserter生成的迭代器，则下面的赋值语句：
*it = val;
//等价于
it = c.insert(it, val); //it指向新加入的元素
++it; //递增it使它指向原来的元素

list<int> lst = {1, 2, 3, 4};
list<int> lst2, lst3; //空list
//拷贝完成后，lst2包含4 3 2 1
copy(lst.cbegin(), lst.cend(), front_inserter(lst2));
//拷贝完成后，lst3包含1 2 3 4，因为inserter会导致迭代器更新，始终指向lst3.end()，即在末尾插入
copy(lst.cbegin(), lst.cend(), inserter(lst3, lst3.begin()));
```

## iostream迭代器
iostream迭代器将它们对应的流当作一个特定类型的元素序列来处理。通过使用流迭代器，我们可以使用泛型算法从流对象读取以及向其写入数据。流迭代器不支持递减运算，因为不可能在一个流中反向移动。

### istream_iterator操作
istream_iterator使用>>来读取流，因此istream_iterator要求读取的类型必须定义了输入运算符。当我们创建istream_iterator时，可以绑定到一个流，默认初始化istream_iterator创建一个当做尾后值的迭代器。
从标准输入流读入int整数，保存到vector中。对于一个绑定到流的迭代器，当关联的流遇到文件尾或错误时，迭代器的值就等于尾后迭代器。
``` cpp
istream_iterator<int> in_iter(cin); //从cin读取int
istream_iterator<int> end_iter; //尾后迭代器
vector<int> vec(in_iter, end_iter);
//或者
 vector<int> vec((istream_iterator<int>(cin)), istream_iterator<int>()); //(istream_iterator<int>(cin))括号不可省略
 vector<int> vec({istream_iterator<int>(cin)}, istream_iterator<int>()); //uniform initialization
cin.clear(); //清除流状态

//使用算法操作流迭代器
int sum = accumulate({istream_iterator<int>(cin)}, istream_iterator<int>(), 0);
```

### ostream_iterator操作
我们可以对任何具有输出运算符的类型定义ostream_iterator，可选的第二参数是一个C风格的字符串，在每次输出元素后都会打印此字符。必须将ostream_iterator绑定到一个指定的流，不允许空的或表示尾后位置的ostream_iterator。
``` cpp
ostream_iterator<int> out_iter(cout, " ");
for(auto e : vec)
{
    *out_iter++ = e; //运算符*和++实际上对ostream_iterator对象什么都不做，等价于 aut_iter = e;
}
cout << endl;
//调用copy打印vec中元素
copy(vec.cbegin(), vec.cend(), out_iter);
cout << endl;
```

## 反向迭代器
反向迭代器就是在容器中从尾元素向首元素反向移动的迭代器。对于反向迭代器，递增以及递减操作的含义会颠倒过来。`递增一个反向迭代器(++it)会移动到前一个；递减一个反向迭代器(--it)会移动到下一个`。除了forward_list之外，所有的容器都支持反向迭代器。
通过向sort传递一对反向的迭代器将vector整理为递减序：
``` cpp
sort(vec.begin(), vec.end()); //由小到大
sort(vec.rbegin(), vec.rend()); //由大到小
```
使用reverse_iterator的base成员函数获得普通的迭代器：
``` cpp
//在一个逗号分隔的列表中查找最后一个元素
auto rcomma = find(line.crbegin(), line.crend(), ',');
//错误：将逆序输出单词的字符
cout << string(line.crbegin(), rcomma) << endl;
//正确：得到一个正向迭代器，从逗号开始直到line末尾
cout << string(rcomma.base(), line.cend()) << endl;
```

# 泛型算法结构
C++标准指明了泛型和数值算法的每个迭代器参数的最小类别。
算法要求的迭代器操作可以分为5个类别：
>输入迭代器      只读，不写；单遍扫描，只能递增
输出迭代器       只写，不读；单遍扫描，只能递增
前向迭代器       可读写；多遍扫描，只能递增
双向迭代器       可读写：多遍扫描，可递增递减
随机访问迭代器   可读写；多遍扫描，支持全部迭代器运算

除了输出迭代器之外，一个高层次类别的迭代器支持低层次类别迭代器的所有操作。
向输出迭代器写入数据的算法都假定目标空间足够容纳写入的数据。

## 算法命名规范
>使用重载形式传递一个谓词
_if后缀版本的算法接受一个谓词
_copy后缀版本拷贝元素 copy是copy全部的元素，但可以根据条件执行替换等操作
_copy_if后缀版本 remove_copy_if

# 特定容器算法
`对于list和forward_list，应该优先使用成员函数版本的算法而不是通用算法`。因为链表可以通过改变元素间的链接而不是真的交换它们的值。因此，这些链表版本的算法性能比对应的通用版本好得多。
``` cpp
list和forward_list成员函数版本的算法
lst.merge(lst2)       将来自lst2的元素合并到lst。lst和lst2必须有序，元素将从lst2中删除。
lst.merge(lst2, comp) 在合并之后，lst2变为空。第一个版本使用<运算符；第二个版本使用给定的比较操作

lst.remove(val)       调用erase删除掉与给定值相等(==)或
lst.remove_if(pred)   令一元谓词为真的每个元素

lst.reverse()         反转lst中元素

lst.sort()            使用<或给定比较操作排序元素
lst.sort(comp)

lst.unique()          调用erase删除同一个值的连续拷贝。第一个版本使用==
lst.unique(pred)      第二个版本使用给定的二元谓词

lst.splice(p, lst2)    将lst2中的元素转移到当前容器中的指定位置之前，将元素从lst2中删除
lst.splice_after(p, lst2) 将lst2中的元素转移到当前容器中的指定位置之后，将元素从lst2中删除
```
<strong>链表特有的操作会改变容器</strong>，例如，remove版本会删除指定的元素，unique版本会删除第二个和后继的重复元素，merge和splice会销毁其参数。
