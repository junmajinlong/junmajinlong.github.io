---
title: Perl函数：字符串相关函数
p: perl/perl_str_funcs.md
date: 2019-07-07 17:39:40
tags: Perl
categories: Perl
---

# Perl函数：字符串相关函数

字符串的内置函数有：
```
chomp, chop, chr, crypt, fc, hex, index, lc, lcfirst, length, oct, 
ord, pack, q//, qq//, reverse, rindex, sprintf, substr, tr///, uc, ucfirst, y///
```

分为几类：  
- 字符大小写转换类：  
    - lc：(lower case)将后面的字母转换为小写，是`\L`的实现  
    - lcfirst：将后面第一个字母转换为小写，是`\l`的实现  
    - uc：(uppercase)将后面的字母转换为大写，是`\U`的实现  
    - ucfirst：将后面第一个字母转换为大写，是`\u`的实现  
    - fc：(foldcase)和lc基本等价，只不过fc可以处理UTF-8类的字母  
- 字符处理类函数：  
    - chomp：去除行尾换行符  
    - chop：去除行尾字符，后文详细示例[chop](#blogchop)  
    - reverse：反转列表、标量字符串、hash，后文详细示例[reverse(#blogreverse)]  
    - substr：获取字串，后文详细示例[substr](#blogsubstr)  
    - tr///：字符映射，见[tr///](https://www.cnblogs.com/f-ck-need-u/p/9655841.html)  
    - y///：等价于`tr///`  
- 字符位置索引：  
    - index：获取字符所在索引位置，后文详细示例[index和rindex](#blogindex)  
    - rindex：也是获取字符所在索引位置，后文详细示例[index和rindex](#blogindex)  
- 进制转换类：  
    - hex：将16进制转换为十进制。当字符以`0x`或`x`开头时会自动被认为是十六进制数  
    - oct：将8进制转换为十进制。当字符以`0`开头时会自动被当作8进制数  
    - ord：将字符串的第一个字符转换为ascii码  
    - chr：将ascii转换为对应的字符  
- 其他类：  
    - crypt：(暂时略)  
    - length：返回字符串字符数量，后文详细示例[length](#bloglength)  
    - pack：(暂时略)  
    - q//：相当于给字符串加单引号(quote)  
    - qq//：相当于给字符串加双引号(double quote)  
    - sprintf：(略)printf的不输出版，不用于输出，而用于返回本该输出的内容  


<a name="blogchop"></a>

## chop

和chomp有点类似。

- chop去除行尾字符，返回行尾去除的字符
- 修改hash的时候，去除每个value的最后一个字符，而不是最后一个key，并返回最后一个value被删除的字符
- 修改列表时，去除每个元素的最后一个字符


1.修改字符串
```
$str="abnc";
print chop $str;   # 输出c
print $str;        # 输出abn
```

2.修改hash

```
%myhash=(name => "malongshuai",
           prov => "jiangxi",
           sch  => "linchuan"
            );

$choped = chop %myhash;

while(($key,$value) = each %myhash){
    print $key,":","$value\n";
}

print "=" x 9,"\n";
print "$choped";
```
执行返回：
```
prov:jiangx
sch:linchua
name:malongshua
=========
i
```

3.修改列表

```
@list=qw(aaa bbb ccc ddd);
chop @list;        # 返回aa bb cc dd
print "@list";
```


<a name="blogreverse"></a>

## reverse

reverse用于反转列表、标量字符串、hash。

- 反转列表时，将返回反序的列表  
- 当放在标量上下文时，将做字符串反转  
- 当反转hash时，将把value反转成key，所以当value有重复值时，反转时会丢弃一部分键值  
- reverse不是作用在原始内容上的，而是通过返回值返回反转结果

**反转列表**：将元素反转
```
@arr=qw(abc def ghi);

@arr=reverse @arr;
print "@arr";       # 输出(ghi def abc)

print join(",",reverse "hello","world") ;  # 输出：world,hello
```

**标量上下文下**：串联各元素得到一个标量，然后反转这个标量，即使反转目标是列表
```
@arr=qw(aA bB cC dD );

print scalar reverse @arr;   # 输出：DdCcBbAa
print "\n";
print @arr;   # 输出：aAbBcCdD
```


**反转字符串**：
```
print scalar reverse "hello";  # 输出olleh
```

**反转hash**：会把value反转成key，所以value重复时，将丢弃一部分键值
```

%arr=qw(aA bB cC dD );

%arr1=reverse %arr;
while(($key,$value)=each %arr1){
    print "$key -> ","$value","\n";
}
```

执行结果：
```
dD -> cC
bB -> aA
```


<a name="blogsubstr"></a>

## substr

用于从给定字符串中提取出一段子字符串。

用法：
```
substr STRING,OFFSET,LENGTH,REPLACEMENT
substr STRING,OFFSET,LENGTH
substr STRING,OFFSET
```

其中：  
- offset从0开始计算  
- offset为负数时，表示从尾部位移(从-1开始计算)往前位移  
- length如果忽略，则从offset处开始取到最尾部  
- length为负数时，length则表示从后往前的位移位置，所以将提取从offset到length处的子串  
- replacement替换string中提取出来的字串。需要注意两点：  
    - 加了replacement，将返回提取的子串
    - 但源字符串STRING已被替换


```
use 5.010;
$str="love your everything";

say substr $str,5;     # 输出：your everything
say substr $str,-10;   # 从后往前取：everything
say substr $str,5,4;   # 从前往后取4个字符：your
say substr $str,5,-3;  # 从位移5取到位移-3(从后往前算)：your everyth
say substr $str,5,4,"fairy's";  # 替换源字符串，但返回提取子串：your
say $str;              # 源字符串已被替换：love fairy's everything
```

可以将substr函数作为左值(lvalue)，这样可以修改源变量，就像给了replacement参数一样：
```
use 5.010;
$str="love your everything";

substr($str,5,4) = "fairy's";
say $str;       # 源字符串已被替换：love fairy's everything
```


<a name="blogindex"></a>

## index和rindex

index和rindex用来找出给定字符串中某个子串或某个字符的索引位置。

用法：
```
(r)index STRING,SUBSTR,POSITION
(r)index STRDING,SUBSTR
```
- index用于搜索STRING中第一次出现SUBSTR的位置，rindex则搜索最后一次出现的SUBSTR位置
- 如果省略position，则从起始位置(从0开始计算)开始搜索第一次出现的子串
- 给定position，则从position处开始搜索，如果是rindex，则是找position左边的
- 如果STRING中找不到SUBSTR，则返回-1

```
use 5.010;
$str="love you and your everything";

say index $str,"you";     # 输出：5
say index $str,"yours";   # 输出：-1
say index $str,"you",136; # 输出：-1
say index $str,"you",6;   # 从offset=6之后搜索，输出：13
say rindex $str,"you";    # 输出：13
say rindex $str,"you",10; # 找出offset=10左边的最后一个you，输出：5
```

<a name="bloglength"></a>

## length

用于返回字符串的字符数量，不是字节数。如果是字节数，则采用unicode模块。

下面的例子中，将输出11。
```
$str="hello world";
print length $str;
```

length不能直接作用于数组和hash来统计元素个数。想要统计个数：  
- 对于数组，直接将其放在标量上下文即可  
- 对于hash，可使用keys函数返回hash键，然后放进标量上下文  

```
# 数组元素个数
@arr=qw(aaa bbb ccc ddddd eee ff);
print scalar @arr;         # 输出6

# hash元素个数
%myhash=qw(aaa bbb ccc ddddd eee ff);
print scalar keys %myhash;
```

如果length的对象未定义，则返回undef。如果length的对象没有字符但已定义，则返回0。
