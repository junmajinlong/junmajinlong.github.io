---
title: 0基础学习Perl一行式必知的Perl基础
p: perl/perl_oneliner_basic.md
date: 2019-07-08 17:39:47
tags: Perl
categories: Perl
---

# 0基础的人学习Perl一行式必知的Perl基础

本文是针对没有Perl基础，但想用perl一行式命令取代grep/awk/sed的人，用于速学Perl基础知识。

Perl一行式系列文章：[Perl一行式程序](/perl/index#blogperloneline)

<a name="blog1546359560"></a>
## perl的-e选项

perl命令的**-e选项**后可以书写表达式，例如：
```
perl -e 'print "hello world\n"'
```

Perl中的函数调用经常可以省略括号，所以`print "hello world\n"`表示的是`print("hello world\n")`，但并非总是可以省略括号，一般来说只要不是自己定义的函数，都可以省略括号，除非少数出现歧义的情况。

在unix下**建议使用单引号包围**`perl -e`的表达式，在windows下建议使用双引号包围表达式。本文所有操作都是在Linux下操作的。

如果表达式中有多个语句，各语句之间使用分号`;`隔开。例如：
```
perl -e 'print "hello";print "world"."\n"'
```

注意，上面使用了点号`.`来连接两个字符串。

稍后会解释另一个表达式选项`-E`，它和`-e`功能几乎一样，唯一不同的是`-E`会自动启用一些高版本的功能。

<a name="blog1546359561"></a>
## print、printf、say和sprintf

Perl中的**print函数不会自动换行，所以需要手动加上"\n"来换行**。
```
perl -e 'print "hello world"'
perl -e 'print "hello world\n"'
```

Perl中的**say()函数**会自动换行，用法和print完全一致。但要使用say，需要使用use指定版本号高于5.010，。
```
$ perl -e 'use 5.010;say "hello world"'
hello world
```

使用**`-E`选项**替换`-e`，可以省略版本的指定，因为`-E`自动启用高版本功能。
```
$ perl -E 'say "hello world"'
hello world
```

Perl也支持**printf函数**，语法格式是`printf "format_string",expr,...`。想必看本文的人都知道如何使用printf，所以只给个简单示例。
```
$ perl -e 'printf "hello %s\n", "Perl"'
hello Perl
```

**sprintf()函数**表示按照printf的方式进行格式化，然后保存起来。保存起来的内容可以赋值给变量。
```
$ perl -e '$a = sprintf "hello %s\n", "Perl";print "$a"'
hello Perl
```

上面将格式化后的字符串`helloPerl\n`保存到了变量`$a`中，然后print输出了变量"$a"。

<a name="blog1546359562"></a>
## 变量

Perl中声明变量x需要使用`$x`。例如：
```
$x = "abc"
$y = 33
($x,$y)=("abc",33)
${var} = 333  # 加大括号是标准写法
```

下面是一行式命令中的变量使用示例：
```
perl -e '$a="hello world";print "$a\n"'
```

Perl中变量赋值时，总是会先计算右边的结果，再赋值给左边的变量。所以，变量交换非常方便：
```
($var1,$var2)=($var2,$var1)
```

对于一行式Perl程序，变量可以无须事先声明直接使用。
```
perl -e 'print $a + 1,"\n"'
```

如果想要判断变量是否已经定义，可以使用`defined($var)`。
```
perl -e 'if(defined($a)){print $a,"\n"}'
```

不带任何修饰的变量是全局变量，如果想要定义为局部变量，可以在变量前加上`my`。my是一个函数。
```
my $a = "abc"
```

对于Perl一行式程序来说，几乎不需要考虑my。但及少数情况下需要使用大括号或其它语句块的时候，可以使用my来声明局部变量，出了语句块范围变量就失效。
```
$ perl -e '
        $a = 33;
        {
            my $a = "abc";
            print $a,"\n";
        }
        print $a,"\n";'
```

<a name="blog1546359563"></a>
## 数值、字符串和反斜线序列

例如：
```
4 + 3              # 7
3.12 + 3.22        # 6.34
"4abc" + "3xyz"    # 7
"abc" + "3xyz"     # 3
"1.2abc" + "3.1x"  # 4.3
1..6            # 1 2 3 4 5 6
1.2..6.4        # 1 2 3 4 5 6
```

需要解释下上面的几行语句。  
1. 数值和字符串之间的转换取决于做什么运算。例如加法表示数学运算，会让字符串转换成数值  
2. 浮点数和整数之间的转换仍然取决于所处的环境，在需要整数的时候浮点数会直接截断成整数  

字符串使用单引号或双引号包围，此外，Perl中也有反引号，这3种引用和shell中的单、双、反引号类似。：  
- 双引号表示弱引用，变量可以在双引号中进行内容替换  
- 单引号表示强引用，内部不会进行变量替换，反斜线序列也会失效  
    - 在unix下的perl一行式程序中因为一般使用单引号包围-e表达式，所以一行式perl中单引号比较少用
    - 如果非要使用单引号，可以考虑使用q()来引用，见下文对q、qq和qx的解释  
- 反引号表示执行操作系统命令并取得命令的输出结果，需要记住的是它自带尾随换行符(除非所执行命令就没有带尾随换行)。例如`` $a = `date +%F %T` ``、`` $files = `ls /root` ``  

```
$ perl -e '$a = `date +"%F %T"`;print $a'
2019-01-03 19:55:32
```

Perl中有以下几个常见的反斜线序列:
```
\n
\r
\t
\l    # 将下个字母转换为小写
\L    # 将后面的多个字母都转换为小写，直到遇到\E
\u    # 将下个字母转换为大写
\U    # 将后面的多个字母都转换为大写，直到遇到\E
\Q    # 和\E之间的所有字符都强制当作字面符号
\E    # \L、\U和\Q的结束符号
```

字符串连接需要使用`.`，例如`"abc"."def"`等价于`"abcdef"`。字符串重复次数可以使用小写字母x，例如`"a" x 3`得到`"aaa"`，`"abc" x 2`得到`abcabc`。

Perl中**数值和字符、字符串**都支持自增、自减操作。相关示例参见[Perl中的自增、自减操作](/perl/perl_addadd)。

### q、qq和qx

在Perl中，引号不一定非要写成符号，可以使用**q()来表示单引号、qq()来表示双引号、qx()表示反引号**。其中这里的括号可以替换成其它成对的符号，例如`qq{}、qq//、qq%%`都是可以的。

使用q类型的引用可以避免在shell中一行式Perl程序的引号转义问题。

例如，在一行式Perl中想要保留单引号：
```
$ perl -e "print q(abc'd)"
```


<a name="blog1546359564"></a>
## 数组

Perl中的数组本质上是一个列表(对于Perl一行式命令，可以将列表等价于数组)，要声明数组，使用`@`符号。
```
@arr = ("Perl", "Python", "Shell")
@{arr} = (1,2,3)  # 加上大括号是标准的写法
@empty = ()       # 空数组
```

书写列表时，字符串需要使用引号包围，逗号分隔各个元素。还可以使用qw()来写列表，不需要再使用逗号分隔元素，而是使用空格，且每个元素都默认以单引号的方式包围。所以下面是等价的：
```
@arr = qw(Perl Python Shell abc\ndef)
@arr = ('Perl','Python','Shell,'abc\ndef')
```

**对于一行式的perl命令，变量和数组可以直接使用而无需事先声明**。

数组可以直接放在双引号中输出，默认输出的时候是用空格分隔各元素。
```
$ perl -e '@arr=qw(Perl Python Shell);print "@arr\n"'
Perl Python Shell
```

要取数组中的某个元素，使用`$`符号。第一个元素`$arr[0]`，第二个元素`$arr[1]`。例如：
```
$ perl -e '@arr=qw(Perl Python Shell);print "$arr[1]\n"'
Python
```

数组`$#arr`或`$#{arr}`表示数组的最后一个数组索引值，所以数组元素个数等于该值加1。

如果想要直接取得数组的个数，将数组赋值给一个变量或者使用scalar()函数即可。这涉及到Perl的上下文知识，不是本文内容，所以记住即可。
```
$ perl -e '
        @arr = qw(Shell Perl PHP);
        $num = @arr;print "$num\n";
        print scalar @arr,"\n";'
```

数组的索引可以是负数，-1表示最后一个元素，-2表示倒数第二个元素。所以`$arr[-1]`等价于`$arr[$#arr]`，都是最后一个元素。

<a name="blog1546359565"></a>
### 数组切片

数组支持切片功能，切片操作使用`@`符号，切片操作会返回一个新的列表(数组)。切片时同一个元素可以出现多次，且顺序随意，这比其它语言的切片要自由的多。例如：
```
@arr = qw(Perl Python Shell Ruby PHP)
@arr[0]   # 取第一个元素形成一个新列表
@arr[1,1,0,-2]  # 取两次第2个元素，一次第1个元素，一次倒数第2个元素形成新列表
@arr[1..3]  # 以序列的方式取第2到第4个元素形成新列表
```

下面是一个示例：
```
$ perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        print "@arr[1,1,0,-2]\n"'
Python Python Perl Ruby
```

如果想要取从第2个到倒数第2个元素，可以使用这种切片方式`@arr[1..$#{arr}-1]`。例如：
```
$ perl -e '@arr=qw(Perl Python Shell Ruby PHP);
        print "@arr[1..$#{arr}-1]\n"'
Python Shell Ruby
```

<a name="blog1546359566"></a>
### 操作数组相关函数

数组可以使用**pop/push函数**来移除、追加最尾部的一个元素，使用**shift/unshift函数**移除、插入首部第一个元素。如果想要操作中间某个元素，可以使用**splice()函数**。这些函数的用法参见：[增、删数组元素](/perl/perl_list_array#blog111)。

另外，还有sort()、reverse()函数，在Perl中sort太过强大，不适合在这里展开解释，所以记住它可以用来排序列表(数组)做简单使用即可。例如：
```
$ perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        @arr1 = sort @arr;
        print "@arr1\n"'
PHP Perl Python Ruby Shell
```

对于sort还需注意的是，它不是在原地排序的，而是生成一个排序后的新列表，原数组中元素的顺序并不会受排序的影响。所以需要将这个新列表赋值给另一个数组变量才能得到排序后的结果，正如上面的示例一样。

但也有技巧可以直接输出排序后的结果，而且这个技巧非常有用：
```
$ perl -e '
         @arr=qw(Perl Python Shell Ruby PHP);
         print "@{ [ sort @arr ] }\n"'
PHP Perl Python Ruby Shell
```

这属于Perl的高级技巧，这里大致解释一下。它分成2个部分：
- 第一部分是`[]`，它表示构造一个匿名列表，匿名列表的内容可以来自于字面元素，也可以来自函数的结果或者表达式的结果，正如上面是将sort函数排序的结果构造成匿名列表；  
- 第二部分是`@{xxx}`，它表示将列表xxx引用进行解除，然后可以插入到双引号中进行输出。

所以，当想要**将某个操作的结果直接输出**时，就可以采取这种方式：
```
@{ [ something you do ] }
```

<a name="blog1546359567"></a>
### 遍历数组

要遍历数组，可以使用for、foreach、each，当然也可以使用while，只不过使用for/foreach/each要更方便。关于for/foreach/each/while详细的内容，见后文。
```
# foreach遍历数组
perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        foreach $i (@arr){print "$i\n"}'

# for遍历数组：元素存在性测试遍历法
perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        for $i (@arr){print "$i\n"}'

# for遍历数组：索引遍历法
perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        for($i=0; $i<=$#arr; $i++){print "$arr[$i]\n"}'

# each遍历数组：key/value遍历
perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        while (($index, $value) = each(@arr)){
            print "$index: $value\n"
        }'
```

必须注意的是，Perl中for/foreach以元素存在性测试遍历时，控制变量`$i`是各个元素的引用，而不是复制各个元素再赋值给`$i`，所以在遍历过程中修改`$i`的值会直接修改原数组。

```
$ perl -e '
        @arr=qw(Perl Python Shell Ruby PHP);
        for $i (@arr){$i = $i."X"};  # 为每个元素加尾随字符"X"
        print "@arr\n"'
PerlX PythonX ShellX RubyX PHPX
```

<a name="blog1546359567x"></a>
## split()和join()函数

join()用于将列表连接成字符串，split()用于将字符串分割成列表。

```
join $sep,$list
split /pattern_sep/,$string,limit
```

详细用法和示例参见：[Perl处理数据(一)：s替换、split和join](/perl/perl_substitute_split_join#blog1546504311)。下面是两个简单示例：
```
print join "--",a,b,c,d,e;   # 输出"a--b--c--d--e"

$str="abc:def::1234:xyz";
@list = split /:/,$str;
```
上面的字符串分割后将有5个元素：abc,def,空,1234,xyz。


<a name="blog1546359568"></a>
## hash(关联数组)

Perl也支持hash数据结构，hash结构中key/value一一映射，和Shell中的关联数组是一个概念。

在Perl中，无论是数组还是hash结构，本质都是列表。所以下面的列表数据可以认为是数组，也可以认为是hash，关键在于它赋值给什么类型。
```
("name","longshuai","age",23)
```

在Perl中，数组类型使用`@`前缀表示，hash类型则使用`%`前缀表示。

所以，下面的表示hash结构：
```
%Person = ("name","longshuai","age",23)
%Person = qw(name longshuai age 23)
```

**列表作为hash结构时，每两个元素组成一个key/value对**，其中key部分必须是字符串类型。所以上面的hash结构表示的映射关系为：
```
%Person = (
        name => "longshuai",
        age  => 23,
)
```

实际上，上面使用胖箭头`=>`的写法在Perl中是允许且推荐的，它是元素分隔符逗号的另一种表示方式，且使用胖箭头更能展现一一对应的关系。在使用胖箭头的写法时，如果key是符合命名规范的，可以省略key的引号(因为它是字符串，正常情况下是应该加引号包围的)，正如上面所写的格式。

hash数据**不能在双引号中进行内容替换**，可以直接输出它，但直接输出时hash的元素是紧挨在一起的，这表示hash的字符串化。
```
$ perl -e '
        %hash = qw(name longshuai age 23);
        print "%hash","\n"'
%hash

$ perl -e '
        %hash = qw(name longshuai age 23);
        print %hash,"\n"'
age23namelongshuai
```

从hash中取元素使用`$`表示，如`$hash{KEY}`，从hash中切片使用`@`表示，如`@hash{KEY1,KEY2}`这和数组是一样的。**虽然hash类型本身不能在双引号中进行内容替换，但hash取值或者hash切片可以在双引号中替换**。

```
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        print "$hash{name}","\n"'
longshuai

$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        print "@hash{name,name,age,age}","\n"'
longshuai longshuai 23 23
```

<a name="blog1546359569"></a>
### hash相关函数

主要有keys()、values()、exists()和delete()。

- keys()返回hash结构中所有key组成的列表。例如`keys %hash`  
- values()则返回hash结构中所有value组成的列表。例如`values %hash`  
- exists()用来检测hash结构中元素是否存在。例如`exists $hash{KEY}`  
- delete()用来删除hash中的一个元素。例如`delete $hash{KEY}`  

下面是keys和values函数的示例。
```
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        print "keys:\n";
        print keys %hash,"\n";
        print "values:\n";
        print values %hash,"\n";'
keys:
agename
values:
23longshuai
```

看起来不是很爽，所以赋值给数组再来输出。
```
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        @keys = keys %hash;
        @values = values %hash;
        print "=========\n";
        print "@keys\n";
        print "=========\n";
        print "@values\n";'
=========
age name
=========
23 longshuai
```

**如何知道hash结构中有多少个key/value对**？是否记得将数组(列表)赋值给一个变量时，得到的就是它的个数？
```
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        $nums = keys %hash;
        print "$nums\n";'
2
```

如果想要直接输出个数而不是先赋值给变量，可以**对一个列表使用函数scalar()，它会强制让Perl将列表当作标量(变量)**：
```
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        print scalar keys %hash,"\n";'
2
```

如何排序hash结构？只需对keys的结果应用sort/reverse函数，再进行遍历输出即可。
```
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        for $key (sort keys %hash){
            print "$key => $hash{$key}\n";
        }'
age => 23
name => longshuai
```

<a name="blog1546359570"></a>
### 遍历hash

要遍历hash结构，可以使用while + each或者for/foreach遍历hash的key或value。

```
# 使用while + each
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        while(($key,$value) = each %hash){
                print "$key => $value","\n"
        }'
name => longshuai
age => 23

# 使用for或foreach去遍历keys或values
$ perl -e '
        %hash = (name=>"longshuai",age=>23);
        for $key (keys %hash){
                print "$key => $hash{$key}","\n"
        }'
age => 23
name => longshuai
```

<a name="blog1546359571"></a>
## 默认变量$\_、@ARGV、@\_

在Perl中，有一个非常特殊的变量`$_`，它表示默认变量。

当没有为函数指定参数时、表达式中需要变量但却没给定时都会使用这个默认变量`$_`。

例如:
```
perl -e '$_="abc\n";print'
```

print函数没有给参数，所以默认输出`$_`的内容。

再例如，for/foreach的遍历行为：
```
for $i (@arr){print "$i\n"}
for (@arr){print "$_\n"}
```

for本该需要一个控制变量指向`@arr`中每个元素，但如果没有给定，则使用默认变量`$_`作为控制变量。

用到默认变量的地方很多，没法列出所有使用`$_`的情况。所以简单地总结是：只要某些操作需要参数但却没有给定的时候，就可以使用`$_`。

`$_`是对于标量变量而言的默认变量。对于需要列表/数组的时候，默认变量不再是`$_`，而是`@ARGV`或`@_`：在自定义子程序(函数)内部，默认变量是`@_`，在自定义子程序外部，默认变量是`@ARGV`。例如：
```
$ perl -e '
        $name = shift;
        $age = shift;
        print "$name:$age\n"' longshuai 23
```

默认数组(列表)变量对一行式perl命令来说可能遇不上，所以了解一下足以，只要能在需要的时候看懂即可。

<a name="blog1546359572"></a>
## 布尔值判断

在Perl中，真假的判断很简单：  
1. 数值0为假，其它所有数值为真  
2. 字符串空`""`为假，字符串`0`为假，其它所有字符串为真(例如`00`为真)  
3. 空列表、空数组、undef、未定义的变量、数组等为假  

注意，Perl中没有true和false的关键字，如果强制写true/false，它们可能会被当作字符串，而字符串为真，所以它们都表示真。例如下面的两行，if的测试对象都认为是字符串而返回真。
```
perl -e "if(true){print "aaaa\n"}"
perl -e "if(false){print "aaaa\n"}"
```

<a name="blog1546359572x"></a>
## 大小比较操作

Perl的比较操作符和bash完全相反。数值比较采用符号，字符串比较采用字母。

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

<a name="blog1546359573"></a>
## 逻辑运算

Perl支持逻辑与(`and &&`)、逻辑或(`or ||`)、逻辑非(`not !`)运算，还支持一个额外的`//`操作。它们都会短路计算。

符号类型的逻辑操作`&& || ! //`优先级很高，为了保险，符号两边的表达式应当使用括号包围。关键字类型的逻辑操作优先级很低，不需要使用括号包围。
```
($a > $b) && ($a < $c)
$a > $b and $a < $c
```

Perl的短路计算非常特别，它返回的是最后运算的表达式的值。也就是说，它有返回值，通过返回值可以判断短路计算的布尔逻辑是真还是假。  
- 如果这个返回值对应的布尔值为真，则整个短路计算自然为真  
- 如果这个返回值对应的布尔值为假，则整个短路计算自然为假  

所以，这个返回值既保证短路计算的结果不改变，又能得到返回值。这个返回值有时候很有用，例如，可以通过逻辑或的操作来设置默认值：
```
$name = $myname || "malongshuai"
```

上面的语句中，如果`$myname`为真，则`$name`被赋值为`$myname`，如果`$myname`为假，则赋值为`malongshuai`。

但上面有一种特殊的情况，如果`$myname`已经定义了，且其值为数值0或字符串`0`，它也返回假。

这和预期有所冲突，这时可以使用`//`来替代`||`。`//`表示只要左边的内容已经定义了就返回真，而无论左边的内容代表的布尔值是真还是假。
```
$name = $myname // "malongshuai"
```

所以，就算`$myname`的值为0，`$name`也会赋值为0而不是"malongshuai"。


<a name="blog1546359574"></a>
## 流程控制

<a name="blog1546359575"></a>
### if、unless和三目运算逻辑

语法格式：
```
# if语句
if(TEST){
    ...
} elsif {
    ...
} else{
    ...
}

# unless语句
unless(TEST){
    ...
}

# 三目运算符
expression ? if_true : if_false
```

if表示TEST为真的时候，执行后面的语句块，否则执行elsif或else语句块。

unless则相反，TEST为假的时候，执行后面的语句块。也就是等价于`if(!TEST)`。unless也有else/elsif块，但基本不会用，因为可以转换成if语句。

注意TEST部分，只要它的结果在布尔逻辑上是真，就表示真。例如：
```
if("abc"){}  # 真
if(0){}      # 假
if(0 > 1){}  # 假
```

<a name="blog1546359576"></a>
### where和until循环

语法：
```
while(CONDITION){
    commands;
}

until(CONDITION){
    commands;
}
```

Perl的until和其它某些语言的until循环有所不同，Perl的until循环，内部的commands主体可能一次也不会执行，因为Perl会先进行条件判断，当条件为假时就执行，如果第一次判断就为真，则直接退出until。

<a name="blog1546359576"></a>
### for和foreach循环

Perl中的for循环和Bash Shell类似，都支持两种风格的for：
```
# (1).C语言风格的for
for(expr1;expr2;expr3){...}
for($i=0;$i<100;$i++){...}

# (2).元素存在性测试的for
for $i (LIST){...}

# (3).元素存在性测试的for等价于foreach
foreach $i (LIST){...}
```


对于第一种for语法(即C语言风格的for)，3个部分都可以省略，但分号不能省略：  
1. 省略第一部分expr1表示不做初始化  
2. 省略第二部分expr2表示不设置条件，意味着可能会无限循环  
3. 省略第三部分expr3表示不做任何操作，可能也会无限循环  

例如下面将3个部分全都省略，将会无限循环：
```
for(;;){
    print "never stop";
}
```

对于(2)和(3)的元素存在性测试的循环，用来迭代遍历列表，每次迭代过程中通过控制遍历`$i`引用当前被迭代的元素。注意：
- `$i`是引用被迭代的元素，而不是复制列表中被迭代的元素然后赋值给`$i`，这意味着迭代过程中修改`$i`会直接影响元素列表数据  
- `$i`可以省略，省略时使用默认变量`$_`  
- `$i`在遍历结束后，会恢复为遍历开始之前的值  

例如：
```
$ perl -e '
        @arr = qw(Perl Shell Python Ruby PHP);
        for (@arr){
                $_ = $_."x";
        }
        print "@arr\n"'
Perlx Shellx Pythonx Rubyx PHPx
```

上面没有显式指定控制变量，所以采用默认变量`$_`。且在迭代过程中为每个`$_`都连接了一个尾随字符`x`，它会直接修改元素数组。

<a name="blog1546359576x"></a>
### each遍历


each用来遍历hash或数组，每次迭代的过程中，都获取hash的key和value，数组的index(数值，从0开始)和元素值。


遍历hash：
```
#!/usr/bin/perl -w
use strict;

my %hash = (
    name1 => "longshuai",
    name2 => "wugui",
    name3 => "xiaofang",
    name4 => "woniu",
);

while(my($key,$value) = each %hash){
    print "$key => $value\n";
}
```

输出结果：
```
name4 => woniu
name3 => xiaofang
name2 => wugui
name1 => longshuai
```

遍历数组：
```
#!/usr/bin/perl -w
use strict;

my @arr = qw(Perl Shell Python PHP Ruby Rust);

while(my($key,$value) = each @arr){
    print "$key => $value\n";
}
```
输出结果：
```
0 => Perl
1 => Shell
2 => Python
3 => PHP
4 => Ruby
5 => Rust
```

<a name="blog1546359577"></a>
### 表达式形式的流程控制语句

Perl支持单表达式后面加流程控制符。如下：
```
command OPERATOR CONDITION;
```

例如：
```
print "true.\n" if $m > $n;
print "true.\n" unless $m > $n;
print "true.\n" while $m > $n;
print "true.\n" until $m > $n;
print "$_\n" for @arr;
print "$_\n" foreach @arr;
```

书写时，很多时候会分行并缩进控制符：
```
print "true.\n"    # 注意没有分号结尾
    if $m > $n;
```

改写的方式几个注意点：  
- 控制符左边只能用一个命令。除非使用do语句块，参见下面解释的do语句块  
- for/foreach的时候，不能自定义控制变量，只能使用默认的$_  
- while或until循环的时候，因为要退出循环，只能将退出循环的条件放进前面的命令中  
    - 例如：`print "abc",($n += 2) while $n < 10;`  
    - `print "abc",($n += 2) until $n > 10;`  

改写的方式不能满足需求时，可以使用普通的流程结构。

<a name="blog1546359578"></a>
### 大括号：运行一次的语句块

使用大括号包围一段语句，这些语句就属于这个语句块。这个语句块其实是一个循环块结构，只不过它只循环一次。语句块有自己的范围，例如可以将变量定义为局部变量。
```
$ perl -e '
        $a = 33;
        {
            my $a = "abc";
            print $a,"\n";
        }
        print $a,"\n";'
```

<a name="blog1546359579"></a>
### do语句块

do语句块结构如下：
```
do {...}
```
do语句块像是匿名函数一样，没有名称，给定一个语句块，直接执行。do语句块的返回值是最后一个执行的语句的返回值。

例如，if-elsif-else分支结构赋值的语句：
```
if($gender eq "male"){
    $name="Malongshuai";
} elsif ($gender eq "female"){
    $name="Gaoxiaofang";
} else {
    $name="RenYao";
}
```
改写成do语句块：
```
$name=do{
    if($gender eq "male"){"Malongshuai"}
    elsif($gender eq "female") {"Gaoxiaofang"}
    else {"RenYao"}
};     # 注意结尾的分号
```

前面说过，表达式形式的流程控制语句控制符左边只能是一个命令。例如：
```
print $_+1,"\n";print $_+2,"\n" if $_>3;

# 等价于下面两条语句：
print $_+1,"\n";
print $_+2,"\n" if $_>3;
```

使用do语句块，可以将多个语句组合并当作一个语句。例如：
```
do{print $a+1,"\n";print $a+2,"\n"} if $a>3;
```

`do{}`中有自己的作用域范围，可以声明属于自己范围内的局部变量。

不要把do和大括号搞混了，大括号是被解释的语句块范围的语法符号，可以用来标记自己的作用域范围。但`do{}`是语句，是被执行的语句块，也有自己的作用域范围。

<a name="blog1546359580"></a>
### last/next/redo/continue

- last相当于其它语言里的break关键字，用于退出当前循环块  
- (for/foreach/while/until/执行一次的语句块都属于循环块)，注意是只退出当前层次的循环，不会退出外层循环  
- next相当于其它语言里的continue关键字，用于跳入下一次迭代。同样只作用于当前层次的循环  
- redo用于跳转到当前循环层次的顶端，所以本次迭代中曾执行过的语句可能会再次执行  
- continue表示每轮循环的主体执行完之后，都执行另一段代码  

熟悉sed的人肯定很容易理解这里redo和continue的作用。sed默认情况下会输出每一行被处理后的内容，这是因为它有一个和这里continue一样的逻辑，在perl命令的`-p`选项也一样，到时候会解释这个continue的逻辑。sed的`-D`选项则是处理后立即回到sed表达式的顶端再次对模式空间进行处理，直到模式空间没有内容，这实现的是redo的逻辑。

<a name="blog1546359581"></a>
## BEGIN/END语句块

Perl像awk一样，也有BEGIN和END语句块，功能和awk是一样的。实际上Perl除了BEGIN/END，还有CHECK、INIT和UNITCHECK语句块，不过对于一行式Perl程序，BEGIN/END就足够了。

<a name="blog1546359582"></a>
## Perl命令行参数和ARGV

perl命令行的参数存放在数组ARGV(@ARGV)中，所以可以访问`$ARGV[n]`、遍历，甚至修改命令行参数。

```
$ perl -e 'print "@ARGV\n"' first second
first second
```

不难看出，`@ARGV`数组是从-e表达式之后才开始收集的。

其实ARGV数组有点特别，如果参数中有被读取的文件参数，那么每开始读一个文件，这个文件就从ARGV数组中剔除。所以，在程序编译期间(BEGIN语句块)，ARGV数组中包含了完整的参数列表，处理第一个参数文件时，ARGV数组中包含了除此文件之外的其它参数列表，处理第二个参数文件时，ARGV数组中继续剔除这个文件参数。

例如，perl一行式命令中，`-p`选项会输出参数文件的每一行被处理后的数据，也就是说它会读取参数文件。
```
$ echo aaaa > a.txt
$ echo bbbb > b.txt
$ perl -pe '
        BEGIN{
            print "in BEGIN:\n";
            print "@ARGV\n"
            }
        print "in argv file: @ARGV\n";
        END{
            print "in END:\n";
            print "@ARGV\n"
        }
' a.txt b.txt
```

输出结果：
```
in BEGIN:
a.txt b.txt
in argv file: b.txt
aaaa
in argv file:
bbbb
in END:
```

其实，除了ARGV数组`@ARGV`，还有几个相关的ARGV：
- `$ARGV`：该变量表示当前正在被读取的参数文件  
- `ARGV`：是一个预定义的文件句柄，只对`<>`有效，表示当前正被`<>`读取的文件句柄  
- `ARGVOUT`：也是一个预定义的文件句柄，表示当前正打开用于输出的文件句柄，比较常和一行式Perl程序的-i选项结合使用  

例如：
```
$ echo aaaa > a.txt
$ echo bbbb > b.txt
$ perl -pe '
        BEGIN{
            print "in BEGIN:\n";
            print "@ARGV\n"
            }
        print "reading me: $ARGV\n";
' a.txt b.txt
```

输出结果：
```
in BEGIN:
a.txt b.txt
reading me: a.txt
aaaa
reading me: b.txt
bbbb
```

<a name="blog1546359583"></a>
## 和shell交互：执行shell命令

perl可以直接执行shell中的命令方式有3种，但这里只介绍两种：反引号(或qx)、system()函数。

<a name="blog1546359584"></a>
### 反引号

反引号`` `COMMAND` ``或 `qx(COMMAND)`的方式是执行shell命令COMMAND后，将COMMAND的输出结果保存下来作为返回值，它可以赋值给变量，也可以插入到某个地方。就像shell中的反引号是一样的。
```
perl -e '$datetime = qx(date +"%F %T");print $datetime'
```

需要注意的是，反引号执行的结果中一般都会保留尾随换行符(除非像printf一样明确不给尾随换行符)，所以在print这些变量的时候，可以不用指定"\n"。或者为了保持同意，使用chomp()或chop()函数操作变量，去掉最后一个字符。
```
$ perl -e '
        $datetime = qx(date +"%F %T");
        chop $datetime;    # 或chomp $datetime
        print "$datetime\n";'
```

还可以更简便一些，直接在赋值语句上chop或chomp：
```
$ perl -e '
        chop($datetime = qx(date +"%F %T"));
        print "$datetime\n";'
```

<a name="blog1546359585"></a>
### system()

第二种和shell交互的方式是system()函数。**它会直接输出所执行的命令，而不是将命令的结果作为返回值**。所以，system()函数不应该赋值给变量。

system()要执行的命令部分需要使用引号包围。而对于一行式perl程序来说，直接使用引号是一个难题，所以可以考虑使用qq()、q()的方式。例如：
```
$ perl -e '
        system q(date +"%F %T");'
2019-01-03 20:18:34
```
如果一定想要将system()的结果赋值给变量，得到的赋值结果是system()的返回值，而它的返回值表示的是命令是否成功调用(严格地说是wait()的返回值)。
```
$ perl -e '
        $datetime = system q(date +"%F %T");
        print "$datetime\n";'
2019-01-03 20:23:21
0
```

还可以使用shell的管道、重定向等功能：
```
$myname="Malongshuai";
system "echo $myname >/tmp/a.txt";
system "cat <1.pl";
system 'find . -type f -name "*.pl" -print0 | xargs -0 -i ls -l {}';
system 'sleep 30 &';
```

system()的用法可以参考[Perl和操作系统交互(一)：system、exec和反引号](/perl/perl_system_exec)，这里面对system()的参数做了非常透彻的分析。


<a name="blog1546359586"></a>
## 和shell交互：向perl命令行传递来自shell的数据

对于一行式命令，可能会想要将shell的变量、shell中命令的执行结果通过变量传递给perl命令。本人收集了几种方式，也许不全，但应该足够应付所有情况。

**方式一：通过环境变量`$ENV{VAR}`**  

perl程序在运行时，会自动注册一个hash结构`%ENV`，它收集来自shell的环境变量。例如读取shell的PATH环境变量、查看当前所使用的SHELL。
```
$ perl -e 'print "$ENV{PATH}\n"'
$ perl -e 'print "$ENV{SHELL}\n"'
```

所以，想要获取shell的变量，可以先将其导出为环境变量，再从`%ENV`中获取。
```
$ export name="longshuai"
$ perl -e 'print "$ENV{name}\n"'
```

**方式二：将perl -e 'xxxx'的单引号拆开重组，直接将需要被shell解析的东西暴露给shell去解释**

```
$ name=longshuai
$ perl -e 'print "'$name'\n"'
longshuai
```

上面分成三部分`'print "'`是一部分，`$name`是一部分，它没有被任何引号包围，所以直接暴露给shell进行变量替换，`'\n"'`是最后一部分。

这种方式需要对shell的引号解析非常熟练，对sed来说这种写法有时候是必要的，因为这是sed和shell交互的唯一方式，但对于perl命令行来说，没有必要这样写，因为可读性太差，且很难写，一般人真的不容易写不来。

**方式三：将变量放入参数位置，使其收集到ARGV数组中，然后在perl表达式中shift这些数据**

```
$ name=longshuai
$ age=23
$ perl -e '
        $name = shift;
        $age = shift;
        print "$name,$age\n"' $name $age
longshuai,23
```

注意上面的shift没有给定参数，所以shift会使用默认变量。由于shift期待的操作对象是一个数组(列表)，且不是在子程序内部(子程序内部表示在sub语句块内)，所以使用默认数组`@ARGV`。也就说上面的代码等价于：
```
$ perl -e '
        $name = shift @ARGV;
        $age = shift @ARGV;
        print "$name,$age\n"' $name $age
```

**方式四：使用perl -s选项**

perl的-s选项允许解析`--`之后的`-xxx=yyy`格式的开关选项。
```
$ perl -e 'xxxxx' -- -name=abc -age=23 a.txt b.txt
```

从`--`之后的参数，它们本该会被收集到ARGV中，但如果开启了-s选项，这些参数部分如果是`-xxx=yyy`格式的，则被解析成perl的变量`$xxx`并赋值为yyy，而那些不是`-xxx=yyy`格式的参数则被收集到ARGV数组中。

例如：
```
$ perl -se '
        print "ARGV: @ARGV\n";
        print "NAME & AGE: $name,$age\n";
        ' -- -name="longshuai" -age=23 abc def
ARGV: abc def
NAME & AGE: longshuai,23
```

上面传递的参数是`-name="longshuai" -age=23 abc def`，因为开启了-s选项，所以解析了两个perl变量`$name=longshuai $age=23`，另外两个`abc def`则被收集到ARGV数组中。

**方式五：杜绝使用shell，完全替代为perl。在perl中反引号或qx()也支持运行shell程序**

例如：
```
$ name=longshuai
$ perl -e '
        $name = qx(echo \$name);
        print "name in perl: $name"'
name in perl: longshuai

$ perl -e '
        $time = qx(date +"%T");
        print "time in perl: $time"'
time in perl: 19:52:39
```

注意上面`qx(echo \$name)`的反斜线不可少，否则`$name`会被perl解析，转义后才可以保留$符号被shell解析。


<a name="blog1546359587"></a>
## Perl读写文件

一行式perl程序的重头戏自然是处理文本数据，所以有必要解释下Perl是如何读、写数据的。

<a name="blog1546359588"></a>
### 读取标准输入\<STDIN\>

`<STDIN>`符号表示从标准输入中读取内容，如果没有，则等待输入。`<STDIN>`读取到的结果中，一般来说都会自带换行符(除非发生意外，或EOF异常)。
```
$ perl -e '$name=<STDIN>;print "your name is: $name";'
longshuai     # 这是我输入的内容
your name is: longshuai   # 这是输出的内容
```

因为读取的是标准输入，所以来源可以是shell的管道、输入重定向等等。例如：
```
$ echo "longshuai" | perl -e '$name=<STDIN>;print "yourname is: $name";'
your name is: longshuai
```

在比如，判断行是否为空行。
```
$ perl -e '
        $line=<STDIN>;
        if($line eq "\n"){
            print "blank line\n";
        } else {
            print "not blank: $line"
        }'
```

注意，`<STDIN>`每次只读取一行，遇到换行符就结束此次读取。使用while循环可以继续向后读取，或者将其放在一个需要列表的地方，也会一次性读取所有内容保存到列表中。
```
# 每次只读一行
$ echo -e "abc\ndef" | perl -e '$line=<STDIN>;print $line;'
abc

# 每次迭代一行
$ echo -e "abc\ndef" | perl -e 'while(<STDIN>){print}'
abc
def

# 一次性读取所有行保存在一个列表中
$ echo -e "abc\ndef" | perl -e 'print <STDIN>'
abc
def
```

另外需要注意的是，如果将`<STDIN>`显示保存在一个数组中，因为输出双引号包围的数组各元素是自带空格分隔的，所以换行符会和空格合并导致不整齐。
```
$ echo -e "abc\ndef" | perl -e '@arr=<STDIN>;print "@arr"'
abc
 def
```

解决办法是执行一下chomp()或chop()操作，然后在输出时手动指定换行符。
```
$ echo -e "abc\ndef" | perl -e '@arr=<STDIN>;chomp @a;print "@arr\n"'
abc def
```

<a name="blog1546359589"></a>
### 读取文件输入\<\>

使用两个尖括号符号`<>`表示读取来自文件的输入，例如从命令行参数`@ARGV`中传递文件作为输入源。这个符号被称为"钻石操作符"。

例如读取并输出/etc/passwd和/etc/shadow文件，只需将它们放在表达式的后面即可：
```
$ perl -e '
        while(<>){
            print $_;  # 使用默认变量
        }' /etc/passwd
```

可以直接在perl程序中指定ARGV数组让`<>`读取，是否还记得`$ARGV`变量表示的是当前正在读取的文件？
```
perl -e '@ARGV=qw(/etc/passwd);while(<>){print}'
```

一般来说，只需要`while(<>)`中一次次的读完所有行就可以。但有时候会想要在循环内部继续读取下一行，就像sed的n和N命令、awk的next命令一样。这时可以单独在循环内部使用`<>`，表示继续读取一行。

例如，下面的代码表示读取每一行，但如果行首以字符串abc开头，则马上读取下一行。
```
perl -e '
        while(<>){
            if($_ =~ /^abc/){
                <>;
            }
            ...
        }'
```

<a name="blog1546359590"></a>
### 文件句柄

其实`<>`和`<STDIN>`是两个特殊的文件读取方式。如果想要以最普通的方式读取文件，需要打开文件句柄，然后使用`<FH>`读取对应文件的数据。

打开文件以供读取的方式为：
```
open FH, "<", "/tmp/a.log"
open $fh, "/tmp/a.log"     # 与上面等价
```

文件句柄名一般使用大写字母(如FH)或者直接使用变量(如$fh)。

然后读取即可：
```
while(<FH>){...}
while(<$fh>){...}
```

例如：
```
$ perl -e 'open FH,"/etc/passwd";while(<FH>){print}'
```

除了打开文件句柄以供读取，还可以打开文件句柄以供写入：
```
open FH1,">","/tmp/a.log";   # 以覆盖写入的方式打开文件/tmp/a.log
open FH2,">>","/tmp/a.log";  # 以追加写入的方式打开文件/tmp/a.log
```

要写入数据到文件，直接使用print、say、printf即可，只不过需要在这些函数的第一个参数位上指定输出的目标。默认目标为STDOUT，也就是标准输出。

例如：
```
print FH1,"hello world\n";
say   FH,"hello world\n";
printf FH1,"hello world\n";
```

在向文件句柄写入数据时，如果使用的是变量形式的文件句柄，那么print/say/printf可能会无法区分这个变量是文件句柄还是要输出的内容。所以，应当使用`{$fh}`的形式避免歧义。
```
print {$fh},"hello world\n";
```

<a name="blog1546359591"></a>
## 正则表达式匹配和替换

本文通篇都不会深入解释Perl正则，因为内容太多了，而且我已经写好了一篇从0基础到深入掌握Perl正则的文章[Perl正则表达式超详细教程](/perl/perl_re)以及`s///`替换的文章[Perl的s替换命令](/perl/perl_substitute_split_join)。

对于学习perl一行式程序来说，无需专门去学习Perl正则，会基础正则和扩展正则足以。虽不用专门学Perl正则，但有必要知道Perl的正则是如何书写的。

使用`str =~ m/reg/`符号表示要用右边的正则表达式对左边的数据进行匹配。正则表达式的书写方式为`m//`或`s///`，前者表示匹配，后者表示替换。关于`m//`或`s///`，其中斜线可以替换为其它符号，规则如下：  
- 双斜线可以替换为任意其它对应符号，例如对称的括号类，`m()`、`m{}`、`s()()`、`s{}{}`、`s<><>`、`s[][]`，相同的标点类，`m!!`，`m%%`、`s!!!`、`s###`、`s%%%`等等  
- 只有当m模式采用双斜线的时候，可以省略m字母，即`//`等价于`m//`  
- 如果正则表达式中出现了和分隔符相同的字符，可以转义表达式中的符号，但更建议换分隔符，例如`/http:\/\//`转换成`m%http://%`  

所以要匹配/替换内容，有以下两种方式：
- 方式一：使用`data =~ m/reg/`、`data =~ s/reg/rep/`，可以明确指定要对data对应的内容进行正则匹配或替换  
- 方式二：直接`/reg/`、`s/reg/rep/`，因为省略了参数，所以使用默认参数变量，它等价于`$_ =~ m/reg/`、`$_ =~ s/reg/rep/`，也就是对`$_`保存的内容进行正则匹配/替换  

<a name="blog1546359592"></a>
### 匹配

Perl中匹配操作返回的是匹配成功与否，成功则返回真，匹配不成功则返回假。当然，Perl提供了特殊变量允许访问匹配到的内容，甚至匹配内容之前的数据、匹配内容之后的数据都提供了相关变量以便访问。见下面的示例。

例如：

**1.匹配给定字符串内容**

```
$ perl -e '
        $name = "hello gaoxiaofang";
        if ($name =~ m/gao/){
            print "matched\n";
        }'
```

或者，直接将字符串拿来匹配：
```
"hello gaoxiaofang" =~ m/gao/;
```

**2.匹配来自管道的每一行内容，匹配成功的行则输出**

```
while (<STDIN>){
    chomp;
    print "$_ was matched 'gao'\n" if /gao/;
}
```
上面使用了默认的参数变量`$_`，它表示while迭代的每一行数据；上面还简写正则匹配方式`/gao/`，它等价于`$_ =~ m/gao/`。

以下是执行结果：
```
$ echo -e "malongshuai\ngaoxiaofang" | perl -e "
        while (<STDIN>){
        chomp;
        print qq(\$_ was matched 'gao'\n) if /gao/;
        }"
gaoxiaofang was matched 'gao'
```

**3.匹配文件中每行数据**

```
while (<>){
    chomp;
    if(/gao/){
        print "$_ was matched 'gao'\n";
    }
}
```

<a name="blog1546359593"></a>
### 替换

`s///`替换操作是原地生效的，会直接影响原始数据。
```
$str = "ma xiaofang or ma longshuai";
$str =~ s/ma/gao/g;
print "$str\n";
```

`s///`的返回值是替换成功的次数，没有替换成功返回值为0。所以，`s///`自身可以当作布尔值进行判断，如
```
if(s/reg/rep/){
    print "$_\n";
}

print "$_\n" if s/reg/rep/

while(s/reg/rep/){
        ...
}
```

如果想要直接输出替换后得到的字符串，可以加上`r`修饰符，这时它不再返回替换成功的次数，而是直接返回替换后的数据：
```
$ perl -e '
        $str = "ma xiaofang or ma longshuai";
        print $str =~ s/ma/gao/gr,"\n";'
gao xiaofang or gao longshuai
```


