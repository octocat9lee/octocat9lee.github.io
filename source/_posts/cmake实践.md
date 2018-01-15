---
title: cmake实践
tags:
  - 技术点滴
toc: true
date: 2018-01-15 10:39:04
---
# cmake基本使用
PROJECT指令隐式定义了两个变量`<projectname>_BINARY_DIR和<projectname>_SOURCE_DIR`。同时cmake系统预定义了`PROJECT_BINARY_DIR和PROJECT_SOURCE_DIR`两个不依赖项目名的变量，分别表示当前编译发生目录和工程代码目录。为了统一，建议直接使用不依赖工程名的预定义变量。
在cmake中使用`${}`引用定义的变量，但在IF控制语句，直接使用变量名引用变量，而不需要`${}`。
在cmake语法中，指令是大小写无关的，但变量和参数是大小写相关的。建议指令全部使用大写。
在内部构建中，cmake并没有指令直接全部清除构建过程的中间文件。因此，cmake强烈推荐的是外部构建(out-of-source build)。
<!--more-->
# 更好一点的Hello World
不论是SUBDIRS还是ADD_SUBDIRECTORY指令(不论是否指定编译输出目录)，我们都可以通过SET指令重新定义EXECUTABLE_OUTPUT_PATH和LIBRARY_OUTPUT_PATH变量来指定最终的目标二进制的位置(指最终生成的hello或者最终的共享库，不包含编译生成的中间文件)。
在指定路径保存可执行文件或者库文件等二进制文件时，使用：
``` bash
SET(EXCUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
```
上述修改目标位置指令位于和`ADD_EXECUTABLE/ADD_LIBRARY`指令相同的CMakeLists.txt文件中。

## 如何安装
cmake的安装依赖指令INSTALL和变量`CMAKE_INSTALL_PREFIX`。CMAKE_INSTALL_PREFIX变量类似configure脚本的-prefix，常见的使用方法：
``` bash
cmake -DCMAKE_INSTALL_PREFIX=/tmp/t2/usr ..
```
如果没有定义CMAKE_INSTALL_PREFIX会安装到什么地方？CMAKE_INSTALL_PREFIX的默认定义是/usr/local。

# 静态库与动态库创建
ADD_LIBRARY指令：
``` bash
ADD_LIBRARY(libname [SHARED|STATIC|MODULE] [EXCLUDE_FROM_ALL] source1 source2 ... sourceN)
```
有时，我们需要同时获得libhello.so/libhello.a两个同名的库，如果直接在CMakeLists.txt文件中添加静态库指令：
``` bash
ADD_LIBRARY(hello STATIC ${LIBHELLO_SRC})
```
因为动态库和静态库target名称相同，所以静态库构建指令无效。
若修改静态库的名字，例如另外构建一个libhello_static.a，我们可以同时获得动态库和静态库。
``` bash
ADD_LIBRARY(hello_static STATIC ${LIBHELLO_SRC})
```
但这种结果并不是我们所需要的，我们需要的是动态库和静态库名字一致。所以，需要我们使用另外的指令。
SET_TARGET_PROPERTIES指令基本语法：
``` bash
SET_TARGET_PROPERTIES(target1 target2... PROPERTIES prop1 value1 prop2 value2 ...)
SET_TARGET_PROPERTIES(hello_static PROPERTIES OUTPUT_NAME "hello") #设置静态库名字
```
cmake在构建一个新的target时，会尝试清理其他使用这个名字的库。为了回避这个问题，需要再次定义CLEAN_DIRECT_OUTPUT属性：
``` bash
SET_TARGET_PROPERTIES(hello PROPERTIES CLEAN_DIRECT_OUTPUT 1)
SET_TARGET_PROPERTIES(hello_static PROPERTIES CLEAN_DIRECT_OUTPUT 1)
```
当我们再次构建时，会发现同时生成了libhello.so和libhello.

