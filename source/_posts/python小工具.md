---
title: python小工具
tags:
  - 调试技术
toc: true
date: 2018-11-19 14:50:23
---
日常使用的Python小工具脚本文件合集：暂时包括按照文件类型分类和文件名日期命名统一脚本。
<!--more-->
# 文件类型分类
``` python
import os
import shutil
from enum import Enum

class dirStruct(Enum):
    DirNone = 1     #直接拷贝到指定的目录
    DirExt = 2      #按后缀名新建文件夹，将相同的文件拷贝到指定的目录
    DirOrigin = 3   #按照原来目录来新建目录并且拷贝文件


def classify_file(srcpath, dstpath, ext, dirstrut):
    for root, _, files in os.walk(srcpath):
        if dirstrut is dirStruct.DirOrigin:
            newpath = root.replace(srcpath, dstpath)
            if not os.path.exists(newpath):
                os.mkdir(newpath)
            for filename in files:
                if os.path.splitext(filename)[1] in ext:
                    filepath = os.path.join(root, filename)
                    shutil.copy(filepath, newpath)

        if dirstrut is dirStruct.DirExt:
            for dirext in ext:
                dirpath = os.path.join(dstpath, dirext.lstrip('.'))
                if not os.path.exists(dirpath):
                    os.mkdir(dirpath)
            for filename in files:
                extname = os.path.splitext(filename)[1]
                if extname in ext:
                    filepath = os.path.join(root, filename)
                    newfilepath = os.path.join(dstpath, extname.lstrip('.'))
                    shutil.copy(filepath, newfilepath)

        if dirstrut is dirStruct.DirNone:
            for filename in files:
                if os.path.splitext(filename)[1] in ext:
                    filepath = os.path.join(root, filename)
                    shutil.copy(filepath, dstpath)


if __name__ == "__main__":
    srcpath = r'E:\download\speedpan\xxxx'
    dstpath = r'E:\download\speedpan\classify'
    classify_file(srcpath, dstpath, ['.jpg'], dirStruct.DirExt)
```

# 统一文件名日期
``` python
import os
import shutil
import re


def normalize_date(filepath, pattern, year):
    for root, _, files in os.walk(filepath):
        for filename in files:
            old_filename = os.path.join(root, filename)
            match = pattern.match(filename)
            if match:
                if len(match.group(2)) == 4:
                    new_filename = match.group(1) + year + match.group(2) + match.group(3)
                    new_filename = os.path.join(root, new_filename)
                    os.rename(old_filename, new_filename)


if __name__ == "__main__":
    filepath = r'E:\download\speedpan\xxxx\jpg'
    pattern = re.compile(r'^(wj)(\d{4,})(.*)')
    # 将形如wj0908文件名开始的文件整理成wj20180908开始的文件名
    normalize_date(filepath, pattern, "2018")
```
