---
title: Google搜索技巧
tags:
  - 技术点滴
toc: true
date: 2018-11-28 11:38:40
---
信息检索能力即知识获取能力，如何更好的使用搜索引擎获取信息则需要一定的搜索技巧。本文总结个人搜索使用的常用技巧点，随时更新。

# 什么是搜索技巧？
搜索技巧就是在搜索关键字时，配合一些通配符，帮助快速定位到想要的结果。SEO（Search engine optimization）是一种通过了解搜索引擎，以及提高目的网站在有关搜索引擎内排名的方式。
<!--more-->

# 常用搜索技巧
## 完全匹配关键字
使用方法："关键字"，通过给关键字加双引号的方法，得到的搜索结果就是完全按照关键字的顺序来搜，而不是多个关键字分割在不同的部分，提高搜索准确率。
在不加双引号的情况下，有的时候， 两个词中间加一个空格, 它会分别搜索两个词, 可能返回的结果不是我们想要的结果。
>"谷歌搜索技巧"

## 排除关键字
使用方法：关键字 -排除关键字，使用-短横线方式排除指定的关键字，从而剔除不需要的检索记录。
例如：使用完全匹配的同时排除某个关键字的搜索方式：
> "流媒体开发" -android <strong>请注意关键字后的空格不可以省略</strong>
> 排除某网站内容，请使用-site，比如：-site:zhihu.com

## 逻辑搜索
Google无需用明文的“+”来表示逻辑“与”操作，只要空格就可以了。
> C++ "流媒体开发"

Google用大写的“OR”表示逻辑“或”操作。在默认搜索下, 搜索引擎会反馈所有和查询词汇相关的结果, 如果通过OR搜索, 可以得到和两个关键词分别相关的结果, 而不仅仅是和两个关键词都同时相关的结果。
> britney OR beatles

## 模糊匹配
使用方法：关键字*关键字，使用\*星号对一个或者多个关键字进行模糊匹配
>"搜索*擎"

## 同义词搜索
使用方法：~关键字，使用\~表示同义词搜索
> 浙江 ~大学

## 站内搜索
使用方法：关键字 site:网址
> “流媒体开发” site:csdn.net

## 相关网站搜索
使用方法：related: 网址，就会得到这个网址相关的结果
> related: csdn.net <strong>请注意关键字后的空格不可以省略</strong>

## 指定文件类型
使用方法：关键字 filetype:文件类型，<strong>请注意filetype和文件类型间没有空格</strong>
> "流媒体开发" filetype:pdf

Tips：只有Google支持的filetype才可用

## 搜索标题、链接和正文
使用方式：
1. <strong>intitle:</strong>keyword 或者 <strong>allintitle:</strong>keyword1 keyword2 匹配标题关键字
2. <strong>inurl:</strong>keyword 匹配网页地址关键字
3. <strong>intext:</strong>keyword 或者 <strong>allintext:</strong>keyword1 keyword2 匹配网页正文关键字
4. <strong>inanchor:</strong>keyword 或者 <strong>allinanchor:</strong>keyword1 keyword2 匹配网页中链接内包含的关键字

>allintitle:C++ 流媒体
>inurl:163.com 流星雨
>流星雨 inurl:163.com

# 组合使用
使用方式：上述的搜索技巧可以组合使用，进一步减小搜索的范围，从而获得匹配度更高的搜索结果
>谷歌搜索技巧 site:zhihu.com
