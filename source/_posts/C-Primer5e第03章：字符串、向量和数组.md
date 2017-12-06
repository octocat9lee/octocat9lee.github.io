---
title: C++Primer5e第03章：字符串、向量和数组
tags:
  - C++
toc: true
date: 2017-12-06 09:25:58
---
# 命名空间的using声明
<strong>头文件不应包含using声明</strong>
因为头文件的内容会拷贝到所有引用它的文件中去，如果头文件里有某个using声明，那么每个使用了该头文件的文件都会有这个声明。对于某些程序来说，由于不经意间包含了一些名字，可能产生意想不到的冲突。

# 标准库类型string
如果使用等号(=)初始化一个变量，实际上执行的是`拷贝初始化(copy initialization)`，因为编译器创建临时对象，然后会把等号右侧的初始值拷贝到新创建的对象中去。如果不使用等号，则执行的是`直接初始化(direct initialization)`。

## string::size_type类型
string的size函数返回的是一个string::size_type类型，它是一个无符号类型。切记：如果表达式中已经有size()函数就不要再使用int了，从而可以避免混用int和unsigned可能带来的问题。例如：假设n是一个负值的int，则表达式`s.size() < n`的判断结果几乎肯定是true，因为负值n会转换成一个很大的无符号值。
在C++11新标准中，可以通过auto或decltype来判断变量类型：
``` cpp
auto len = s.size(); //len的类型是string::size_type
```

## 处理string对象中的字符
使用C++版本的C标准库头文件，C++标准兼容C语言标准库。C语言的头文件刑如name.h，C++则将文件命名为cname。在名为cname的头文件中定义的名字从属于std命名空间，而定义在名为.h的头文件中则不然。一般来说，C++程序应该使用名为cname的头文件，标准库的名字总能在std中找到。
<!--more-->
### 基于范围for语句
<strong>范围for语句体内不应该改变其所遍历的序列的大小</strong>
``` cpp
string str("Some string");
for(auto c : str)  //编译器推断c的类型为char
{
    cout << c << endl;
}

for(auto &c : str) //对于s中的每个字符，注意c是引用
{
    c = toupper(c); //c为引用，因此赋值语句将改变s中字符的值
}

const string s = "Keep out!";
for(auto &c : s)   //编译器推断c是常量引用，因此c所绑定的对象值不能改变
{
    c = 'x'; //error:assignment of read-only reference 'c'
}
```
C++标准并不要求标准库检测下标是否合法，一旦使用了超出范围的下标，就会产生不可预知的结果。`最佳实践方法是设下标的类型为string::size_type，并保证下标小于size()值`。确保下标合法的一种有效手段是尽可能使用范围for语句。

# 标准库类型vector
C++11新标准可以使用列表初始化的方式初始化vector对象。
``` cpp
vector<string> articles = {"a", "an", "the"};
```
<strong>vector对象能高效增长</strong>
C++标准要求vector应该能在运行时高效快速地添加元素，那么在定义vector对象时设定其大小就没有必要了，甚至性能可能更差。

# 迭代器
如果容器为空，则begin和end返回的是同一个迭代器，都是尾后迭代器(指向尾元素的下一位置，不存在的元素)。
所有标准库容器的迭代器都定义了==和!=，但是它们中的大多数都没有定义<运算符，因此，我们使用!=从而不用在意容器类型。
## begin和end运算符
begin和end返回的具体类型由对象是否是常量决定，如果对象是常量，begin和end返回const_iterator；如果对象不是常量，则返回iterator；C++11新标准引入cbegin和cend，不论对象是否为常量，返回值都是返回值都是const_iterator。
``` cpp
vector<int> v;
const vector<int> cv;
auto it1 = v.begin();  //it1类型是vector<int>::iterator
auto it2 = cv.begin(); //it2类型是vector<int>::const_iterator
auto it3 = v.cbegin(); //it3类型是vector<int>::const_iterator
```
<strong>所有使用了迭代器的循环，都不要向迭代器所属的容器添加元素</strong>

## 迭代器运算
迭代器和一个整数相加或相减，其返回值是向前或向后移动若干位置的迭代器。移动后的迭代器或指示原容器内的一个元素，或者容器尾元素的下一个位置。
迭代器还可以使用关系运算符(<、<=、>、>=)进行比较，参与比较的迭代器必须指向同一个容器的元素。
当两个迭代器指向同一个容器元素时，还可以将其相减，结果就是两个迭代器的距离。类型为`difference_type`的有符号整数。获取容器中间元素迭代器的两种方式：
``` cpp
auto mid = v.begin() + v.size() / 2;
auto mid = v.begin() + (v.end() - v.begin()) / 2;
```
C++并没有定义两个迭代器的加法运算，因为在实际中，将两个迭代器加起来是没有意义的。

# 数组
数组与vector的相似之处在于都是存放相同类型的对象，并且对象本身没有名字，需要通过位置访问。但数组与vector的最大不同在于数组大小固定不变，不能随意添加元素。

## 理解复杂的数组声明
数组的元素应该为对象，但引用不是对象，所以不存在引用的数组。
理解复杂数组声明的含义，办法就是从数组的名字开始，按照由内向外的顺序阅读。
``` cpp
int *ptrs[10];   //因为下标优先级高于*，所以ptr是包含10个整型指针的数组
int &refs[10];   //error：不存在引用的数组
int (*Parray)[10] = &arr; //Parray是一个指向包含10个整数的数组的指针
int (&arrRef)[10] = arr;  //arrRef引用一个包含10个整数的数组
int *(&arry)[10] = ptrs;  //arry是一个含有10个int指针的数组的引用
```
在使用数组下标时，通常将其定义为`size_t`类型，一种与机器相关的无符号类型。在cstddef头文件中定义了size_t类型，该文件是C标准库stddef.h对应的C++语言版本。

