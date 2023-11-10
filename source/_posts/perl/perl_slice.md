---
title: Perl的分片技术
p: perl/perl_slice.md
date: 2019-07-06 17:37:48
tags: Perl
categories: Perl
---

# Perl分片(slice)

在perl中，如果想要取得一部分变量、一部分列表内容、一部分hash内容，可以采用分片(切片)的方式。

注意，perl并未提供字符串的切片方式，但可以使用内置函数substr()来实现一样的功能。

## 空变量赋值

例如，有些语言(如golang)支持空变量赋值(如golang)，以便丢弃那些不准备使用的变量，perl也支持，只需在不想使用的位置上设置undef即可。

例如，下面的变量列表中，就丢弃了php和ruby对应的赋值操作。
```
@arr=qw(python perl shell php ruby);
($py,$perl,$shell,undef,undef) = @arr;
```
perl中有些函数(比如stat和localtime)在列表上下文会返回很多个字段值列表，这时空变量赋值的方式就排上用场了：
```
($sec,$min,$hour,$mday,$mon,$year,undef,undef,undef) = localtime();
```

但是这样的赋值方式还是麻烦，弄错了undef的位置和数量，就会出错，而且有些时候只是想取得值即刻使用，而不想将其赋值给变量存储起来再通过变量来引用。于是，切片就排上用场了。

## Perl数组切片

先看列表切片：

```
qw(aaa bbb ccc ddd)[1,2];
```

这表示将列表(aaa bbb ccc ddd)进行切片，取出其中索引位为1和2的元素，由于索引位从0开始计算，所以表示取出(bbb ccc)。

- 切片返回的是一个列表，所以可以方便地对取出的元素赋值
- 切片中括号中的索引值只要不越界，可以随意写，且可以重复
- 切片的索引中需要的一个列表
- `-1`索引位表示从后向前取的倒数第一个元素，同理`-2`表示倒数第二个
- 中括号中的逗号不是表示范围，而是索引位分隔符

例如，下面的例子中多次取了索引位1和2的元素，且索引位完全乱序的，但这些行为都是允许的。
```
qw(aaa bbb ccc ddd perl shell python)[1,-1,3,2,0,1,2];
```

由于索引位是列表，所以使用范围序列的方式也是允许的：
```
qw(aaa bbb ccc ddd perl shell python)[1..3]; # 等价于 [1,2,3]
```

再看数组切片。所谓数组切片，实际上是将数组转换为列表(数组底层就是列表)，再通过列表的有序性来切片。例如：
```
@arr = qw(aaa bbb ccc ddd);
($a,$b) = @arr[1,3];
print $a,$b;
```

多数时候，数组切片和列表切片是等价的，但是有两点不同：
1. 数组切片可以放在双引号中被解析，从而进行数组的切片替换，而列表切片则不能解析
2. 可以将一系列值赋值给数组切片(也就是切片表达式在等号左边)，从而实现修改数组元素的目的

第一点，示例如下：
```
@arr=qw(perl python shell php);
print "@arr[1,2,3]\n";                  # 成功切片
print  qw(aaa bbb ccc ddd)[1,2],"\n";   # 成功切片
print "qw(aaa bbb ccc ddd)[1,2]\n";     # 不会切片，而是直接当字符串输出
```

第二点，示例如下：
```
@arr=qw(perl python shell php);
@arr[1,2]=qq(cpython csh);    # 将数组的元素python改为cpython，shell改为csh
print "new arr: @arr\n";
```

范围切片时使用`M..N`的方式，如果想要切到倒数第2个元素呢？指定N为-2吗？这肯定是错的。所以如果想切到倒数第某个元素，可以使用`($#arr-N+1)`的方式来表示倒数第N个，例如5个元素的数组，`$#arr`为4，倒数第1个为`$#arr - 0`，倒数第二个为`$#arr - 1`。示例：
```
@arr=qw(perl python shell php);
print @arr[0..($#arr - 2)];
```

## Perl hash切片

hash切片和数组切片行为上类似，但写法上可能有些令人疑惑。例如：
```
%phone_num=(longshuai =>"18012345678",
             xiaofang =>"17012345678",
             tun_er   =>"16012345678",
             fairy    =>"15012345678");

($a,$b,$c)=@phone_num{qw(xiaofang fairy xiaofang)};
print $a,"\n",$b,"\n",$b,"\n";
```

几个需要说明的地方：
- 尽管是hash切片，切片使用的符号仍然是`@`，例如`@hash_name`
- 切片过程使用大括号包围想要取得的**hash键列表**
- 切片的索引是hash键，而非从0开始计算的数值索引位
- 取多个切片元素时，大括号中的hash键是一个由键组成的列表
- 可以将一系列值赋值给hash切片(也就是切片表达式在等号左边)，从而实现修改hash元素值的目的
- hash是不能在双引号中进行替换的，但是hash切片可以在双引号中替换

以下三种hash键形式都是允许的：
```
@phone_num{qw(xiaofang fairy xiaofang)};
@phone_num{("xiaofang","fairy","xiaofang")};
@phone_num{"xiaofang","fairy","xiaofang"};
```

和数组切片可以赋值一样，也可以为hash的切片元素赋值，从而实现修改对应键值对的值。
```
%phone_num=(longshuai =>"18012345678",
             xiaofang =>"17012345678",
             tun_er   =>"16012345678",
             fairy    =>"15012345678");

@number=qw(18087654321 17087654321);
@phone_num{qw/longshuai xiaofang/} = (@number);
print "@phone_num{qw/longshuai xiaofang/}","\n";
```
