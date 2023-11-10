---
title: Perl处理数据(一)：s替换、split和join
p: perl/perl_substitute_split_join.md
date: 2019-07-07 17:38:33
tags: Perl
categories: Perl
---

# Perl处理数据(一)：s替换、split和join

<a name="blog1546504310"></a>
# s替换
`m//`模式用来匹配文本，也就是说用来找数据。而`s///`用来查找并替换文本，所以可以用来处理文本文件。在有了正则的基础之后，`s///`用起来会简单很多。

用法格式为：

```
$str =~ s/reg/replacement/FLAGS;
```
它表示用reg去搜索`$str`中的内容，并将搜索出来的内容替换为replacement。

1.`s///`**的斜线可以替换为其他对称的符号(括号类)或相同的符号**。

例如`s!!!`、`s###`、`s%%%`、`s()()`、`s{}{}`、`s<><>`、`s[][]`等，还可以混用符号，例如`s{}##`、`s{}()`等。

```
$str = "ma xiaofang or ma longshuai";

$str =~ s/ma/gao/g;

print "$str\n";
```
第二句直接会替换掉原来的`$str`。

2.`s//`**替换的返回值是替换成功的次数(数量)**。

例如上面使用全局替换修饰符`g`，使得替换了两个"ma"，返回值为2，如果去掉全局替换修饰符，则只替换第一个"ma"，返回值为1。

所以，**通过`s///`返回值可以当作布尔值来做判断**：没有替换成功，将返回0，如果替换成功，则返回值至少为1。

```
$str ="ma xiaofang or ma longshuai";
print "substituted" if $str =~ s/ma/gao/;
```

3.还有一种操作符`$str !~ s///`，它的用法和`=~`是相同的，只不过它转换了布尔逻辑：替换成功时返回false，替换失败时返回1。

4.**由于分组后会立即将分组捕获的结果保存到特殊变量`\1`和`$1`中，所以在replacement部分可以使用这些变量**。

```
$str = "gao xiaofang or ma longshuai";

$str =~ s/(gao)(.*)(ma)(.*)/\3$2\1$4/;   # \1和$1都可以在replacement中使用

print "$str\n";      # 输出ma xiaofang or gao longshuai
```

## 修饰符

除了全局修饰符`g`外，`m//`可用的修饰符在`s///`中基本都可用，最常用的修饰符还是`gimsx`。此外，`s///`还有自己的修饰符r和e，稍后解释。

```
$str = "Gao xiaofang or Ma longshuai";
$str =~ s/(gao).* or (ma).*/$2 xiaofang or $1 longshuai/ig;
print "$str\n";
```

再例如，压缩空白：
```
s/(\s)+/\1/g;   # 将多个空白缩减成一个
s/^\s+//;       # 去除行首空白
s/\s+$//;       # 去除行尾空白
s/^\s+|\s+$//;  # 去除行首空白和行尾空白
```

### r修饰符

原本`s///`的返回值是替换成功的次数，使用r修饰符，可以让这个替换操作返回替换后的字符串。几个注意点：  
1. r修饰符实际上是在替换前先拷贝一份待替换数据，然后在副本上进行替换，所以原始数据不会有任何改变  
2. r修饰符实际上是返回拷贝后的数据，如果替换成功，则返回替换后的字符串，如果替换失败，则直接返回这个副本  
3. r修饰符的替换返回结果一定是纯文本字符串，即使它操作的是一个对象  

```
$str = "ma xiaofang or ma longshuai";

print  $str =~ s/Ma/gao/igr,"\n";   # 输出替换后的内容
print "$str\n";                     # 原始内容不变
$copy = $str =~ s/Ma/gao/igr,"\n";  # 替换后的内容赋值给新的变量
print "$copy\n";                    # 输出替换后的内容
```

如果不使用r修饰符，想要将替换的内容输出，只能先将其保存到一个新的变量中，然后输出这个变量：
```
$str = "ma xiaofang or ma longshuai";

($copy = $str) =~ s/Ma/gao/ig;
print "$str\n";        # 原始数据不变
print "$copy\n";       # 替换后的数据
```

如果上面省略了括号，那么表示`$str`被替换，且将成功替换的次数返回给`$copy`。
```
#!/usr/bin/perl
$str = "ma xiaofang or ma longshuai";

$copy = $str =~ s/Ma/gao/ig;
print "$str\n";        # 输出gao xiaofang or gao longshuai
print "$copy\n";       # 输出2 
```

