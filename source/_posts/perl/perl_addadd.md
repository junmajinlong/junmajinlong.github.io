---
title: Perl的自增和自减
p: perl/perl_addadd.md
date: 2019-07-06 17:37:36
tags: Perl
categories: Perl
---

# Perl的自增和自减

perl也支持数值类型的自增和自减操作。不仅如此，还支持字符、字符串的自增、自减。

- 如果自增(++)和自减(--)符号放在数值的前面，则先增减，再返回；
- 如果自增(++)和自减(--)符号放在数值的后面，则先返回，再增减；
- 如果自增、自减操作是独立的一句表达式，则自增、自减符号放在前面或后面都是等价的；
```
$a=10;

# 以下4句为独立的自增、自减表达式，自增自减符号的位置无所谓
$a++;    # 先返回10，再递增为11
++$a;    # 先递增为12，再返回12
--$a;    # 先递减为11，再返回11
$a--;    # 先返回11，再递减为10

# 以下4句为非独立的自增、自减表达式，自增自减符号的位置有影响
$m = $a++;    # 先返回10赋值给$m，然后$a再递增为11，所以执行结束后$m=10,$a=11
$m = ++$a;    # 先递增为12，再赋值给$m，所以执行结束后$m=12,$a=12
$m = --$a;    # 先递减为11，再赋值给$m，所以执行结束后$m=11,$a=11
$m = $a--;    # 先返回11赋值给$m，然后$a再递减为10，所以执行结束后$m=11,$a=10
```

对于字符和字符串的自增、自减。规则是从最后一个字符按照ascii顺序向上进一位，也就是A-->Z,a-->z，如果是Z或z字符，再进一位表示多加一个字符。看示例更容易理解。
```
$a="b"; say ++$a;
c
$a="ba"; say ++$a;
bb
$a="bz"; say ++$a;
ca
$a="Az"; say ++$a;
Ba
$a="bZ"; say ++$a;
cA
$a="zz"; say ++$a;
aaa
```