与SET_TARGET_PROPERTIES指令对应的是指令为`GET_TARGET_PROPERTY`：
``` bash
# GET_TARGET_PROPERTY基本语法：
GET_TARGET_PROPERTY(VAR target property)
GET_TARGET_PROPERTY(OUTPUT_VALUE hello_static OUTPUT_NAME)
MESSAGE(STATUS  "This is the hello_static OUTPUT_NAME: " ${OUTPUT_VALUE})
如果没有这个属性定义，则返回 NOTFOUND。
```

## 动态库版本号
为了实现动态库版本号，我们仍然需要使用SET_TARGET_PROPERTIES指令。具体使用方法如下：
``` bash
SET_TARGET_PROPERTIES(hello PROPERTIES VERSION 1.2 SOVERSION 1) #VERSION指代动态库版本，SOVERSION指代API版本
```
在build/lib目录下会生成：
``` bash
libhello.so.1.2
libhello.so.1->libhello.so.1.2
libhello.so ->libhello.so.1
```

## 安装共享库和头文件
为了供其他人使用我们的库，我们需要将hello的共享库安装到<prefix>/lib目录，将hello.h安装到<prefix>/include/hello目录。我们使用INSTALL指令：
``` bash
INSTALL(TARGETS hello hello_static
LIBRARY DESTINATION lib
ARCHIVE DESTINATION lib)
#目标类型也就相对应的有三种，ARCHIVE特指静态库，LIBRARY特指动态库，RUNTIME特指可执行目标二进制
INSTALL(FILES hello.h DESTINATION include/hello)
```
然后在build/lib目录下执行如下命令：
``` bash
cmake -DCMAKE_INSTALL_PREFIX=/tmp/cmake/t3 ..
make
make install
```
我们就可以将头文件和库文件安装到目录`/tmp/cmake/t3`中了。

库目录文件下完整的CMakeLists.txt文件如下：
``` bash
#t3/lib/CMakeLists.txt
SET(LIBHELLO_SRC hello.c)
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
ADD_LIBRARY(hello SHARED ${LIBHELLO_SRC})
ADD_LIBRARY(hello_static STATIC ${LIBHELLO_SRC})
SET_TARGET_PROPERTIES(hello_static PROPERTIES OUTPUT_NAME "hello")
SET_TARGET_PROPERTIES(hello PROPERTIES VERSION 1.2 SOVERSION 1)
SET_TARGET_PROPERTIES(hello PROPERTIES CLEAN_DIRECT_OUTPUT 1)
SET_TARGET_PROPERTIES(hello_static PROPERTIES CLEAN_DIRECT_OUTPUT 1)

INSTALL(TARGETS hello hello_static
LIBRARY DESTINATION lib
ARCHIVE DESTINATION lib)
INSTALL(FILES hello.h DESTINATION include/hello)
```

# 使用外部共享库和头文件
在构建失败时，可以使用`make VERBOSE=1`查看构建细节。
## 引入头文件搜索路径
为了让我们的工程能够找到hello.h头文件，我们需要引入一个新的指令INCLUDE_DIRECTORIES，其完整语法为：
``` bash
INCLUDE_DIRECTORIES([AFTER|BEFORE] [SYSTEM] dir1 dir2 ...)
```
我们可以通过两种方式控制搜索路径添加的方式：
``` bash
#设置CMAKE_INCLUDE_DIRECTORIES_BEFORE变量为ON，将添加的头文件搜索路径放在已有路径的前面
SET(CMAKE_INCLUDE_DIRECTORIES_BEFORE ON)
#或者通过AFTER或BEFORE参数控制
INCLUDE_DIRECTORIES(BEFORE /tmp/cmake/t3/include/hello)
```

