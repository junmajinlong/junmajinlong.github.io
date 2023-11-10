---
title: CSV文件格式说明
p: coding/about_CSV.md
date: 2019-07-06 18:20:41
tags: Coding
categories: Coding
---


csv文件应用很广泛，历史也很悠久。有很多种类型的csv格式，常用的是rfc 4180定义的格式。

csv文件包含一行或多行记录，每行记录中包含一个或多个字段。记录与记录之间使用换行符分隔，最后一个记录可以没有换行符。

```
field1,field2,field3
```

空白不会分隔字段。例如下面有3个字段，第一个字段是"abc def"。
```
abc def,ddd,eee
```

![](/img/referer.jpg)

空行被忽略。带有任何空白字符的(除换行符)行都不算是空行。

字段可以包含双引号，其中引号部分不属于字段的内容：
```
normal string,"quoted-field"
```
的结果是：
```
{`normal string`, `quoted-field`}
```

两个双引号的结果是单个双引号，相当于转义。例如：
```
"the ""word"" is true","a ""quoted-field"""
```
的结果是：
```
{`the "word" is true`, `a "quoted-field"`}
```

换行符和逗号可以被包含在双引号字段中：
```
"Multi-line
field","comma is ,"
```
的结果是：
```
{`Multi-line
field`, `comma is ,`}
```
