---
title: C++Primer5e第11章：关联容器
tags:
  - C++
toc: true
date: 2017-12-13 14:47:45
---
关联容器中的元素是按关键字来保存和访问的，支持高效的关键字查找和访问。标准库提供8个关联容器，体现在3个维度：
(1) set或者map；(2)关键字能否重复，允许重复关键字的包含单词multi；(3)按顺序保存或无序保存，不保持关键字按顺序存储的容器的名字都以单词unordered开头。因此，unordered_multi_set是一个允许关键字重复，元素无序保存的集合，而set则是一个要求关键字不重复，有序存储的集合。`set和map使用红黑树，无序容器使用哈希函数来组织元素。`
关联容器类型：
>按关键字有序保存元素
map
set
multimap
multiset
无序集合
unordered_map
unordered_set
unordered_multimap
unordered_multiset

# 使用关联容器
在忽略大小写和标点的单词计数程序中，首先是对输入的字符串进行处理，剔除单词中的标点，并将大写转换成小写。
``` cpp
string &trans(string &s)
{
    for(string::size_type p = 0; p < s.size(); ++p)
    {
        if(s[p] >= 'A' && s[p] <= 'Z')
        {
            s[p] -= ('A' - 'a');
        }
        else if(s[p] == ',' || sp[p] == '.')
        {
            s.erase(p, 1);
        }
    }
    return s;
}
```

# 关联容器概述
关联容器的迭代器都是双向的。在C++11新标准中，我们可以对关联容器进行值初始化：
``` cpp
map<string, size_t> word_count; //空容器
//列表初始化
set<string> exclude = {"the", "but", "and", "or", "an"};
//三个元素：authors将姓映射为名
map<string, string> authors = {{"Joyce", "James"}, {"Austen", "Jane"}, {"Dickens, "Charles"}};
```
<!--more-->
## 关键字类型要求
对于有序容器map、multimap、set以及multiset，关键字类型必须定义元素比较的方法。默认情况下，标准库使用关键字类型的<运算符来比较两个关键字。
传递给排序算法的可调用对象必须满足与关联容器中关键字一样的类型要求。
在实际编程中，如果一个类型定义了正常的<运算符，则它可以用作关键字类型。
使用自定义的操作作为关联容器类型的比较操作类型(函数指针类型个)：
``` cpp
bool comparseIsbn(const Sales_data &lhs, const Sales_data &rhs)
{
    return lhs.isbn() < rhs.isbn();
}
//定义容器对象时，需要提供想要使用的操作的函数指针
multiset<Sales_data, decltype(comparseIsbn)*> bookStore(comparseIsbn);
```

## pair类型
pair标准库类型定义在头文件utility中。pair也是一个模板类，当创建一个pair时，必须提供两个类型名。pair的数据成员是public的，分别命名为first和seconde。我们可以为每个成员提供初始化：
``` cpp
pair<string, string> author1("James", "Joyce");
pair<string, string> author2{"James", "Joyce"}; //列表初始化
pair<string, string> author3 = make_pair("James", "Joyce");
```
三种创建pair对象的方法：
``` cpp
vector<pair<string, int>> data; //pair的vector
data.push_back(pair<string, int>(s, v));
data.push_back({s, v});  //C++11新标准的列表初始化
data.push_back(make_pair(s, v));
```

# 关联容器操作
关联容器额外的类型别名，使用域运算符提取类型成员，例如，`map<string, int>::key_type`。
> key_type        容器的关键字类型，键的类型
  mapped_type     关键字关联的类型，即键值对中值的类型，只适用于map
  value_type      对于set，与key_type相同
                  对于map，为pair<const key_type, mapped_type>

