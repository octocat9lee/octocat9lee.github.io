---
title: C++Primer5e第09章：顺序容器
tags:
  - C++
toc: true
date: 2017-12-11 13:11:24
---
# 顺序容器概述
顺序容器中顺序不依赖元素的值，而与元素加入容器时的位置相对应。所有顺序容器都提供快速顺序访问元素的能力，但在以下方面有不同的性能折中：
>向容器中添加和删除元素的代价
非顺序访问容器中元素的代价

## 顺序容器类型
``` cpp
vector        可变大小数组。支持快速随机访问。在尾部之外的位置插入或删除元素可能很慢
deque         双端队列。支持快速随机访问。在头尾位置插入、删除速度都很快
list          双向链表。只支持双向顺序访问。在list中任何位置进行插入、删除操作速度都很快
forward_list  单向链表。只支持单向顺序访问。在链表任何位置进行插入、删除操作速度都很快
array         固定大小数组。支持快速随机访问。不能添加或删除元素
string        与vector相似的容器，但专门用于保存字符。随机访问快。在尾部插入、删除速度快
```
新标准库容器的性能几乎肯定与最精心优化过的同类数据结构一样好(通常会更好)。现代C++程序应该使用标准库容器，而不是更原始的数据结构，如内置数组。

## 选择容器的基本原则
>1）除非有很好的理由选择其他容器，否则应使用vector；
2）如果程序有很多小元素，且空间的额外开销很重要，则不要使用list或forward_list；
3）如果程序要求随机访问元素，应使用vector或deque；
4）如果程序要求在容器的中间插入或删除元素，应使用list或forward_list；
5）如果程序需要在头尾位置插入或删除元素，但不会在中间位置进行插入或删除操作，则使用deque；
6）如果程序只有在读取输入时才需要在容器中间位置插入元素，随后需要随机访问元素，则:
    - 首先，确定是否真的需要在容器中间位置添加元素。当处理输入数据时，通常可以很容易地向vector追加数据，然后在调用标准库的sort函数来重排序，从而避免在中间位置添加元素。
    - 如果必须在中间位置插入元素，考虑在输入阶段使用list，一旦输入完成，将list中的内容拷贝到一个vector中。

<strong>如果不确定应该使用哪种容器，那么在程序中使用vector和list公共操作：使用迭代器，不使用下标访问，避免随机访问。这样，在必要时选择使用vector和list都很方便。</strong>
<!--more-->
# 容器库概述
一般来说，每个容器都定义在一个头文件中，文件名与类型名相同。对于大多数容器，我们还需要额外提供元素类型信息。另外，我们还可以定义一个容器，其元素的类型是另一个容器。
## 容器操作
``` cpp
下列操作适用于所有的标准库容器

类型别名
iterator                    容器类型的迭代器类型
const_iterator              可以读取元素，但不能修改元素的迭代器类型
size_type                   无符号整数类型，足够保存容器类型最大可能容器的大小
difference_type             有符号整数类型，足够保存两个迭代器之间的距离
value_type                  元素类型
reference                   元素的左值类型：与value_type&含义相同
const_reference             元素的const左值类型，即const value_type&

构造函数
C c;                        默认构造函数，构造空容器
C c1(c2);                   构造c2的拷贝c1
C c(b, e);                  构造c，将迭代器b和e指定的范围内的元素拷贝到c(array不支持)
C c{a, b, c...};            列表初始化c

赋值与swap
c1 = c2                     将c1中的元素替换为c2中元素
c1 = {a, b, c...}           将c1中的元素替换为列表中元素(不适用于array)
a.swap(b)                   交换a和b的元素
swap(a, b)                  等价于a.swap(b)

大小
c.size()                    c中元素的数目(不支持forward_list)
c.max_size()                c可保存的最大元素数目
c.empty()                   若c中存储了元素，返回false，否则返回true

添加/删除元素(不适用于array)
注：在不同容器中，这些操作的接口都不同
c.insert(args)               将args中的元素拷贝进c
c.emplace(inits)             使用inits构造c中的一个元素
c.erase(args)                删除args指定的元素
c.clear()                    删除c中的所有元素

关系元素符
==, !=                       所有容器都支持相等，不相等运算符
<, <=, >, >=                 关联容器不支持

获取迭代器
c.begin(), c.end()           返回指向c的首元素和尾元素之后位置的迭代器
c.cbegin(), c.cend()         返回const_iterator

反向容器的额外成员(不支持forward_list)
reverse_iterator             按逆序寻址元素的迭代器
const_reverse_iterator       只读逆序迭代器
c.rbegin(), c.rend()         返回指向c的尾元素和首元素之前位置的迭代器
c.crbegin(), c.crend()       返回const_reverse_iterator
 ```

 ## 迭代器
 构成范围迭代器的要求：
 >begin和end必须指向同一个容器中的元素，或者容器尾元素之后的位置
 end不在begin之前

 标准库构成范围迭代器使用左闭合区间，因为如下性质：
 >如果begin和end相等，则范围为空
 如果begin和end不相等，则范围内至少包含一个元素
 我们可以对begin递增，使得begin==end

 vector和deque的元素在内存中连续保存，而list则是将元素以链表方式存储，所以在list中，两个指针的大小关系并不表示元素的前后关系。<strong>因此，list的迭代器不支持<运算，只支持递增、递减、==以及!=运算</strong>。
