---
title: Perl的undef类型和defined()函数
p: perl/perl_undef_defined.md
date: 2019-07-06 17:37:41
tags: Perl
categories: Perl
---

# Perl的undef和defined()函数

undef表示的像是数据库中的`null`。它表示空，啥也没有，是完全未定义的。这不等于字符串的空，不等于数值0，它是另一种类型。

在某些时候，perl程序本该报错的时候(如使用未赋值的变量，参数越界，读取文件时到了文件结尾eof)，perl实际上不会报错，而是返回undef。但如果开启了warnings功能，则这种类型的问题，某些情况下会给出warning信息，而不是返回undef。

一般情况下，将其当作空或0就好了，因为在需要数值的时候，undef代表的就是0，需要字符串的时候，undef就是空字符串。

所以，perl中的完全可以直接使用未定义的变量，因为未定义的变量起始就是undef。它可以被当作0，也可被当作空字符串。

例如，下面两个语句中，$sum和$str都是未定义的，初始时它们分别表示数值0和空字串''。
```
$sum += $i;
$str .= "abc";
```

可以直接将undef关键字赋值给某个变量，表示这个变量是undef的，这可以取消一个变量的定义。相当于bash shell中的unset。
```
$line=undef;
```

如果想要判断这个undef确实是undef而不是字符串的空，可以使用defined()函数。如果是undef，则该函数返回false，否则返回true。
```
if(defined($ma)){
    print "valid var\n";
}else{
    print "invalid var\n";      # <--- 输出此行
}
```