## 关联容器迭代器
当解引用一个关联容器迭代器时，会得到一个类型为容器的value_type的值的引用。
对map而言，value_type是一个pair类型，first成员保存const的关键字，second成员保存值。
set的迭代器是const的
当使用迭代器遍历map、multimap、set和multiset时，迭代器按关键字升序遍历元素。
我们通常不对关联容器使用泛型算法。在实际编程中，如果我们真的需要对关联容器使用算法，要么将它当作一个源序列，要么当作一个目的位置。例如，用泛型copy算法将元素从一个关联容器拷贝到另一个序列。通过inserter，将关联容器当作一个目的位置来调用另一个算法。

## 添加元素
由于map和set包含不重复的关键字，因此插入一个已存在的元素对容器没有任何影响。
``` cpp
map<string, size_t> word_count;
word_count.insert({word, 1}); //C++11新标准支持
word_count.insert(make_pair(word, 1));
word_count.insert(pair<string, size_t>(word, 1));
word_count.insert(map<string, size_t>::value_type(word, 1));
```
对于不包含重复关键字的容器你，添加单一元素的insert和emplace版本返回一个pair，first成员是一个迭代器，指向具有给定关键字的元素；second成员是一个bool值，表示插入是否成功。如果元素已经在容器中，insert什么也不做，bool值为false。
对于允许重复关键字的容器，接受单个元素的insert总是一个指向新元素的迭代器。因为insert总会插入一个元素。

## 删除元素
关联容器定义三个版本的erase实现删除操作：
``` cpp
c.erase(k)     从c中删除关键字为k的元素，返回size_type的值，指示删除元素的数量。返回0，表示删除元素不在容器中，对于允许重复关键字的容器，删除元素的数目可能大于1

c.erase(p)     从c中删除迭代器p指定的元素。p必须指向c中一个真实的元素，不能等于c.end()。返回一个指向p之后元素的迭代器，如果p指向c中的尾元素，则返回c.end()

c.erase(b, e)  删除迭代器对b和e表示的范围内的元素，返回e
```

## map的下标操作
map和unordered_map容器提供了下标运算符和对应的at函数。multimap和unordered_multimap不能进行下标操作，因为容器中可能有多个值和一个关键字相关联。
map下标的类型为key_type类型，即关键字类型，获得与此关键字相关联的值。使用一个不在容器中的关键字作为下标，会添加一个具有此关键字的元素到map中。因此，我们只可以对非const的map使用下标操作。
使用at关键字访问元素，若k不在c中，则抛出一个out_of_range异常。
``` cpp
map<string, size_t> word_count; // empty map
//插入一个关键字为Anna的元素，关联值进行值初始化；然后将1赋予它
word_count["Anna"] = 1;
```
上述代码的过程如下：
- 在word_count中搜索关键字为Anna的元素，未找到
- 将新的关键字-值对插入到word_count中。关键字保存Anna，值进行值初始化，在本例中意味着值为0
- 提取出新插入的元素，并将值1赋予它

map的下标操作返回mapped_type类型的引用，也就是map的值类型的引用，可以作为左值，进行读写。

## 访问元素
在multimap或multiset中查找元素，如果一个multimap或multiset中包含有多个元素具有给定的关键字，则这些元素会相邻存储。
``` cpp
multimap<string, string> authors;
string search_item("Alain de Botton");
auto enteries = authors.count(search_item); //元素的数量
auto iter = authors.find(search_item); //此作者的第一本书
while(enteries){
    cout << iter->second << endl;
    ++iter;
    --enteries;
}
```
<strong>面向迭代器的解决方法</strong>
使用lower_bound和upper_bound，如果关键字在容器中，lower_bound返回的迭代器将指向第一个具有给定关键字的元素，而upper_bound返回的迭代器则指向最后一个匹配给定的关键字的元素之后的位置。如果元素不在multimap中，则lower_bound和upper_bound返回相等的迭代器。因此，使用相同的关键字调用lower_bound和upper_bound则会得到一个迭代器范围，表示所有具有该关键字的元素。
重写前面的程序：
``` cpp
multimap<string, string> authors;
string search_item("Alain de Botton");
auto beg = authors.lower_bound(search_item);
auto end = authors.upper_bound(search_item);
for(; beg != end; ++beg)
{
    cout << beg->second << endl;
}
```
<strong>equal_range函数</strong>
equal_range函数接受一个关键字，返回一个迭代器pair。若关键字存在，则第一个迭代器指向第一个匹配的元素，第二个迭代器指向最后一个匹配元素之后的位置。若未找到匹配元素，则两个迭代器都指向关键字可以插入的位置。
使用equal_range再次修改我们的程序：
``` cpp
multimap<string, string> authors;
string search_item("Alain de Botton");
for(auto pos = authors.equal_range(search_item);
    pos.first != pos.second; ++pos.first)
{
    cout << pos.first->second << endl;
}
```