## 为target添加共享库
为了将目标文件链接到libhello，我们需要使用指令：`LINK_DIRECTORIES和TARGET_LINK_LIBRARIES`。它们的基本语法：
``` bash
LINK_DIRECTORIES(directory1 dirctory2 ...) #添加非标准的共享库搜索路径
TARGET_LINK_LIBRARIES(target library1
<debug | optimized> library2
...)
```
完整的t4/src/CMakeLists.txt文件：
``` bash
INCLUDE_DIRECTORIES(/tmp/cmake/t3/include/hello)

LINK_DIRECTORIES(/tmp/cmake/t3/lib)
ADD_EXECUTABLE(main main.c) #如果位于文件开始处，会提示找不到动态库文件

TARGET_LINK_LIBRARIES(main libhello.so)
```

## 特殊环境变量
<strong>CMAKE_INCLUDE_PATH和CMAKE_LIBRARY_PATH这两个是环境变量而不是cmake变量。</strong>
使用方法是要在Bash Shell中用export设置：
``` bash
$ export CMAKE_INCLUDE_PATH=/tmp/cmake/t3/include/hello/
$ export CMAKE_LIBRARY_PATH=/tmp/cmake/t3/lib
```
这两个变量主要是用来解决以前autotools工程中--extra-include-dir等参数的支持的。也就是，如果头文件没有存放在常规路径(/usr/include、/usr/local/include)，则可以通过这些变量进行弥补。CMAKE_LIBRARY_PATH可以用在FIND_LIBRARY中。
使用设置了CMAKE_INCLUDE_PATH环境变量后的t4/src/CMakeLists.txt文件：
``` bas
FIND_PATH(myHeader hello.h)
IF(myHeader)
MESSAGE(STATUS "header dir: " ${myHeader})
INCLUDE_DIRECTORIES(${myHeader})
ENDIF(myHeader)

LINK_DIRECTORIES(/tmp/cmake/t3/lib)
ADD_EXECUTABLE(main main.c)

TARGET_LINK_LIBRARIES(main libhello.so)
```

# cmake常用变量
cmake变量定义主要有隐式定义和显式定义两种。隐式定义就是例如PROJECT指令，显式定义就是使用SET指令。
## cmake常用变量
>CMAKE_BINARY_DIR/PROJECT_BINARY_DIR/<projectname>_BINARY_DIR 内部构建就是工程顶层目录，外部构建就是编译发生目录
CMAKE_SOURCE_DIR/PROJECT_SOURCE_DIR/<projectname>_SOURCE_DIR 工程顶层目录
CMAKE_CURRENT_SOURCE_DIR 当前处理的CMakeLists.txt所在的路径
CMAKE_CURRENT_BINARY_DIR target编译目录 使用ADD_SURDIRECTORY(src bin)可以更改此变量的值；SET(EXECUTABLE_OUTPUT_PATH <新路径>)并不会对此变量有影响，只是改变了最终目标文件的存储路径
CMAKE_CURRENT_LIST_FILE 输出调用这个变量的CMakeLists.txt的完整路径
CMAKE_CURRENT_LIST_LINE 输出这个变量所在的行
CMAKE_MODULE_PATH 定义自己的cmake模块所在的路径
EXECUTABLE_OUTPUT_PATH 重新定义目标二进制可执行文件的存放位置
LIBRARY_OUTPUT_PATH 重新定义目标链接库文件的存放位置
PROJECT_NAME 返回通过PROJECT指令定义的项目名称
CMAKE_INCLUDE_PATH 系统环境变量，非cmake变量
CMAKE_LIBRARY_PATH 系统环境变量，非cmake变量

## 环境变量
使用`$ENV{NAME}`可以调用系统的环境变量，比如：
``` bash
MESSAGE(STATUS "HOME dir:" $ENV{HOME})
SET(ENV{name} value) #设置环境变量
```
CMAKE_INCLUDE_CURRENT_DIR自动添加CMAKE_CURRENT_BINARY_DIR和CMAKE_CURRENT_SOURCE_DIR到当前处理的CMakeLists.txt。
``` bash
set(CMAKE_INCLUDE_CURRENT_DIR ON)
#相当于在每个CMakeLists.txt加入：
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR})
```
CMAKE_INCLUDE_DIRECTORIES_PROJECT_BEFORE将工程提供的头文件目录始终至于系统头文件目录的前面，当你定义的头文件确实跟系统发生冲突时可以提供一些帮助。
``` bash
SET(CMAKE_INCLUDE_DIRECTORIES_PROJECT_BEFORE ON)
```

