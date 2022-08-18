---
title: Windows系统下支持H265的ffmpeg和ffplay工具编译
tags:
  - 技术点滴
toc: true
date: 2022-08-17 17:35:12
---
在 `Windows 10` 操作系统下，编译金山云支持 `H265` 编码 `RTMP` 的 `ffmpeg` 和 `ffplay` 工具。
(金山云 `FFmpeg` 地址)[https://github.com/ksvc/FFmpeg]
<!--more-->
# 工具安装
## msys2 安装
`msys2` 下载地址：[www.msys2.org](https://www.msys2.org/)
下载完成之后点击安装，建议安装在默认路径：`C:\msys64`，然后进行下一步安装，当安装完成后先不启动程序，修改代理软件全局代理，加速后续安装软件下载。
在开始任务栏，启动 `MYSY2 MinGW2 x64`：
依次执行如下命令：
``` bash
# 更新所有包
pacman -Syu  # 选择 Y 更新全部
# 再次执行
pacman -Syu
# 安装基础工具
pacman -S base-devel
# 安装编译工具
pacman -S mingw-w64-x86_64-toolchain
# 安装其他工具，注意不要使用 msys2 安装 cmake 工具
pacman -S yasm
pacman -S nasm
pacman -S make
```
安装完成后，将 `msys2` 安装路径下的 `C:\msys64\mingw64\bin` 路径添加至 `Windows 10` 操作系统环境变量。

## 安装 cmake 工具
`cmake` 下载地址：[camke](https://cmake.org/download/)
`cmake` 安装版本：`cmake-3.17.0-rc3-win64-x64`，安装时将路径添加至环境变量，否则手动添加。
安装完成之后我们在 `msys2` 中是找不到 `cmake` 命令的，这里我们把 `windows path` 添加到 `msys2` 中。
在 `Windows` 环境变量中新建一个名字为 `MSYS2_PATH_TYPE` 的环境变量，值改为 `inherit`，然后重启 `msys2` 就可以在 `msys2` 中使用安装的 `cmake` 了。

# 代码编译
## 编译 libx264
`libx264` 下载地址：[x264](https://www.videolan.org/developers/x264.html)
将 `libx264` 源码下载到：`/home/octocat9lee/h265ffmpeg/x264` 路径下，进入上层目录，创建 `x264_install` 目录，并执行如下命令：
``` bash
#!/bin/bash
basepath=$(cd `dirname $0`;pwd)
echo ${basepath}
cd ${basepath}/x264
pwd
./configure --prefix=${basepath}/x264_install --enable-static --enable-shared --extra-ldflags=-Wl,--output-def=libx264.def
make -j8
make install
```
上述命令执行完成后，`libx264` 将安装到 `/home/octocat9lee/h265ffmpeg/x264_install` 目录下。

## 编译 libx265
`libx265` 下载地址：[x265](https://www.videolan.org/developers/x265.html)
将 `libx265` 源码下载到：`/home/octocat9lee/h265ffmpeg/x265_git` 路径下，进入 `/home/octocat9lee/h265ffmpeg/x265_git/build/msys` 路径，执行如下命令生成 `Makefile`：
``` bash
sh make-Makefiles.sh
```
在弹出的窗口中修改安装路径为 `C:\msys64\home\octocat9lee\h265ffmpeg\x265_install`，然后点击 `configure`，最后点击 `generate` 生成 `Makefile` 并关闭窗口。
生成 `Makefile` 后，执行如下命令进行编译和安装：
``` bash
make -j8
make install
```
编译安装完成后，`libx265` 将安装到 `/home/octocat9lee/h265ffmpeg/x265_install` 目录下。

## 编译 SDL
官网下载：[SDL2-2.0.22](https://www.libsdl.org/release/SDL2-2.0.22.tar.gz)
解压至 `/home/octocat9lee/h265ffmpeg` 目录，进入 `SDL2-2.0.22` 目录下，然后执行如下命令：
``` bash
mkdir /home/octocat9lee/h265ffmpeg/sdl_install
./configure --prefix=/home/octocat9lee/h265ffmpeg/sdl_install
make -j
make install
```
编译安装完成后，`sdl` 将安装到 `/home/octocat9lee/h265ffmpeg/sdl_install` 目录下。

## 编译 ffmpeg
下载金山云支持 `RTMP H265` 编码版本：[H265 RTMP ffmpeg](https://github.com/ksvc/FFmpeg)，切换至最新的 `3.4` 版本。
`ffmpeg ` 源码路径：`/home/octocat9lee/h265ffmpeg/FFmpeg`，执行如下命令：
``` bash
#!/bin/bash

#添加x264 x265 sdl pkg路径
x264_pkg_path=/home/octocat9lee/h265ffmpeg/x264_install/lib/pkgconfig
x265_pkg_path=/home/octocat9lee/h265ffmpeg/x265_install/lib/pkgconfig
sdl_pkg_path=/home/octocat9lee/h265ffmpeg/sdl_install/lib/pkgconfig

export PKG_CONFIG_PATH=$x264_pkg_path:$x265_pkg_path:$sdl_pkg_path:$PKG_CONFIG_PATH

./configure --enable-static --enable-pic \
        --disable-encoders --enable-encoder=aac --enable-encoder=libx264 --enable-gpl --enable-libx264 --enable-encoder=libx265  --enable-libx265 \
        --disable-decoders --enable-decoder=aac --enable-decoder=h264 --enable-decoder=hevc  \
        --disable-demuxers --enable-demuxer=aac --enable-demuxer=mov --enable-demuxer=mpegts --enable-demuxer=flv --enable-demuxer=h264 --enable-demuxer=hevc --enable-demuxer=hls  \
        --disable-muxers --enable-muxer=h264  --enable-muxer=flv --enable-muxer=f4v  --enable-muxer=mp4 \
        --disable-doc
make -j8
```
因 `libx264` 版本问题，可能出现 `x264_bit_depth` 未定义问题，将 `libavcodec/libx264.c` 文件中的 `x264_bit_depth` 全部替换成 `X264_BIT_DEPTH`，然后重新编译。
编译完成后，将目录下的 `ffmpeg`、`ffplay` 以及`libx264-164.dll`、`libx265.dll` 和 `SDL2.dll` 文件复制到同一目录下，即可使用 `ffmpeg` 和 `ffplay` 进行 `H265` 的 `RTMP`、`HTTP-FLV`、`HTTP-HLS` 的使用。
使用示例如下：
``` bash
ffmpeg.exe -re -i hevc.mp4 -vcodec copy -an -f flv rtmp://10.0.15.44:20107/live/test
ffplay.exe rtmp://10.0.15.44:20107/live/test
ffplay.exe http://10.0.15.44:20109/live/test.flv
ffplay.exe http://10.0.15.44:20109/live/test.m3u8
```

# 参考资料
[ffmpeg5.0+h264+h265 windows下编译方法](https://blog.csdn.net/qq_37363702/article/details/123277359)