---
title: Markdown基本语法
tags:
  - 技术点滴
toc: true
date: 2017-10-27 16:04:53
---
# 前言
<strong>Markdown</strong>编辑器支持代码块高亮，是一款简洁高效的编辑器。

# Markdown简介

> Markdown 是一种轻量级标记语言，它允许人们使用易读易写的纯文本格式编写文档，然后转换成格式丰富的HTML页面。 —— [维基百科](https://zh.wikipedia.org/wiki/Markdown)

<blockquote>[百度一下](http://www.baidu.com)</blockquote>

# 代码块
常用代码块指示标签有`bash python c cpp java javascript css xml x86asm haskell`

## Python
``` python
@requires_authorization
def somefunc(param1='', param2=0):
    '''A docstring'''
    if param1 > param2: # interesting
        print 'Greater'
    return (param2 - param1 + 1) or None
class SomeClass:
    pass
>>> message = '''interpreter
... prompt'''
```
<!--more-->
## 汇编
``` x86asm
test   %eax,%eax
je     0x400ef7 <phase_1+23>
callq  0x40143a <explode_bomb>
add    $0x8,%rsp
retq
```
## C++
``` cpp
void foo()
{
    printf("this is a cpp function\n");
}
```

# 列表
Markdown支持有序列表和无序列表，无序列表使用`+`或者`-`作为列表标记：
+ 微信
+ QQ
+ 微博

有序列表使用`#. `其中#表示列表数字标号：
1. 打开冰箱门
2. 将大象塞进去
3. 关上冰箱门

# 其他样式
<small>小号字</small>
<strong>字体加粗</strong>
上标x<sup>2</sup>
下标x<sub>1</sub>
<ins>插入线</ins>
<del>删除线</del>
[百度一下](http://www.baidu.com)

# 图片
## 默认位置
![avatar](http://oyh38rhr2.bkt.clouddn.com/github_2.png)
## 图片居中
<center>
![avatar](http://oyh38rhr2.bkt.clouddn.com/github_2.png)
</center>

# 反馈与建议
- 博客：[@章鱼猫](https://octocat9lee.github.io/)
- 邮箱：<295861542@qq.com>
--------------
感谢阅读这份帮助文档。