## 系统信息
>CMAKE_MAJOR_VERSION   cmake主版本号，如3.2.2中的3
CMAKE_MINOR_VERSION    cmake次版本号，如3.2.2中的2
CMAKE_PATCH_VERSION    cmake补丁等级，如3.2.2中的2
CMAKE_SYSTEM           系统名称，例如Linux-2.6.22
CAMKE_SYSTEM_NAME      不包含版本的系统名，如Linux
CMAKE_SYSTEM_VERSION   系统版本，如2.6.22
CMAKE_SYSTEM_PROCESSOR 处理器名称，如i686
UNIX                   在所有的类UNIX平台为TRUE，包括OS X和cygwin
WIN32                  在所有的win32平台为TRUE，包括cygwin

## 开关选项
>BUILD_SHARED_LIBS 控制默认的库编译方式，默认编译生成的库都是静态库
CMAKE_C_FLAGS 设置C编译选项，也可以通过指令ADD_DEFINITIONS()添加
CMAKE_CXX_FLAGS 设置C++编译选项，也可以通过指令ADD_DEFINITIONS()添加
CMAKE_C_COMPILER  指定C编译器
CMAKE_CXX_COMPILER 指定C++编译器
CMAKE_BUILD_TYPE  build类型(Debug, Release)，CMAKE_BUILD_TYPE=Debug

# cmake常用指令
## 基本指令
>ADD_DEFINITIONS 向C/C++编译添加-D定义，例如ADD_DEFINITIONS(-DDEABLE_DEBUG -DABC)
ADD_DEPENDENCIES 定义target依赖的其他target
AUX_SOURCE_DIRECTORY(dir VARIALBE)  列出目录下所有的源代码文件并存储在一个变量中
EXEC_PROGRAM 在CMakeLists.txt处理过程中执行命令
FILE 文件操作指令
INCLUDE  用来载入CMakeLists.txt文件，也用于载入预定义的cmake模块
FIND_FILE/FIND_LIBRARY/FIND_PATH/FIND_PROGRAM  FIND_系列指令

``` bash
#FIND_LIBRARY示例:
FIND_LIBRARY(libX X11 /usr/lib)
IF(NOT libX)
MESSAGE(FATAL_ERROR "libX not found")
ENDIF(NOT libX)
```

## 控制指令
### IF指令
IF指令的基本语法为：
``` bash
IF(expression)
  # THEN section
  COMMAND1(ARGS ...)
ELSE(expression)
  # ELSE section
  COMMAND1(ARGS ...)
ENDIF(expression)
```
表达式的使用方法如下：
>IF(var)   如果变量不是：空、0、N、NO、OFF、FALSE、NOTFOUND或`<var>_NOTFOUND`时，表达式为真
IF(NOT var)  与上述条件相反
IF(var1 AND var2) 当两个变量都为真是为真
IF(var1 OR var2)  当两个变量其中一个为真时为真
IF(COMMAND cmd)   当给定的cmd确实是命令并调用是为真
IF(EXISTS dir)或IF(EXISTS file)  当目录名或者文件名存在时为真
IF(file1 IS_NEWER_THAN file2)  当file1比file2新，或者file1/file2其中有一个不存在时为真，文件名请使用完整路径
IF(IS_DIRECTORY dirname) 当dirname是目录时为真

匹配正则表达式：
>当给定的变量或者字符串能够匹配正则表达式regex时为真
IF(variable MATCHES regex)
IF(string MATCHES regex)