r修饰符在map函数中非常好用，它可以替换一个列表中的某些元素。

例如，下面的map将`@list`中首字母大写的单词替换为小写。需要注意的是这里使用了`{}`。
```
@list = qw(Ma longshuai Gao xiaofang);
@new_list = map {s/([A-Z])([a-z]+)/\L\1\E\2/rg} @list;
print "@new_list\n";
```


### e修饰符


e是一个超神的修饰符，它可以让replacement部分当作一个后执行的表达式。


```
$str="ma longshuai or ma xiaofang";
$str =~ s/ma/$& x 2/eg;
print $str,"\n";
```

执行上面的程序，它将输出"mama longshuai or mama xiaofang"

上面的过程大致为：搜索字符串"ma"，然后将其保存到特殊变量`$&`中，在replacement部分，将其重复一次，所以得到"mama"。

甚至，可以用sprintf直接格式化输出替换后的内容。

<a name="blog1546504311"></a>
# split函数

split函数用于将字符串分割为一个列表(所以在标量上下文返回的是列表的大小)。

split用法如下：
```
split /pattern_sep/,$string,limit
```

其中pattern_sep用于指定分隔符，允许使用正则表达式(一般都是很简单的正则)，且可以指定多个分隔符。limit表示最多分割为几个元素，如果指定为1(默认)，则表示尽可能多地分隔。，

例如，用split分割字符串`abc:def::1234:xyz`，分隔符指定为`:`
```
$str="abc:def::1234:xyz";
@list = split /:/,$str;
print "list: [@list]\n";
print "list_size: ",scalar(@list),"\n";
```

上面的字符串分割后将有5个元素：abc,def,空,1234,xyz。

可以加上一个limit参数，限制最多分隔为多少个元素，例如上面指定limit=2，表示只分隔一次：
```
$str="abc:def::1234:xyz";
@list = split /:/,$str,2;   # 返回"abc","def::1234:xyz"两个元素
```

split在分割字符串的时候，如果分割后字符串首部会出现空字串，split会保留这些空元素，但如果是尾部空字串，则舍弃。

例如：
```
$str=":::abc:def:1234:xyz::::";
@new_list=join(".",split /:/,$str);
print "@new_list\n";       # 输出：...abc.def.1234.xyz
```

上面使用了join函数，指定了用"."将split后列表中的各元素连接起来。

split可以指定多个分隔符，且可以使用正则表达式来表示。

例如：
```
$str="abc:def::12:xyz";
@list = split /::/,$str);  # 返回："abc:def","12:xyz"
@list = split /[:]+/,$str);  # 返回："abc","def","12","xyz"
@list = split /[:0-9]/,$str);  # 返回："abc","def","","","","","xyz"
```

如果不设置分隔符，那么将认为所有的空白(包括行首空白)都是分隔符，且会将连续的多个空白(即使是多个连续的空行)自动压缩当作一个分隔符，但同时它必须也省略第二个字符串参数。也就是说，这时只能对`$_`进行处理。如果省略了分隔符，却设置了待处理的字符串参数，则返回空。
```
$_="abc 123   xyz\tmn\t\tdef\n\nABC";
@arr=split;
print "@arr";   #输出：abc 123 xyz mn def ABC
```

对省略分隔符做个总结：只要是空白，无论是否在行首，无论是否是换行符，所有连续空白都会被当作单个分隔符。

所以它有点类似于`split /\s+/,$str`的行为，只不过指定了分隔符的话就会保留行首一个空白。

还可以指定空分隔符，它会将字符串中的每个字符都分隔。
```
$str="abcdef";
print join(",",split //,$str);   # 输出a,b,c,d,e,f
```

一般来说，分隔符的正则都很简单，如果需要写复杂的模式，请避免在分隔符正则中使用用于分组的括号，因为括号会被当作分隔符。如果想要使用括号，则可以使用非捕获分组`(?:)`的形式。


<a name="blog1546504312"></a>
# join函数

join函数用于将列表中各个元素用给定字符连接起来，和split的行为有点相反。它返回一个列表。

join用法如下：
```
join $sep,$list
```

其中`$sep`只能是字符串，这一点和split不一样。


例如：
```
print join "-",a,b,c,d,efg;   # 输出："a-b-c-d-efg"
```


可以将split后的结果用join换一个分隔符连接起来：
```
$str="abc:def::1234:xyz";
@new_list = join(",",split /:/,$str);
print "@new_list\n";          # 输出：abc,def,,1234,xyz
```