<strong>当不需要写元素时，应使用cbegin和cend。</strong>

## 标准库array
与内置数组一样，标准库array的大小也是类型的一部分。当定义一个array时，除了指定元素类型，还要指定容器大小：
``` cpp
array<int, 42>   //类型为：保存42个int的数组
//为了使用array类型，我们必须同时指定元素类型和大小
array<int, 42>::size_type i; //数组元素类型包括元素类型和大小
array<int>::size_type j;     //错误，array<int>不是一个类型
```
可以对array进行拷贝，但要求array元素类型和大小一致。
``` cpp
array<int, 10> digits = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
array<int, 10> copy = digits; //正确：只要数组类型匹配
```

## 赋值和swap
赋值相关运算会导致指向左边容器内部的迭代器、引用和指针失效。而swap操作将容器内容交换不会导致指向容器的迭代器、引用和指针失效(容器类型为string和array的情况除外)。
使用assign实现将-个vector中的一段`char *`值赋予给一个list中的string：
``` cpp
list<string> names;
vector<const char*> oldstyle;
names = oldstyle; //错误：容器类型不匹配
names.assign(oldstyle.cbegin(), oldstyle.cend());
```
由于其旧元素被替换，因此传递给assign的迭代器不能指向调用assign的容器。
除array外，交换两个容器内容的操作保证会很快，因为元素本身并未交换，swap只是交换了两个容器的内部数据结构，因此可以保证在常数时间内完成。
因为元素不会移动，因此除了string外，指向容器的迭代器、引用和指针在swap操作之后都不会失效。它们仍然指向swap操作之前所指向的那些元素。但是，在swap之后，这些元素已经属于不同的容器了。例如，`swap(svec1, svec2);`，在swap前，iter指向svec1[3]的string，那么swap后它指向svec2[3]的元素。
<strong>对一个string调用swap会导致迭代器、引用和指针失效</strong>
swap两个array会真正交换它们的元素，因此交换两个array所需时间与array中元素的数目成正比。对于array，在swap操作之后，指针、引用和迭代器绑定的元素保持不变，但元素的值已经与另个一个array中对应元素的值进行了交换。
<strong>在C++11新标准中，统一使用非成员版本的swap是一个好习惯。</strong>

## 关系运算
对于容器使用关系运算符的条件：
>1）容器类型相同，并且元素类型也必须相同
2）对应的元素类型定义了相应的比较运算符

<strong>容器的相等运算通过使用元素的==运算符实现比较，而其他关系运算是使用元素的<运算符。</strong>

# 顺序容器操作
## 向顺序容器添加元素
<strong>向vector、string或deque插入元素会导致迭代器、引用和指针失效</strong>
容器元素是拷贝，也就是说当我们把一个元素添加到容器中时，加入到容器中的是对象的拷贝，而不是对象本身。因此，对容器中元素的任何改变都不会影响到原始对象，反之亦然。
insert成员函数可以通过迭代器在指定位置放置新元素；还可以将指定数量的元素添加到指定位置之前。在C++11新标准中，insert返回指向第一个新加入元素的迭代器。
使用insert返回值循环在list的头部插入元素：
``` cpp
list<string> lst;
auto iter = lst.begin();
while(cin >> word)
{
    iter = lst.insert(lst, word); //等价于调用push_front
}
```

### 使用emplace操作
新标准引入三个新成员`emplace_front、emplace和emplace_back`，将元素放置在容器头部、指定位置或容器尾部。emplace成员使用这些参数在容器管理的内存空间中直接构造元素。因此传递给emplace函数参数必须与元素类型的构造函数相匹配。
``` cpp
//在c的末尾构造一个Sales_data对象
//使用三个参数的Sales_data构造函数
c.emplace_back("978-012345", 25, 13.99);
//错误：没有接受三个参数的push_back版本
c.push_back("978-012345", 25, 13.99);
//正确：创建一个临时的Sales_data对象传递给push_back
c.push_back(Sales_data("978-012345", 25, 13.99));
```

