---
title: Perl处理数据(二)：tr和y///
p: perl/perl_tr_y.md
date: 2019-07-07 17:38:34
tags: Perl
categories: Perl
---

# Perl处理数据(二)：tr和y///

tr和`y///`是等价的。用来实现一一映射，但也有额外的功能，就像Linux下的tr命令一样。

用法：
```
tr/SEARCH/REPLACEMENT/cdsr
y/SEARCH/REPLACEMENT/cdsr

其中：
c：取search的补集，将search中未找到的字符全都替换成replacement的最后一个字符
d：删除search中出现的字符
s：压缩重复字符，仅仅只需要压缩不需要替换时，可将replacement指定为空
r：返回的不是替换成功的数量，而是替换成功后的内容，和s///的r修饰符是一样的
```

例如：

**1.映射功能**

将小写字母e替换为大写字母E。  
```
$str="abcdef";
$str =~ y/e/E/;
print "$str\n";
```

将小写字母全替换为大写字母。  
```
$str="abcdef";
$str =~ y/[a-z]/[A-Z]/;
print "$str\n";
```

如果对同一个字母指定不同的映射集，那么第一个映射将生效。
```
$str="aaa ddd eee fff";
$str =~ tr/aaa/xyz/;
print "$str\n";   # 输出xxx ddd eee fff
```

**2.使用r返回替换后的结果**

该修饰符使得处理数据前会先拷贝一份数据，然后对副本数据进行操作，所以原始数据会保持不变。
```
$str="abcdef";
print $str =~ y/e/E/r;
print $str,"\n";
```

所以，可以将r修饰符的操作结果赋值给其它变量。
```
$str="abcde";
$copy = $str =~ y/e/E/r;
print $copy,"\n";
print $str,"\n";
```

**3.用s修饰符压缩字符**
```
$str="abc    ddd eee    ffff";
print $str =~ tr/ //sr;
```

可以压缩换行符：
```
$str1="abc\n\nddd\neee    fff";
print $str1 =~ tr/\n //sr;
```

**4.d修饰符删除search中有，但replacement中没有的字符**

也就是保留replacement的字符，其余删掉。

例如，直接删除某些字符，不指定replacement，那么search中的字符都被删除。
```
$str="abc ddd eee fff";
$str =~ y/de//d;
print qq("$str");   # 输出："abc   fff"
```

```
$str="abc ddd eee fff";
$str =~ y/de/e/d;      # e被保留
print qq("$str");
```

关于d修饰符，有些复杂。如果search部分指定的不是精确的字符，那么被编译时就不知道是哪些字符需要查找，也就无法从replacement中找到哪些字符需要保留，从而replacement部分变得多余。
```
$str="abc ddd eee fff";
$str =~ y/[d-f]/e/d;     # e不会被保留，得到"abc   "
$str =~ y/[def]/e/d;    # e不会被保留
$str =~ y/[df]e/e/d;    # e不会被保留
```

**5.c修饰符取补集，将search中未指定的字符全部替换为replacement中的最后一个字符**

```
$str="aaa bbb ccc ddd";
print $str =~ y/ab/xy/cr;   # aaaybbbyyyyyyyy
```
注意上面，除了a和b外，全都替换成了y字符，x字符被忽略。

如果replacement比search长，则仍然是取replacement的最后一个字符作为替换字符。所以下面的等价：
```
y/ab/xy/c;
y/ab/zxy/c
```
所以，指定一个补集的替换字符即可。

如果同时指定了s修饰符，则补集替换后，再压缩。
```
$str="aaa bbb ccc ddd";
print $str =~ y/ab/xy/scr;   # aaaybbby
```

如果指定了d修饰符，则删除所有未在searchlist中的字符，也就是说replacement部分成了多余的：
```
$str="aaa bbb ccc ddd";
print $str =~ y/ab/xy/dcr;   # aaabbb
```

**6.c和d修饰符一起用时，一个取search的补集保留search，一个删除search，如何处理**？

perl的`y///`是保留search，但不作补集替换，也就是cd修饰符各生效一部分。也因此，replacement部分是多余的。

```
$str="abc ddd eee fff";
$str =~ y/df/x/cd;
print qq("$str");   # 返回：dddfff
```
