---
title: Office 常用操作
tags:
  - 技术点滴
toc: true
date: 2020-04-15 16:01:28
---
本篇文章主要记录工作生活中 Office 三大办公软件的常用操作，根据个人使用记录不断更新完善。
<!--more-->
# Excel
## 数据查找
案例：判断一个表中某列的数据在另一个表数据中是否存在？例如，表 A 的数据如下：

| 是否存在 | 设备编码 | 设备名称 |
| :------- | -------- | -------- |
|          | 510101   | 计算机   |
|          | 510102   | 显示器   |
|          | 510103   | 打印机   |

表 code 数据如下：

| 设备编码 |
| :------- |
| 510101   |
| 510102   |

目标：判断表 A 中设备编码列中的编码在表 code 中是否存在，若存在，则在表 A 的`是否存在`列输出`YES`，否则输出`NO`。在表 A 的是否存在列键入公式，公式具体内容如下：
```
=IF(ISERROR(VLOOKUP(B2,code!A:A,1,FALSE)),"YES","NO")
说明：
B2 表示目标单元格
code!A:A 表示匹配列
1 表示匹配区域的第一列，在上述案例中表示表 code 的设备编码列
```

## 连接多列
使用 Excel 中的两列更新 MySQL 数据库：
```
="update m_monitor set monitor_name_py = '"&B1&"' where monitor_name = '"&A1&"';"
```
待拼接字符串全部使用双引号包含，特别注意单元格位置前后的 `&` 连接符号。

## 快速填充
选中待复制单元格，鼠标悬停在单元格右下角，变成黑色实心，双击即可完成快速填充。

# Word

# Power Point