数字比较表达式：
>IF(variable LESS number)
IF(string LESS number)
IF(variable GREATER number)
IF(string GREATER number)
IF(variable EQUAL number)
IF(string EQUAL number)

按照字母序的排列比较：
>IF(variable STRLESS string)
IF(string STRLESS string)
IF(variable STRGREATER string)
IF(string STRGREATER string)
IF(variable STREQUAL string)
IF(string STREQUAL string)

检测某个变量是否定义的表达式：
>IF(DEFINED variable)   如果变量被定义时为真

判断平台差异小例子：
``` bash
IF(WIN32)
  MESSAGE(STATUS “This is windows.”)
  #作一些Windows相关的操作
ELSE(WIN32)
  MESSAGE(STATUS “This is not windows”)
  #作一些非Windows相关的操作
ENDIF(WIN32)
```
上述代码用来控制在不同的平台进行不同的控制，但是，阅读起来却并不是那么舒服，ELSE(WIN32)之类的语句很容易引起歧义。
使用CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS开关，重新上述的示例：
``` bash
SET(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)
IF(WIN32)
  #do something related to WIN32
ELSEIF(UNIX)
  #do something related to UNIX
ELSEIF(APPLE)
  #do something related to APPLE
ENDIF(WIN32)
```

### WHILE指令
WHILE指令的语法是：
``` bash
WHILE(condition) #判断条件参考IF指令
  COMMAND1(ARGS ...)
  COMMAND2(ARGS ...)
ENDWHILE(condition)
```

### FOREACH
FOREACH指令的使用方式有三种形式：
``` bash
#列表形式
FOREACH(loop_var arg1 arg2 ...)
  COMMAND1(ARGS ...)
ENDFOREACH(loop_var)

#使用AUX_SOURCE_DIRECTORY和FOREACH遍历目录下的所有文件
AUX_SOURCE_DIRECTORY和FOREACH(. SRC_LIST)
FOREACH(F ${SRC_LIST})
  MESSAGE(STATUS ${F})
ENDFOREACH(F)

#范围形式
FOREACH(VAR RANGE 10)
  MESSAGE(STATUS ${VAR})
ENDFOREACH(VAR)

#范围和步进
FOREACH(loop_var RANGE start stop [step])
  COMMAND1(ARGS ...)
ENDFOREACH(loop_var)

FOREACH(A RANGE 5 15 3)
  MESSAGE(STATUS ${A})
ENDFOREACH(A)
```
特别注意的是，<strong>在控制语句条件中使用变量，不能用${}引用，而是直接应用变量名</strong>

# 模块的使用和自定义模块
cmake系统提供类各种模块，一般情况需要使用INCLUDE指令显示的调用，FIND_PACKAGE指令是一个特例，可以直接调用预定义的模块。
## 使用FindCURL模块
向t5/src/CMakeLists.txt中添加：
``` bash
ADD_EXECUTABLE(curltest main.c)
FIND_PACKAGE(CURL)
IF(CURL_FOUND)
  INCLUDE_DIRECTORIES(${CURL_INCLUDE_DIR}) #将头文件加入INCLUDE_DIRECTORIES
  TARGET_LINK_LIBRARIES(curltest ${CURL_LIBRARY}) #指定依赖的库文件
ELSE(CURL_FOUND)
  MESSAGE(FATAL_ERROR "CURL library not found")
ENDIF(CURL_FOUND)
```
对于系统预定义的`Find<name>.cmake`模块，使用方法一般如上所示：
每一个模块都会定义以下几个变量：
>`<name>_FOUND`  #判断模块是否被找到，如果没有找到，按照需要关闭特性或者终止编译
`<name>_INCLUDE_DIR or <name>_ICLUDES` #模块对应头文件目录
`<name>_LIBRARY or <name>_LIBRARIES` #模块对应的库名称

