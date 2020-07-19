---
title: Qt开发环境搭建
tags:
  - Qt
toc: true
date: 2020-07-19 20:14:23
---
# Qt 安装
1. Qt 官网下载地址：`https://download.qt.io/archive/qt/`
2. Visual Studio 插件下载地址：`https://download.qt.io/archive/vsaddin/`
3. Qt 目录说明：
```
Tools  QtCreator
Qt5.9.8 各个版本编译工具(bin)、头文件(include)、库文件(lib)、插件(plugins)
```
<!--more-->
# 使用 Qt 创建项目
1. 创建项目时不用使用中文路径。
2. 在跨平台应用程序时，文件名全部使用小写，因为在 Windows 平台文件名不区分大小写，Linux 平台文件名区分大小写。
`*.pro.user` 文件指定 Qt 环境，例如 Qt 版本，编译 Visual Studio 版本等，当需要更换编译环境时，可以手工删除该文件，然后重新配置对应的环境。
3. pro 文件用于生成 Makefile 文件，Makefile 文件用于编译。

# Qt Creator 调试
- Windows 下调试工具安装
1. win10 SDK 下载：`https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk`
2. 在安装过程中仅需要安装 `Debugging Tools for Windsows`，其他均不需安装。
3. 安装完成后，在 `C:\Program Files (x86)\Microsoft SDKs\Windows Kits\10\Debuggers\x86`路径下可以看到调试工具`cdb.exe`程序。
4. 在 Qt Creator 工具 -> 选项 -> Kits -> Debuggers 通过手动配置路径方式添加 `cdb.exe` 文件路径。

# Qt Creator 项目配置
- 添加三方库
在 Qt Creator 选择对应的项目，右键选择"添加库"，选择库和头文件路径。
 在更改 pro 配置之后，执行 qmake，然后执行重新构建。
 对引入的库进行简单测试。
 在 pro 文件中使用 `$$PWD` 表示当前路径。待明确当前路径指 pro 文件所在路径还是 Makefile 文件所在路径。
 
 # VS+QT 项目配置
 在安装好 Visual Studio 关于 Qt 插件后，在 Visual Studio 中创建 Qt 项目。
-  在 Visual Studio 对项目设置如下几项内容：
1. 常规：输出目录、平台工具集
2. 调试：工作目录与输出目录一致
3. C/C++设置三方头文件目录
4. 链接器设置三方库文件目录以及引入的库

- 在 Visual  Studio 中设置 Qt 项目
1. 选择 `Qt VS Tools`中的`Convert Project to Qt Tools Project`；然后选择`Export Project`，将创建对应的`pro,pri`文件；最后选择使用 Qt Creator 打开创建的`pro`文件。
2. 在`Qt Vs Tools`通过 `Qt Project Settings`重新配置 Qt 项目。
3. 在`Qt Vs Tools`中选择`Qt Option`添加新的版本，然后在解决方案右键选择`Change Qt Version`更改 Qt 版本。
需要注意三方库的版本一致问题。

