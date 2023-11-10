---
title: Perl的比较操作符
p: perl/perl_cmp.md
date: 2019-07-06 17:37:37
tags: Perl
categories: Perl
---

# Perl的比较操作符

perl的比较操作符和bash完全相反。数值比较采用符号，字符串比较采用字母。

```
数值     字符串      意义
-----------------------------
==       eq        相等
!=       ne        不等
<        lt        小于
>        gt        大于
<=       le        小于或等于
>=       ge        大于或等于
<=>      cmp       返回值-1/0/1
```
最后一个`<=>`和`cmp`用于比较两边的数值/字符串并返回状态码-1/0/1：
- 小于则返回-1
- 等于则返回0
- 大于则返回1

对于`<=>`，如果比较的双方有一方不是数值，该操作符将返回undef。

几个示例：
```
35 != 30 + 5       # false
35 == 35.0         # true
'35' eq '35.0'     # false(str compare)
'fred' lt 'bay'    # false
'fred' lt 'free'   # true
'red' eq 'red'     # true
'red' eq 'Red'     # false
' ' gt ''          # true
10<=>20            # -1
20<=>20            # 0
30<=>20            # 1
```