## 指针和数组
在很多使用数组名字的地方，编译器会自动地将其替换为一个指向数组首元素的指针：
``` cpp
int ia[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}; //ia是一个包含10个元素的数组
auto ia2(ia);  //ia2是一个整型指针，指向ia的第一个元素
auto ia2(&ia[0]]); //编译器执行类似过程，所以ia2是一个整型指针
ia2 = 42;  //错误，不能用int值给指针赋值
```
但当使用decltype关键字时，上述的转换不会发生，decltype(ia)返回的类型是由10个整数构成的数组。

### 指针也是迭代器
为了让指针的使用更简单，C++11新标准引入了begin和end函数，将数组作为参数，begin函数返回首元素的指针，end函数返回尾元素下一个位置的指针。这两个函数定义在iterator头文件中。
``` cpp
int ia[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}; //ia是一个包含10个元素的数组
int *beg = begin(ia); //指向ia首元素的指针
int *end = end(ia);  //指向ia尾元素下一个位置的指针
```
两个指针相减的结果为ptrdiff_t类型，和size_t一样，也是一种带符号的类型。

### 使用数组初始化vector对象
使用数组初始化vector对象，只需指明拷贝区域的首元素地址和尾后元素地址就可以了：
``` cpp
int ia[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}; //ia是一个包含10个元素的数组
vector<int> ivec(begin(ia), end(ia)); //ivec有10个元素，分别为ia对应的副本
vector<int> subVec(ia + 1, ia + 4); //拷贝ia[1], ia[2], ia[3] 3个元素
```
现代的C++程序应当尽量使用vector和迭代器，避免使用内置数组和指针；应该尽量使用string，避免使用C风格的基于数组的字符串。

# 多维数组
多维数组其实是数组的数组。同样，按照由内向外的顺序阅读多维数组的定义。
``` cpp
int ia[3][4]; //定义一个大小为3的数组，该数组的每个元素都是含有4个整数的数组
```

## 多维数组下标引用
如果表达式下标运算符数量比数组维度小，则表示的结果将是给定索引处的一个内层数组：
``` cpp
//定义一个大小为3的数组，该数组的每个元素都是含有4个整数的数组
int ia[3][4] = {{1, 2, 3, 4}, {5, 6, 7, 8}, {9, 10, 11, 12}};
int (&row)[4] = ia[1]; //把row绑定到ia的第2个4元素的数组上，row是一个含有4个整数的数组的引用
int (*prow)[4] = &ia[1]; //将prow指向ia的第2个4元素的数组的首地址
for(int i = 0; i < 4; ++i)
{
    row[i] = 100;
    (*prow)[i] = 100; //对prow解引用后获得一个4个元素的数组
}
```

## 使用范围for语句处理多维数组
``` cpp
size_t cnt = 0;
int ia[3][4] = {0};
for(auto &row : ia)
{
    for(auto &col : row)
    {
        col = cnt;
        ++cnt;
    }
}
```
第一个for循环遍历ia的所有元素，元素是大小为4的数组，因此row的类型是含有4个整数的数组的引用；第二个for循环遍历4元素数组中的某一个，因此col类型是整数的引用。
在上述示例中，因为要改变数组元素的值，所以我们使用引用类型作为循环控制变量。但其实深层次的原因是如果row不是引用类型，编译器初始化row时会将数组形式的元素转换成指向数组首元素的指针，这样row的类型就是int*类型，那么第二个for循环就不合法了。所以`使用范围for语句处理多维数组，除最内层的循环外，其他所有循环的控制变量都应该是引用类型`。
``` cpp
for(auto &row : ia)
{
    for(auto col : row)
    {
        cout << col << endl;
    }
}
```

## 指针和多维数组
因为多维数组实际上是数组的数组，所以由多维数组名转换得来的指针实际上是指向第一个内层数组的指针：
``` cpp
int ia[3][4]; //定义一个大小为3的数组，该数组的每个元素都是含有4个整数的数组
int (*p)[4] = ia; //p指向包含4个整数的数组
p = &ia[2]; //p指向ia的第3个4元素的数组的首地址
```
在C++11新标准中，使用auto或者decltype就能尽可能避免在数组前面加上一个指针类型了：
``` cpp
for(auto p = ia; p != ia + 3; ++p)
{//p指向包含4个整数的数组
    for(auto q = *p; q != *p + 4; ++q)
    {//q指向4元素数组的首地址，也就是说q指向一个整数
        cout << *q << endl;
    }
}
//使用标准库的begin和end函数
for(auto p = begin(ia); p != end(ia); ++p)
{
    for(auto q = begin(*p); q != end(*p); ++q)
    {
        cout << *q << endl;
    }
}
//使用范围for语句
for(auto &p : ia)
{
    for(auto q : p)
    {
        cout << q << endl;
    }
}
```

## 类型别名简化多维数组指针
如下代码将“4个整数组成的数组”命名为int_array，用类型别名int_array定义外层循环变量让程序更易理解。
``` cpp
using int_array = int[4];
//typedef int int_array[4];
for(int_array *p = ia; p != ia + 3; ++p)
{
    for(auto q = *p; q != *p + 4; ++q)
    {
        cout << *q << endl;
    }
}
```