## 访问元素
如果容器中没有元素，则访问操作的结果是未定义的。每个顺序容器都有一个front成员函数，除forward_list之外所有的顺序容器都有一个back成员函数，这两个操作分别返回首元素和尾元素的引用。
<strong>访问成员函数返回的都是引用</strong>
如果我们希望确保下标是合法的，可以使用at成员函数。at会在下标越界时，抛出out_of_range异常。

## 删除元素
在删除元素之前，程序员必须确保它们是存在的。erase返回指向删除的(最后一个)元素之后位置的迭代器。假设j是i之后的元素，那么erase(i)将返回指向j的迭代器。下面代码删除list中所有奇数元素：
``` cpp
list<int> lst = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
auto it = lst.begin();
while(it != lst.end())
{
    if((*it % 2) != 0)
    {
        it = lst.erase(it);
    }
    else
    {
        ++it;
    }
}
```

## 特殊的forward_list操作
在单向链表中，删除或添加的元素之前的那个元素的后继会发生改变。在一个forward_list中添加或删除元素的操作是通过改变给定元素之后的元素来完成的。从forward_list中删除奇数元素：
``` cpp
forward_list<int> flst = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
auto prev = flst.before_begin(); //表示flst的“首前元素”
auto curr = flst.begin(); //表示flst中第一个元素
while(curr != flst.end())
{
    if((*curr % 2) != 0)
    {
        curr = flst.erase_after(prev); //erase_after返回被删除元素之后的元素的迭代器
    }
    else
    {
        prev = curr;  //prev指向curr之前的元素
        ++curr;   //移动迭代器curr指向下一个元素
    }
}
```

## 改变容器大小
如果在一个循环中插入或者删除deque、string或vector中的元素，不要缓存end返回的迭代器。因为这个原因，C++标准库中的end()操作都很快。

# vector对象是如何增长的
为了减少向vector和string频繁添加元素时，重新分配空间，拷贝，释放旧空间带来的性能开销。标准库实现者采用了可以减少容器空间重新分配次数的策略。当需要重新分配空间时，通常会分配比新的空间需求更大的内存空间，预留空间作为备用，用来保存更多的新元素。这样，就不需要每次添加新元素都重新分配容器的内存空间了。
## 管理容量的成员函数
``` cpp
shrink_to_fit只适用于vector、string和deque
capacity和reserve只适用于vector和string
c.shrink_to_fit()       请将capacity()减少为与size()相同大小，仅仅是一个请求，标准库并不保证退还内存
c.capacity()            不重新分配内存空间，c可以保存多少元素 capacity将会大于或等于传递给reserve的参数
c.reserve(n)            分配至少容纳n个元素的空间
```
只有在执行insert操作时size与capacity相等，或者调用resize或reserve时给定的大小超过当前capacity，vector才可能重新分配内存空间。
vector一般采用的分配策略是在每次需要分配新内存时将当前容量翻倍。总的原则：<strong>只有当迫不得已时才可以分配新的内存空间</strong>

# 额外的string操作
## substr操作
substr操作返回一个string的一部分或全部的拷贝，参数是一个可选的开始位置和计数值。如果开始位置超过了string大小，则substr函数抛出一个抛出out_of_range异常。如果开始位置加上计数值大于string的大小，则substr会调整计数值，只拷贝到string的末尾。
``` cpp
//使用vector<char>初始化string，vector提供data成员函数，返回其内存空间首地址
vector<char> vc = {'H', 'e', 'l', 'l', 'o'};
string s(vc.data(), vc.size());
```

## 修改string的操作
```
s.insert(pos, args)   在pos之前插入args指定的字符，pos可以是下标或迭代器，下标版本返回指向s的引用；迭代器版本返回指向第一个插入字符的迭代器
s.erase(pos, len)   删除从位置pos开始的len个字符，如果len省略，删除至末尾，返回指向s的引用
s.assign(args)    将s中的字符替换为args指定的字符，返回指向s的引用
s.append(args)    将args追加到s，返回一个指向s的引用
s.replace(range, args) 删除s范围range内的字符，替换为args所指定的字符。range可以是下标和一个长度，或者一对指向s的迭代器。返回指向s的引用
```