# 无序容器
如果关键字类型就是无序的，或者性能测试发现问题可以使用哈希技术解决，就可以使用无序容器。
C++11新标准定义了4个无序关联容器。这些容器使用哈希函数和关键字类型的==运算符。除哈希管理操作之外，无序容器还提供与有序容器相同的操作。

## 管理桶
无序容器使用哈希函数将元素映射到桶。为了访问一个元素，容器首先计算元素的哈希值，指出应该搜索哪个桶。如果容器允许重复关键字，所有具有相同关键字的元素都会在同一个桶中。因此，无序容器的性能依赖于哈希函数的质量和桶的数量和大小。如果一个桶中保存了很多的元素，那么查找一个特定元素就需要大量的比较操作。
无序容器提供了一组管理桶的函数，允许我们查询容器的状态以及在必要时强制容器进行重组：
``` cpp
无序容器管理操作
桶接口
c.bucket_count()                  正在使用的桶的数目
c.max_bucket_count()              容器能容纳的最多的桶的数量
c.bucket_size(n)                  第n个桶中有多少个元素
c.bucket(k)                       关键字为k的元素在哪个桶中

桶迭代
local_iterator                    用来访问桶中元素的迭代器类型
const_local_iterator              桶迭代器的const版本
c.begin(n), c.end(n)              桶n的首元素迭代器和尾后迭代器，返回local_iterator类型
c.cbegin(n), c.cend(n)            返回const_local_iterator

哈希策略
c.load_factor()                   每个桶的平均元素的数量，返回float值
c.max_load_factor()               c试图维护桶的平均桶大小，返回float值。c会在需要时
                                  添加新的桶，使得load_factor <= max_load_factor
c.rehash(n)                       重组存储，使得bucket_count >= n且bucket_size > size/max_load_factor
c.reserve(n)                      重组存储，使得c可以保存n个元素且不必rehash
```

## 无序容器对关键字类型的要求
``` cpp
template < class Key,                                    // unordered_map::key_type
           class T,                                      // unordered_map::mapped_type
           class Hash = hash<Key>,                       // unordered_map::hasher
           class Pred = equal_to<Key>,                   // unordered_map::key_equal
           class Alloc = allocator< pair<const Key,T> >  // unordered_map::allocator_type
           > class unordered_map;
```
默认情况下，无序容器使用关键字类型的==运算符比较元素，使用一个hash<key_type>类型的对象生成每个元素的哈希值。标准库为`内置类型(包括指针)、string和智能指针类型`提供了hash模板。[更多的默认hash模板](http://www.cplusplus.com/reference/functional/hash/)

我们不能直接定义关键字类型为自定义类型的无序容器。为了能将Sales_data用作关键字，我们需要提供函数来代替==运算符和哈希值计算函数。
``` cpp
size_t hasher(const Sales_data &sd)
{
    return hash<string>()(std.isbn()); //使用标准库string的hash函数计算ISBN成员的哈希值
}

bool eqOp(const Sales_data &lhs, const Sales_data &rhs)
{
    return lhs.isbn() == rhs.isbn();
}
//定义类型别名
using SD_multiset = unordered_set<Sales_data, decltype(hasher)*, decltype(eqOp)*>;
//参数是桶的大小、哈希函数指着和相等性判断运算符指针
SD_multiset bookstore(42, hasher, eqOp);
```