通过`<name>_FOUND`控制工程特性：
``` bash
SET(mySources viewer.c)
SET(optionalSources)
SET(optionalLibs)
FIND_PACKAGE(JPEG)
IF(JPEG_FOUND)
  SET(optionalSources ${optionalSources} jpegview.c)
  INCLUDE_DIRECTORIES(${JPEG_INCLUDE_DIR})
  SET(optionalLibs ${optionalLibs} ${JPEG_LIBRARIES})
  ADD_DEFINITIONS(-DENABLE_JPEG_SUPPORT)
ENDIF(JPEG_FOUND)

FIND_PACKAGE(PNG)
IF(PNG_FOUND)
  SET(optionalSources ${optionalSources} pngview.c)
  INCLUDE_DIRECTORIES(${PNG_INCLUDE_DIR})
  SET(optionalLibs ${optionalLibs} ${PNG_LIBRARIES})
  ADD_DEFINITIONS(-DENABLE_PNG_SUPPORT)
ENDIF(PNG_FOUND)

ADD_EXECUTABLE(viewer ${mySources} ${optionalSources})
TARGET_LINK_LIBRARIES(viewer ${optionalLibs})
```
通过判断系统是否提供JPEG类库来决定是否支持JPEG功能。

## 编写自己的FindHello模块
定义cmake/FindHELLO.cmake模块，在t6/cmake/目录下新建FindHELLO.cmake文件，添加如下内容：
``` bash
FIND_PATH(HELLO_INCLUDE_DIR hello.h /usr/include/hello /usr/local/include/hello /tmp/cmake/t3/include/hello)
FIND_LIBRARY(HELLO_LIBRARY NAMES hello PATH /usr/lib /usr/local/lib /tmp/cmake/t3/lib/)
IF(HELLO_INCLUDE_DIR AND HELLO_LIBRARY)
  SET(HELLO_FOUND TRUE)
ENDIF(HELLO_INCLUDE_DIR AND HELLO_LIBRARY)

IF(HELLO_FOUND)
  IF(NOT HELLO_FIND_QUIETLY)
    MESSAGE(STATUS "Found Hello: ${HELLO_LIBRARY}")
  ENDIF(NOT HELLO_FIND_QUIETLY)
ELSE(HELLO_FOUND)
  IF(HELLO_FIND_REQUIRED)
    MESSAGE(FATAL_ERROR "Could not find hello library")
  ENDIF(HELLO_FIND_REQUIRED)
ENDIF(HELLO_FOUND)
```
因为我们的库未安装到标准的/usr或/usr/local目录下，因此需要在bash中设置CMAKE_LIBRARY_PATH，否则FIND_LIBRARY无法找到对应的库名称。
``` bash
$ export CMAKE_LIBRARY_PATH=/tmp/cmake/t3/lib
```
FIND_PACKAGE指令：
``` bash
FIND_PACKAGE(<name> [major.minor] [QUIET] [NO_MODULE] [[REQUIRED|COMPONENTS] [componets...]])
```
QUIET参数对应FindHELLO中的HELLO_FIND_QUIETLY，REQUIRED参数对应于FindHELLO.cmake模块中的HELLO_FIND_REQUIRED变量。
建立src/CMakeLists.txt文件，内容如下：
``` bash
FIND_PACKAGE(HELLO)
IF(HELLO_FOUND)
  ADD_EXECUTABLE(hello main.c)
  INCLUDE_DIRECTORIES(${HELLO_INCLUDE_DIR})
  TARGET_LINK_LIBRARIES(hello ${HELLO_LIBRARY})
ENDIF(HELLO_FOUND)
```
为了能够让工程找到FindHELLO.cmake模块(存放在工程的cmake目录中)，在主工程文件CMakeLists.txt中加入：
``` bash
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)
PROJECT(NEWHELLO)
SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake) #必须位于ADD_SUBDIRECTORY前面
ADD_SUBDIRECTORY(src)
```

# 代码托管
该博客中所有的源代码归档于[github](https://github.com/octocat9lee/tools/tree/master/cmake)，欢迎共同交流，一起学习进步。
