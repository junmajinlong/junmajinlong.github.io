---
title: Perl的变量
p: perl/perl_var.md
date: 2019-07-06 17:37:35
tags: Perl
categories: Perl
---

# Perl的变量

在perl中，普通变量被称为"标量变量"(scalar)，标量是指单个值的意思。还有非标量的数据，如数组、列表、hash等。标量变量和这种非标量的关系，类似于英语里面的单数和复数。

"$"开头表示变量，也同样引用变量，这和bash不一样。变量名区分大小写，允许中文字符作为变量名。

```
$age=17; 
$name="longshuai";
$me="$name"." ".$age;
$meme=$me x 2;
print ${meme}."\n";
```

双目赋值：
```
$age = $age+5;
$age += 5;

$age = $age ** 2;
$age **= 2;

$name = ${name}." hello";
$name .= " hello";
```

还有以下变量赋值的方式：
```
$var1=value1,$var2=value2;
($var1,$var2)=(value1,value2);
```
perl中赋值时，总是先计算右边的，再赋值给左边的，所以交换变量非常容易：
```
($var1,$var2)=($var2,$var1)
```