---
title: CSAPP之链接
tags:
  - CSAPP
toc: true
date: 2017-11-09 21:25:54
---
# 前言
链接就是将代码和数据组合成一个单一文件的过程，组成的文件可以被加载到内存并执行。链接可以执行于编译时，也可以执行于加载时，甚至执行于运行时。链接在软件开发中扮演着一个关键的角色，因为它使得分离编译成为可能，我们可以将一个大型的应用程序分解为更小更好管理的模块，并且独立的修改和编译这些模块，当我们修改这些模块中的一个时只需要重新编译它，并重新链接应用，而不必重新编译其他文件。

# 编译器驱动程序
在将源代码编译成可执行程序时，会经历如下步骤：首先，运行预处理器将源程序翻译成一个ASCII码的中间文件；然后，运行C编译器将中间文件翻译成一个汇编文件；再运行汇编器将汇编文件翻译成可重定位目标文件；最后，运行链接程序将可重定位目标文件以及一些必要的系统目标文件组合起来，创建一个可执行目标文件。
``` bash
cpp sum.c /tmp/sum.i或者gcc -E sum.c -o /tmp/sum.i  #预处理
/usr/lib/gcc/x86_64-linux-gnu/5/cc1 /tmp/sum.i -Og -o /tmp/sum.s或者gcc -Og -S sum.i -o /tmp/sum.s #编译
as -o /tmp/sum.o /tmp/sum.s或者gcc -c /tmp/sum.s -o /tmp/sum.o  #汇编
ld -o prog /tmp/main.o /tmp/sum.o #链接
```
<!--more-->
<center>![avatar](http://oyh38rhr2.bkt.clouddn.com/github/171109/A04eBDGA1j.jpg)</center>

# 静态链接
链接器的主要任务：
>符号解析  符号解析就是将每个符号引用和一个符号定义关联起来
重定位  编译器和汇编器生成从地址0开始的代码和数据节，链接器通过把每个符号定义与一个内存位置关联起来，从而重定位这些节，然后修改所有对这些符号的引用，使得它们指向这个内存位置。

# 可重定位目标文件
.bss节：未初始化的全局变量，和静态变量以及所有被初始化为0的全局或静态变量。在目标文件中这个节不占实际空间，它仅仅是一个占位符。目标文件格式区分已初始化和未初始化变量是为了空间效率：在目标文件中，未初始化变量，不需要占用任何实际的磁盘空间。运行时在内存中分配这些变量，初始值为0。所以，一种区分.data和.bss节的简单方式就是把`bss`看成是“更好地节省空间(Better Save Sapce)”的缩写。现代的GCC版本根据以下规则将可重定位目标文件的符号分配到COMMON和.bss中：
>COMMON   未初始化的全局变量
.bss     未初始化的静态变量以及初始化为0的全局或静态变量

可重定位目标文件格式如下：
<center>![avatar](http://oyh38rhr2.bkt.clouddn.com/github/171109/ia5cF7fIGk.jpg)</center>

# 符号解析
链接器对重载函数的区分是通过重整(mangling)实现，编译器将每个唯一的方法和参数列表组合编码成一个对链接器来说唯一的名字。
每个定义符号在代码段或数据段中都被分配了存储空间，`将引用符号与对应定义符号建立关联后，就可在重定位时将引用符号的地址重定位为相关联的定义符号的地址`。“符号的定义”其实质就是符号被分配了虚拟地址空间，符号为函数名时指代码所在区；符号为变量即指其占的静态数据区。
强符号和弱符号定义：
>函数和已经初始化的全局变量是强符号，
未初始化的全局变量是弱符号

多重定义符号的处理规则：
>Rule 1: 强符号不能多次定义；强符号只能被定义一次，否则链接错误
Rule 2: 若一个符号被定义为一次强符号和多次弱符号，则按强定义为准
Rule 3: 若有多个弱符号定义，则任选其中一个

GCC的"-fno-common"选项允许我们把所有未初始化的全部变量不以COMMON块的形式处理，一旦一个未初始化的全局变量不是以COMMON块形式存在，那么它就相当于一个强符号，如果其他目标文件中还有同一个变量的强符号定义，链接时就会发生符号重定义错误。GCC编译器选项"-ffunction-section"和"-fdata-section"将函数或变量分别保存到独立的段中。
规则2和规则3的应用会造成一些不易察觉的运行时错误，尤其是如果重复的符号定义还有不同的类型。当怀疑有此类错误时，在GCC中使用`-fno-common`的选项调用链接器，这个选项会告诉链接器在遇到多重定义的符号时，触发一个错误。或者使用`-Werror`选项，它会把所有的警告都变为错误。多重定义导致难以察觉错误示例:
<center>![avatar](http://oyh38rhr2.bkt.clouddn.com/github/171110/bLLL6Hadaj.jpg)</center>

# 静态库链接
把函数编译为独立的目标模块，然后封装成一个单独的静态库。在Linux系统中，静态库以一种称为存档(archive)的文件格式存放在磁盘中，存档文件是一些可重定位目标文件的集合，有一个头部用来描述每个成员目标文件的大小和位置，存档文件名由后缀.a标识。所以应用程序员只要包含较少的库文件的名字，在链接时链接器只复制被应用程序引用的目标模块，从而减少了可执行文件在磁盘和内存中的大小。
关于库的一般准则是将它们放在命令行的结尾。如果各个库的成员是相互独立的，那么这些库就可以以任意的顺序放在命令行的结尾处；如果不是相互独立的，那么必须对它们排序，使得对于每个被静态库的外部成员引用的符号s，在命令行中至少有一个s的定义是在s的引用之后。在某些情况下，为了满足依赖需求，需要重复包含库或者将多个库合并成一个库。

# 重定位

# 动态共享库
动态库使用`-shared和-fPIC`编译选项从而编译位置无关的代码。对于创建依赖动态库的可执行文件其基本思路是`创建可执行文件使，静态执行一些链接，生成部分链接的可执行目标文件，此时动态库中的代码和数据节都没有复制到可执行文件中，仅仅复制了重定位和符号表信息；在程序加载时，完成链接过程，代码和数据才真正的复制到可执行文件中`。在依赖动态库的可执行文件中，包含`.interp`节，包含动态链接器的路径名。动态链接器也是共享目标，但加载器会特殊对待，从而加载和运行加载器，动态链接器通过重定位完成链接过程，具体的动态共享库链接过程如下所示：
<center>![动态共享库链接过程](http://oyh38rhr2.bkt.clouddn.com/github/171116/0JD765eBgD.jpg)</center>

## soname
soname(Short for Shared Object Name)的存在主要是为了共享库兼容性。比如说，有一个程序prog，以及依赖的共享库库libtest.so.1，prog启动的时候需要libtest.so.1，如果链接的时候直接把libtest.so.1传给prog，那么将来库升级为libtest.so.2的时候prog仍然只能使用libtest.so.1，并不能链接到升级后的动态库。然而如果指定soname为libtest.so，那么prog启动的时候将查找的就是libtest.so，而不是其在被链接时实际使用的库libtest.so.1这个文件名。在发布版本1时，我们使用`ln -sf libtest.so.1 libtest.so`对版本1的动态库创建软链接；而在库升级后，我们`ln -sf libtest.so.2 libtest.so`创建新的软链接即可，这样prog不需要任何变动就能享受升级后的库的特性了。另外，`共享库libtest.so.1与libtest.so.2`可以同时存在于系统内，不必非得把libtest.so.2重命名成libtest.so.1。
编译选项`-Wl`中的l应该表示ld的缩写，后面的参数就是选项信息。`-Wl`参数可以将指定的参数传递给链接器。
>`-Wl,-soname,my_soname`指定输出共享库的SONAME
    `gcc -shared -fPIC -Wl,-soname,libfoo.so.1 -o libfoo.so.1.0.0 libfoo1.c libfoo2.c -lbar1 -lbar2`
`-Wl,-rpath=/home/mylib`指定链接器产生的目标程序的共享库查找路径
`-Wl,-export-dynamic`将所有全局符号导出到动态符号表

## 位置无关代码
对GCC使用`-fpic`选项指示GNU编译器生成PIC代码。共享库的编译必须总是使用该选项。
对同一个目标模块中符号的引用因为可以使用PC相对寻址来编译，所以不需要特殊处理就可以实现PIC。然而，对于模块外部和对全局变量的引用需要特殊处理。
<center>![模块内函数调用示例](http://oyh38rhr2.bkt.clouddn.com/github/171116/gc3j58GK5f.jpg)</center>
<center>![模块内数据引用示例](http://oyh38rhr2.bkt.clouddn.com/github/171116/BGlCccbJ0a.jpg)</center>

PIC数据引用基于`代码段中指令和数据变量之间的距离为常量，与代码段和数据段的绝对内存位置无关`。编译器为GOT中每个条目生成一个重定位记录，在加载时，动态链接器会重定位GOT(Global Offset Table,全局偏移表)中的每个条目，使得它包含目标正确的绝对地址。
<center>![模块外部数据GOT引用示例](http://oyh38rhr2.bkt.clouddn.com/github/171116/a8HjA1iJl5.jpg)</center>

PIC函数调用通过GOT和PLT(Procedure Linkage Table,过程链接表)实现。
<center>![模块间函数调用](http://oyh38rhr2.bkt.clouddn.com/github/171116/b3f621I3G6.jpg)</center>
<center>![模块间函数调用](http://oyh38rhr2.bkt.clouddn.com/github/171116/DKCmI6a51D.jpg)</center>

地址无关代码(PIC,Position-independent Code)解决共享对象指令中对绝对地址的重定位问题。希望程序模块中的共享指令部分在装载时不要因为装载地址的改变而改变，其基本想法就是把指令中需要被修改的部分分离出来，跟数据部分放在一起，从而指令部分就可以保持不变，而数据部分可以在每个进程中拥有一个副本。
>如何区分一个动态库是否为PIC
    `readelf -d foo.so | grep TEXTREL`
如果上面的命令有任何输出，那么foo.so就不是PIC的，否则就是PIC的。PIC的DSO是不会包含任何代码段重定位表的，TEXTREL表示代码段重定位表地址

## 运行时加载共享库
使用`dlopen`系列函数在运行时加载共享库时，需要使用`-rdynamic`编译选项。`-rdynamic`编译选项`This instructs the linker to add all symbols, not only used ones, to the dynamic symbol table. This option is needed for some uses of dlopen or to allow obtaining backtraces from within a program`。关于`-rdynamic`更多详细介绍参考博文[gcc选项-g与-rdynamic的异同](https://stackoverflow.com/questions/8623884/gcc-debug-symbols-g-flag-vs-linkers-rdynamic-option)。在Java中使用JNI技术调用本地的C和C++接口函数。

# 库打桩机制
库打桩就是截获对共享库函数的调用，取而代之执行自己的代码。
## 基本原理
给定需要打桩的目标函数，常见一个wrapper函数，其原型和目标函数一致。利用特殊的打桩机制，可以实现让系统调用你的wrapper函数而不是目标函数。wrapper函数中通常会执行自己的逻辑，然后调用目标函数，再将目标函数的返回值传递给调用者。打桩可以发生在编译时、链接时或者程序被加载执行的运行时。不同的阶段都有对应的打桩机制，也有其局限性。

# 处理目标文件工具

## 工具列表
``` bash
ar        #创建静态库，插入、列出、删除和提取成员
strings   #列出一个目标文件中所有可打印的字符串
strip     #从目标文件中删除符号表信息
size      #列出目标文件中节的名字和大小
readelf   #显示目标文件的完整结构
objdump   #二进制工具之母，最重要功能反汇编.text节中的二进制指令
ldd       #列出可执行文件所需要的共享库
```
## readelf
``` bash
readelf -h main.o           #获取目标文件ELF Header信息
readelf -S target.o         #获取ELF文件section信息
readelf -s target.o         #获取ELF文件symbol信息
readelf -r target.o         #获取ELF文件重定位条目信息
readelf -l proc             #获取可执行文件中的程序头表信息
readelf -x .text target.o   #以字节(HEX或字符)形式dump某节的内容
readelf -a target.o         #查看全部段的详细信息
```

## objdump
``` bash
objdump -h target.o           #查看section信息
objdump -r target.o           #查看重定位表
objdump -t libc.a             #查看lbc.a中符号信息
objdump -x target.o           #查看全部头信息
objdump -s target.o           #dump所有节的内容
objdump -s -j .text target.o  #dump指定节的内容
```

## ar
``` bash
ar -t libc.a                 #查看静态库包含了哪些目标文件
ar -x libc.a                 #解压libc.a中所有的目标文件到当前目录
```

## 其他
``` bash
file target.o               #判断文件类型
size target.o               #查看ELF文件代码段，数据段和BSS段长度
nm target.o                 #查看ELF文件符号表
c++filt _ZN1N1C4funcEi      #解析修饰过的符号名
strip libfoo.so             #清除共享库或者可执行文件的所有符号和调试信息
```
