---
title: Windows 平台使用 gSOAP 生成 ONVIF 框架代码
tags:
  - 技术点滴
toc: true
date: 2022-01-19 10:35:27
---
# `gSOAP` 
`gSOAP` 官方网址：[http://www.cs.fsu.edu/~engelen/soap.html](http://www.cs.fsu.edu/~engelen/soap.html)
`gSOAP` 下载地址：[https://sourceforge.net/projects/gsoap2/files/](https://sourceforge.net/projects/gsoap2/files/)
在 `Windows` 平台，`gsoap_2.8.118.zip` 版本因部分变量未声明，会导致生成 `ONVIF` 源码失败，其他版本可以生成成功。

<!--more-->
# `wsdl2h` 和 `soapcpp2` 简介

要用 `gSOAP` 生成 `ONVIF` 源码，必须用到 `wsdl2h` 和 `soapcpp2` 两个工具（执行文件）。其中 `wsdl2h` 工具用于生成 `ONVIF` 框架头文件，`soapcpp2` 工具生成 `ONVIF` 框架源码。

# 代码生成

## 解压
将下载的 `gSOAP` 工具解压至本地目录，进入 `gsoap_2.8.117\gsoap-2.8\gsoap\bin\win64` 目录，在目录中创建 `wsdl`, `onvifgen`, `runtime` 文件夹。 

## 下载 `wsdl`
`wsdl` 下载地址：[https://www.onvif.org/profiles/specifications/](https://www.onvif.org/profiles/specifications/)
根据需要 `ONVIF` 特性下载所需的 `wsdl` 文件，保存至 `wsdl` 文件夹。

## 生成 `C++` 代码
- 打开命令行管理工具，进入 `gsoap_2.8.117\gsoap-2.8\gsoap\bin\win64` 目录，执行如下命令生成 `ONVIF` 框架头文件：
``` bash
wsdl2h -o onvif.h -t ../../typemap.dat ./wsdl/accesscontrol.wsdl ./wsdl/accessrules.wsdl ./wsdl/actionengine.wsdl ./wsdl/advancedsecurity.wsdl ./wsdl/analytics.wsdl ./wsdl/analyticsdevice.wsdl ./wsdl/appmgmt.wsdl ./wsdl/authenticationbehavior.wsdl ./wsdl/bw-2-vs-mod.wsdl ./wsdl/credential.wsdl ./wsdl/deviceio.wsdl ./wsdl/devicemgmt.wsdl ./wsdl/display.wsdl ./wsdl/display2.wsdl ./wsdl/doorcontrol.wsdl ./wsdl/event.wsdl ./wsdl/event-vs.wsdl ./wsdl/federatedsearch.wsdl ./wsdl/imaging.wsdl ./wsdl/media.wsdl ./wsdl/media2.wsdl ./wsdl/provisioning.wsdl ./wsdl/ptz.wsdl ./wsdl/receiver.wsdl ./wsdl/recording.wsdl ./wsdl/replay.wsdl ./wsdl/schedule.wsdl ./wsdl/search.wsdl ./wsdl/security.wsdl ./wsdl/thermal.wsdl ./wsdl/uplink.wsdl
```
编译成功后，在 `win64` 目录中，将生成 `onvif.h` 文件。

- 支持认证，有些 `ONVIF` 接口调用时需要携带认证信息，要使用 `soap_wsse_add_UsernameTokenDigest` 函数进行授权，所以要在 `onvif.h` 头文件开头加入：
``` bash
#import "wsse.h"
```

- 头文件修改完成后，使用如下命令产生框架代码：
``` bash
soapcpp2 -2 -L -x -i -I../../ -I../../import -I../../custom -d./onvifgen onvif.h
```
命令执行成功后，将在 `onvifgen` 目录下生成 `ONVIF` 框架源码。

- 复制其他运行所需头文件至 `runtime` 目录
``` bash
cp ..\..\dom.cpp .\runtime
cp ..\..\custom\duration.c .\runtime
cp ..\..\custom\duration.h .\runtime
cp ..\..\plugin\mecevp.c .\runtime
cp ..\..\plugin\mecevp.h .\runtime
cp ..\..\plugin\smdevp.c .\runtime
cp ..\..\plugin\smdevp.h .\runtime
cp ..\..\stdsoap2.cpp .\runtime
cp ..\..\stdsoap2.h .\runtime
cp ..\..\plugin\threads.c .\runtime
cp ..\..\plugin\threads.h .\runtime
cp ..\..\plugin\wsaapi.c .\runtime
cp ..\..\plugin\wsaapi.h .\runtime
cp ..\..\plugin\wsseapi.cpp .\runtime
cp ..\..\plugin\wsseapi.h .\runtime
```

# 参考资料
[CSDN|ONVIF协议网络摄像机（IPC）客户端程序开发](https://blog.csdn.net/benkaoya/article/details/72424335)