## string搜索操作
string搜索函数返回string::size_type值，表示匹配位置下标，该类型是一个unsigned类型。因此，使用int或其他有符号类型保存这些函数返回值不是一个好主意。如果搜索失败，返回一个名为string::npos的static成员。
``` cpp
搜索操作返回指定字符的下标，如果未找到则返回npos
s.find(args)                   查找s中args第一次出现的位置
s.rfind(args)                  查找s中args最后一次出现的位置
s.find_first_of(args)          在s中查找args中任何一个字符第一次出现的位置
s.find_last_of(args)           在s中查找args中任何一个字符最后一次出现的位置
s.find_fist_not_of(args)       在s中查找第一个不在args中的字符
s.find_last_not_of(args)       在s中查找最后一个不在args中的字符

args必须是以下形式之一
pos是string::size_type类型
c,pos     从s中位置pos开始查找字符c。pos默认为0
s2,pos    从s中位置pos开始查找字符串s2。pos默认为0
cp,pos    从s中位置pos开始查找指针cp指向的以空字符结尾的C风格字符串。pos默认为0
cp,pos,n  从s中位置pos开始查找指针cp指向的数组的前n个字符。pos和n无默认值
```

## compare函数
compare类似strcmp，根据s是等于、大于还是小于参数指定的字符串，s.compare返回0、正数或负数。

## 数值转换
string和数值之间的转换
``` cpp
to_string(val)        返回数值val的string表示。val可以是任何算术类型

stoi(s, p, b)         返回s的起始子串(表示整数内容)的数值，返回类型分别为
stol(s, p, b)         int、long、unsigned long、long long、unsigned long long。
stoul(s, p, b)        b表示转换用的基数，默认值为10。p是size_t指针，用来保存s中第一个
stoll(s, p, b)        非数值字符的下标，p默认值为0，即函数不保存下标。
stoull(s, p, b)

stof(s, p)            返回s的起始子串(表示浮点数内容)的数值，返回值类型分别是float、
stod(s, p)            double、long double。
stold(s, p)
```
string参数中第一个非空白字符必须是符号(+或-)或数字。对于将字符串转换成浮点的数，string参数也可以以小数点(.)开头，并可以包含e或E来表示指数部分。对于转换为整型值的函数，根据基数不同，string参数可以包含字母字符，对应大于数字9的数。如果string不能转换为一个数值，则函数抛出invalid_argument异常。如果转换得到的数值无法用任何类型表示，则抛出一个抛出out_of_range异常。
``` cpp
string s2 = "pi = 3.14";
//转换s中以数字开始的第一个子串，结果d=3.14
d = stod(s2.substr(s2.find_first_of("+-.0123456789")));
```

# 容器适配器
标准库定义三个顺序容器适配器：stack、queue和priority_queue。适配器使某种事物的行为看起来像另外一种事物一样。一个容器适配器接受一种已有的容器类型，使其行为看起来像一种不同的类型。
所有容器适配器都支持的操作和类型
``` cpp
size_type               一种类型，足以保存当前类型的最大对象的大小
value_type              元素类型
container_type          实现适配器的底层容器类型
A a;                    创建一个名为a的空适配器
A a(c);                 创建一个名为a的适配器，带有容器c的一个拷贝
关系运算(== != < <= > >=)返回底层容器的比较结果
a.empty()               若a包含元素，返回false；否则返回true
a.size()                返回a中元素数目
swap(a, b)              交换a和b的内容，a和b必须具有相同类型，包括底层容器类型也必须相同
a.swap(b)
```
每个容器适配器基于底层容器类型的操作定义了自己的特殊操作。我们只可以使用适配器操作，而不能使用底层容器类型的操作。
另外，我们可以在创建一个适配器时将一个命名的顺序容器作为第二个类型参数，来重载默认容器类型。
``` cpp
//在vector上实现的空栈
stack<string, vector<string>> str_stk;
```

## 栈适配器
stack适配器接受一个顺序容器(除array或forward_list外)，并使其操作起来像一个stack一样。
stack的额外操作：
``` cpp
栈默认基于deque实现，也可以在list或vector上实现
s.pop()            删除栈顶元素
s.push(item)       创建一个新元素压入栈顶，该元素通过拷贝或移动item而来，或者
s.emplace(args)    由args构造
s.top()            返回栈顶元素，但不将元素弹出
```

## 队列适配器
标准库queue使用一种FIFO的存储和访问策略。priority_queue允许我们为队列中的元素建立优先级。新加入的元素会排在所有优先级比它低的已有元素之前。默认情况下，标准库使用<运算符确定优先级。
``` cpp
queue默认基于deque实现，priority_queue默认基于vector实现
queue也可以list或vector实现，priority_queue也可以使用deque实现
q.pop()       返回queue首元素或priority_queue的最高优先级元素，但不删除此元素
q.front()     返回首元素或尾元素，但不删除此元素
q.back()      只适用于queue
q.top()       返回最高优先级元素，但不删除此元素，只适用于priority_queue
q.push(item)  在queue末尾或priority_queue中恰当位置创建一个元素，其值为item或者args构造
q.emplace(args)
```
