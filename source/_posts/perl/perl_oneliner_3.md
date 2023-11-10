---
title: Perl一行式：字段处理和计算
p: perl/perl_oneliner_3.md
date: 2019-07-08 17:39:52
tags: Perl
categories: Perl
---

# Perl一行式：字段处理和计算

**perl一行式程序系列文章**：[Perl一行式](/perl/index#blogperloneline)

-----------------------------

## 获取每行最后一个字段

```
$ perl -alne 'print $F[$#F]' file.log
```

这里涉及到了选项`-a`、数组`@F`。这里同时还会解释-F选项，它和-a常一起使用。

选项`-a`和awk的自动字段分割一样，会自动将每行数据划分为几个字段。划分字段的分隔符由-F选项指定。如果没有指定-F，则默认以空白符号进行分割(连续空格被认为是单空格)。

分割后的元素全都收集到一个数组`@F`中，所以第一个字段的内容是`$F[0]`，最后一个字段是`$F[-1]`或`$F[$#F]`。

如果想取多个字段，可以对数组`@F`进行切片，例如第3个字段和第第5个字段`@F[2,4]`，第3个字段到倒数第二个字段是`@F[2..$#F-1]`或`@F[1..~~@F-2]`。


## 获取范围字段

正如上面所解释的，如果想要获取第二个字段到倒数第二个字段：
```
$ perl -lane 'print "@F[1..~~@F-2]"' file.log
$ perl -lane 'print "@F[1..$#F-1]"' file.log
```

## 指定字段分隔符

之所以单独拿出来解释，是因为-F指定分隔符时，空白符号的特殊之处。

对于普通字符，-F自然没有什么问题：
```
$ perl -F: -alne 'print $F[1]' /etc/passwd
```

但是想指定空白字符作为字段的分隔符时，-F选项将出现故障：
```
$ perl -F" " -alne 'print $F[1]' file.log
o
a
i
y
y
```

发现空格分隔符根本没起作用，而是按照NUL作为分隔符对每个字符都分割了。

这个问题在-F选项的官方手册中已经注明了：
```
You can't use literal whitespace or NUL characters in the pattern
```

如果想要指定空白符号作为字段分隔符，可以考虑其它方式。例如使用`\s`的正则模式，或者直接不使用-F，而是直接在-e表达式中使用split函数进行行的分割。
```
$ perl -F'\s+' -anle 'print $F[1]' file.log
$ perl -alne 'split / /;print $F[1]' file.log
```


## 计算所有行数值的总和

假如文件内容为：
```
1 3 5 9
2 3 1
10 2 3 6
```

想要总计所有这些数值之和，可以使用如下方式：
```
$ perl -M'List::Util=sum' -alne '$num += sum @F;END{print $num}' file.log
```

或者，将所有行读取到数组中，最后对数组加总：
```
$ perl -M'List::Util=sum' -alne 'push @S,@F;END{print sum @S}' file.log
```

这种方式对于大文件肯定是不如前一种方式友好的，因为它会将所有行内容都存储起来，而前一种方式为所有行都只存储一个结果`$num`，占用的内存要低的多的多。

## 打乱字段的顺序

```
$ perl -M'List::Util=shuffle' -alne 'print "@{[shuffle @F]}"' file.log
```

在`List::Util`模块中有一个函数shuffle，它会按照随机的顺序打乱一个列表(了解即可，这不是本文的重点)。例如：

```
$ perl -M'List::Util=shuffle' -le 'print "@{[shuffle 1..10]}"'
8 1 3 7 10 5 2 4 6 9
$ perl -M'List::Util=shuffle' -le 'print "@{[shuffle 1..10]}"'
1 2 7 10 8 4 3 5 6 9
$ perl -M'List::Util=shuffle' -le 'print "@{[shuffle 1..10]}"'
2 5 7 1 4 6 3 8 9 10
```

这里我想介绍的重点是`"@{[ shuffle @F ]}"`，它是让一个操作可以插入到双引号中的方法，这个在[Perl一行式必知语法基础](/perl/perl_oneliner_basic#blog1546359566)解释过。

虽然目的是插入到双引号中，但它的最终目标是为了让数组元素输出时以空格分隔。所以，这种技巧不是唯一的方法，见下一小节。

## 指定字段输出分隔符(无引号包围)

上一小节通过`@{[shuffle @F]}`的方式可以将打乱数组的操作插入到双引号中。下面是其它方法。

**1.指定数组的字段输出分隔符**`$,`。

```
$ perl -M'List::Util=shuffle' -alne 'BEGIN{$,=" "}print shuffle @F' file.log
```

默认情况下，print/say输出列表的时候，如果数组/列表不插入到双引号中，各元素之间紧连在一起被输出：
```
$ perl -le '@arr=qw(Shell Perl Python);print @arr'
ShellPerlPython
```

特殊变量`$,`指定的就是print/say输出数组且不插入双引号时的元素分隔符，其默认值为undef。

例如指定为空格：
```
$ perl -le '
    @arr=qw(Shell Perl Python);
    $,=" ";
    print @arr'
Shell Perl Python
```

**2.将列表join成字符串，join的连接符指定为空格即可**。
```
$ perl -M'List::Util=shuffle' -anle 'print join " ",shuffle @F' file.log
```

## 指定字段分隔符(引号包围)

变量`$,`控制无双引号包围的数组/列表在print/say输出时的元素分隔符。其实双引号包围的数组/列表被print/say输出时也可以指定元素分隔符。

控制输出双引号包围的列表的元素分隔符的特殊变量是`$"`，默认值为空格。
```
# 默认空格分隔双引号中的元素
$ perl -le '
    @arr = qw(Shell Perl PHP Python);
    print "@arr"'
Shell Perl PHP Python

# $"改变双引号中的元素分隔符
$ perl -le '
    @arr = qw(Shell Perl PHP Python);
    $" = ":";
    print "@arr"'
Shell:Perl:PHP:Python
```

## 所有行中最小数值

假如文件内容为：
```
1 3 0 9
2 -3 1
10 -2 -3 6
```

所有行的最小值为-3，如何取得？

最简单的方式是将所有行都读入并保存到数组中，然后使用`List::Util`模块的min函数取得。
```
$ perl -M'List::Util=min' -anle 'push @nums,@F;END{print min @nums}' num.log
```

但这对于大文件来说内存占用率会很高。比较好的方式是从每行中取出最小值，保留到数组中，最后从这个数组中取出最小值(稍后继续解释更好的方式)。

```
$ perl -M'List::Util=min' -anle '
    push @nums,min @F;
    END{print min @nums}' num.log
```

如果文件的行数量非常大，这也会在内存中保留很多数值，也不是最佳方式。

更好的方式是从每行中取出最小值保存下来，然后和后面的行结合在一起取最小值。这样的方式使得整个处理过程都只占用一行内存空间。

```
$ perl -M'List::Util=min' -anle '
    $min = min ($min // (),@F);
    END{print $min};' num.log
```

这里的关键是`min $min//(),@F`。

首先，`$min//()`表示如果`$num`未定义，则返回空列表`()`，否则返回`$min`。如果这里不进行`$min`是否已定义的判断，那么第一次使用`$min`的时候，它被当作空。所以如果文件中没有负数，下面的操作将会因此而返回空。
```
$ perl -M'List::Util=min' -anle '
    $min = min ($min,@F);
    END{print $min};' num.log
```

再者，上面`$min//()`在`$min`未定义的时候返回的是空列表`()`，不能编写为返回0或空，否则就多出了一个要比较的值。

最后，min函数操作的是一个列表，而Perl会将多个列表压扁形成一个大列表，所以`$min//(),@F`被压成了一个被min函数操作的列表，而`()`表示空列表，这使得第一次使用`$min`的时候不会影响要比较的值。


## 将所有数值取绝对值

假如文件内容为：
```
1 3 0 9
2 -3 1
10 -2 -3 6
```

要返回所有数值的绝对值，可以借用默认函数abs来操作。

简单的逻辑可以遍历`@F`，并应用abs函数，最后追加回列表被输出：
```
$ perl -lane '
    @line=();
    for(@F){push @line,abs};
    print "@line";
    ' num.log
```

在Perl中，对于列表中每个元素都要检查、操作的情形，可以使用map函数。map函数太强大了，堪称逆天级，map的详细用法参见[Perl map用法详解](/perl/perl_list_funcs#blogmap)。这里根据此处需求给出map的使用示例：

```
$ perl -lane 'print "@{[map {abs} @F]}"' num.log
```

其中`map {op} LIST`表示对LIST中的每个元素都执行op操作，操作后的值构成一个新列表。`@{[  ]}`的格式在前面已经出现多次了，不再解释。


## 统计所有匹配字段数量


```
$ perl -lane '/pattern/ && ++$num for @F;END{print $num || 0}' file.log
```

这种统计方式是安全的。先划分字段，然后匹配每个字段，只要匹配到就将计数器变量加1。最后输出计数器的值。但可能匹配不到任何东西，所以必须给计数器变量设置一个默认值，也就是`$num || 0`。

另一种改写方式是：
```
$ perl -anle '$t += /pattern/ for @F;END{print $t}' file.log
```

这里采用的赋值方式`$t += /pattern/`，因为`/pattern/`返回的是匹配成功的数量，不匹配成功则会返回0，所以无需像前面一样设置计数器变量的默认值。

如果使用grep来改写，则一行式命令如下：
```
$ perl -alne '$num += grep /pattern/,@F;END{print $num}' file.log
```

grep返回一个新的表示能匹配的元素列表，但是在标量上下文中列表返回的是元素个数，所以可直接加总计算。


## 生成10个5-15之间的随机数

```
$ perl -le 'print "@{[ map int(rand(10)+5),1..10 ]}"'
10 9 9 11 13 10 14 12 13 6
```

rand(10)表示生成0-9之间的随机浮点数，int(rand(10))表示生成0-9之间的随机整数，加上5表示5-14的随机整数，整个过程执行10次，所以生成了10个随机整数。



## 生成字母表

```
$ perl -le 'print a..z'
abcdefghijklmnopqrstuvwxyz
```

更准确的写法是加上双引号：
```
$ perl -le 'print "a".."z"'
abcdefghijklmnopqrstuvwxyz
```

逗号分隔字母表：
```
$ perl -le 'print join ",","a".."z"'
a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z
```

输出`a`到`zz`：
```
$ perl -le '"a".."zz"'
```

`..`符号会按照字符序列进行递增，`z`递增后得到`aa`，再递增得到`ab`，`az`递增得到`ba`，依次类推。所以这里会返回大量字符(共702个字符)：
```
a..z
aa..az
ba..bz
...
xa..xz
ya..yz
za..zz
```


## 生成8字符随机密码

```
$ perl -le 'print map { ("a".."z")[rand 26] } 1..8'
```

这里`("a".."z")[rand 26]`取26个字母中的一个随机字母，`1..8`表示生成8个字符。

如果想要包含大小写字母、数字，可以：
```
$ perl -le 'print map { ("a".."z","A".."Z",0..9)[rand 62] } 1..8'
```

## 生成8字符随机密码，包含特殊字符

如果想要让生成的随机密码中包含大小写字母、数字、各种标点符号，可以通过ascii码表来指定范围，然后使用chr函数来转换ascii码为字符。

根据ascii，从33开始到126结束是大小写字母、数字、各种标点符号部分。所以：
```
$ perl -le 'print map { chr int(rand(94)) + 33 } 1..8'
```


`int(rand(94) + 33)`表示生成33-126(包括126)之间的随机整数，chr函数可以将数值转换为对应的字符。

```
$ for i in `seq 1 10`;do
    perl -le 'print map {chr int(rand(94))+33} 1..8'
done

>t]>W69#
>]e<Hub6
D}R1l)._
*(HKFZ6Q
x++\"=1O
@K)%.N@s
sSji5&FX
o+.#?@/x
^6[l~%-k
.bkTKA[%
```
