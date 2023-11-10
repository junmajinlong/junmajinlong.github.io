---
title: Perl函数：列表相关函数
p: perl/perl_list_funcs.md
date: 2019-07-07 17:39:41
tags: Perl
categories: Perl
---

# Perl函数：列表相关函数

内置的列表函数有：
```
grep, join, map, qw//, reverse, sort, unpack
```

- join：将多个元素使用给定字符串联起来[join](#blogjoin)  
- grep：从列表中筛选符合条件的元素执行对应的代码块[grep](#bloggrep)  
- map：对列表中的元素执行给定操作，后文详细示例[map](#blogmap)  
- reverse：反转列表、标量字符串、hash，后文详细示例[reverse](#blogreverse)  
- sort：按照给定规则进行排序[sort](#blogsort)  
- qw//：生成一个列表  
- unpack：暂略   

列表相关的内置函数并不多，有些东西实现起来需要自己写逻辑。不过，有额外的列表工具模块`List::Util`，可以从该模块中获取更完整、更丰富的列表处理工具。

<a name="blogjoin"></a>

## join

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


<a name="bloggrep"></a>

## grep

对给定列表，执行一个或多个代码块，并返回一个经过操作后的新列表，所以在标量上下文中它返回的是新列表中元素的个数。

例如，分别取出一段数值之间的奇数、偶数：
```
@new_arr1 = grep {$_ % 2} (1..100);   # 取奇数
print "@new_arr1";

@new_arr1 = grep {$_ % 2 eq 0} (1..100); # 取偶数
print "@new_arr2";
```


以上面取奇数为例。它类似于下面的循环过程：
```
my @new_arr1;
foreach (1..100) {
    push @new_arr1,$_ if $_ % 2;
}

print "@new_arr1";
```

在grep中，每次从列表中取得一个元素，`$_`将引用这个元素，然后执行代码块(此处的`$_ % 2`和`$_ % 2 eq 0`)，如果返回真，则放进新的列表中。


其中grep后面的大括号为一段要执行的代码块，里面可以有任何想要实现的逻辑。如果要执行的代码块只有一句话(只有一个操作)，则可以省略大括号，使用逗号来传递参数，注意参数不要加引号。例如改写为：
```
@new_arr1 = grep $_ % 2,(1..100);
print "@new_arr1";
```

### grep使用示例

在Linux中也同样有一个非常好用的grep工具，用来筛选文本非常方便。perl的grep函数同样能实现这些功能，且更强大。

例如，筛选文件(假如以参数的方式传递文件)中包含3个连续数字的行：
```
while (<>){
    chomp;
    @new_line = grep { /\d{3,}/ } $_;
    print "@new_line\n";
}
```
如果my.txt文件内容为：
```
aaa,bbb,123,ccc,ddd
eee,fff,456,ggg,hhh
iii,jjj,kkk,lll,mmm
ooo,ppp,qqq,888,xxx
```

执行上述程序，将输出：
```
aaa,bbb,123,ccc,ddd
eee,fff,456,ggg,hhh

ooo,ppp,qqq,888,xxx
```

上面的程序代码中将`$_`交给grep处理，这个`$_`被当作是个只有单个元素的列表，于是对整行内容进行匹配，匹配到有3个数字连续的，就赋值给数组，然后被输出。当某行匹配不到3个连续数字时，数组将被赋值为undef。

如果要去掉上面的空行，可以在grep的代码块里加个判断逻辑：
```
while (<>){
    chomp;
    @new_line = grep { next unless /\d{3,}/ } $_;
    print "@new_line\n";
}
```
将输出：
```
aaa,bbb,123,ccc,ddd
eee,fff,456,ggg,hhh
ooo,ppp,qqq,888,xxx
```

注意，grep返回的是一个新的列表，所以将其放在标量上下文将返回新列表的元素个数。所以，`@new_line`不能替换为`$line`，尽管每个列表只有一个元素。


如果只想取出被匹配的3个连续数字，其它字符不要，则可以：
```
while (<>){
    chomp;
    grep { 
            next unless /(\d{3,})/;
            $line=$1;
         } $_;
    print "$line\n";
}
```

相对来说，这里可以不用grep来是实现，它其实是多余的。grep的主要目的还是留下列表中匹配到的特定元素，所以，采用grep的行为模式，来留下连续数字。

由于本处给的文件比较规律，可以将每行转换成列表：
```
 while (<>){
    chomp;
    @line = split /,/,$_;
    @new_line = grep { /\d{3,}/ } @line;
    print "@new_line\n" if @new_line;
}
```

如果文件数据不规律，无法转换成列表，则不建议使用grep来操作。


### grep使用注意事项

- grep的代码块中最好只做条件匹配操作，因为它要返回哪些元素是根据布尔值来判断的  
- grep将列表中的每个元素引用赋值给`$_`，它可以直接修改原始数据，例如正则的s替换，如无特殊需求，不建议修改原始数据  
- grep代码块中能不操作元素就不要操作元素，能不加逻辑就不加逻辑  
- **在grep的代码块中做循环控制是不明智的**，除非待操作列表是单元素的列表。见下面示例分析  
- 如有需求，可以将逻辑控制放在grep的外面  
- 如果待操作的目标无法转换成列表，则不建议使用grep，它的优点毕竟体现在操作列表上。而且，任何形式的grep都能通过其它循环的方式来实现，只不过需要借助中间变量来保存临时结果

例如：

```
while (<>){
    chomp;
    @line = split /,/,$_;
    @new_line = grep { next unless /\d{3,}/ } @line;
    print "@new_line\n";
}
```

原意本是匹配`@line`中的3个连续数值，如果没有就跳过。但是问题出在next和unless上，如果`@line`第一个元素就不匹配，这一个列表就直接跳过了，它并不会去测试后面是否还有能匹配的元素，甚至第一个元素就算符合匹配，也会因为grep的【中断】而未产生新的列表，也就没有成功去赋值。




<a name="blogmap"></a>

## map

map和grep的行为非常相似，也是对列表的每个元素(赋值给`$_`)执行一段给定的代码块(一般来说是一个子程序或函数)，并返回执行结果。

与grep不同的是：  
- map的返回结果是代码块执行后的结果组成的列表，而不像grep是通过真假值来返回元素的  
- map对列表中的每个元素都执行一段代码并返回执行结果，然后继续迭代下一个元素。中间的流程控制我们无法控制，所以很可能会出现各元素处理后的【缝接】不当问题。见下文示例即可体会

需要注意，任何map和grep能实现的操作，都能通过其它循环结构来实现，只不过其它循环结构需要借助临时变量来保存临时结果。

例如，将列表中的每个元素复制3次：
```
@str=qw(abc def ghij);
print map {$_ x 3} @str;
```
执行结果：
```
abcabcabcdefdefdefghijghijghij
```
上面的map首先对`@str`的第一个元素复制3份，然后第二个元素、第三个元素都复制三份，各复制3份后，作为一个列表返回给上下文。也就是说，上面的map实际返回的是如下列表：
```
@temp_arr= qw(abcabcabc defdefdef ghijghijghij);
```

看上去没什么问题，但想要控制map对每个元素操作后的结果，却不那么容易，比如想让`@temp_arr`的每个元素换行输出。
```
@str=qw(abc def ghij);
print map {($_ x 3)."\n"} @str;
```
或者，用sprintf：
```
@str=qw(abc def ghij);
print "haha:\n",map {sprintf("%s\n",$_ x 3)} @str;
```
或者直接在map内部使用print：
```
@str=qw(abc def ghij);
map {print $_ x 3,"\n"} @str;
```

本想只是复制3份，结果却多出一些其它的逻辑和操作。但没办法，map就是有这样的**元素缝接问题**。

当map的代码块只有一条语句(一个操作)时，可以省略代码块的大括号，用逗号来传递参数，例如：
```
map $_ x 3,@str;
```

### map丰富的特性

map的块结构中可以有更复杂的逻辑控制，而不像grep只建议在块结构中放匹配条件。实际上，grep能实现的，map都能实现，map能实现的，都能转换为循环结构来实现，只不过用grep、map更简洁、易读。

例如，筛选出数值列表中每个以4结尾的数值。
```
my @nums=qw(12 34 56 14 25 64 32 84);
my @end_by_4=map {
    $_ if /4$/;
} @nums;
```

注意，map每次迭代逻辑中，返回值和子程序类似，直接将值赋值出去即可，无需return的形式。正如上面的`$_`，它前面并没有任何操作。再例如，在map中进行一些判断，某个判断分支中，将空列表赋值给左值：
```
my @result =map {
    if(...){
        $_;  # 将$_赋值给@result
    }else{
        ();  # 将空列表赋值给@result
    }
} @arr;
```

**map还可以在迭代每一个元素的时候，产生多元素的结果，比如列表、数组、hash，这种功能在某些情况下，作用巨大**。

例如，每次迭代一个元素的时候，却产生两个元素保存到数组中：
```
use 5.010;
my @name=qw(ma long shuai gao xiao fang);
my @new_names=map {$_,$_ x 2} @name;
say "@new_names";
```
当迭代第一个元素的时候，将生成(ma mama)，保存到数组`@new_names`中，迭代第二个元素的时候，将生成(long longlong)，保存到数组`@new_names`中。所以，最后`@new_names`的结果是：
```
ma mama long longlong shuai shuaishuai gao gaogao xiao xiaoxiao fang fangfang
```
同样，还可以对每个列表中的元素生成3个元素、4个元素，等等。

正因为如此，当对每个元素生成两个元素的时候，可以将其赋值给hash：
```
use 5.010;

my @name=qw(ma long shuai gao xiao fang);
my %new_names=map {$_,$_ x 2} @name;
while (my ($key,$value)=each %new_names){
    say "$key --> $value";
}
```
输出结果：
```
long --> longlong
xiao --> xiaoxiao
gao --> gaogao
ma --> mama
shuai --> shuaishuai
fang --> fangfang
```

有些时候，会比较两个列表或hash，看看某个列表中的哪些元素存在于另一个列表中，哪些元素不存在于另一个列表中(例如A列表中的元素，是否存在于B中)。这时可以使用map，将目标列表(即列表B)改换成hash，因为hash有exists()函数。
```
my @name=qw(ma long shuai gao xiao fang);
my @name1=qw(ma long gao xiao);
my %name_hash=map {$_,1} @name;
foreach (@name1){
    print $_,"\n" if exists $name_hash{$_};
}
```

其实可以不用exists函数，因为`$HASH{KEY}`如果KEY不存在时，将直接返回undef，所以可以作为条件：
```
my @name=qw(ma long shuai gao xiao fang);
my @name1=qw(ma long gao xiao);
my %name_hash=map {$_,1} @name;
foreach (@name1){
    print $_,"\n" if $name_hash{$_};
}
```

上面，为hash的每个key赋值数值1。因为这个时候只需要用到hash的key，value并没有任何用处，赋值为1只是为了将列表构建成hash形式而加入的。

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

**标量上下文下**：反转字符串，即使反转目标是列表
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



<a name="blogsort"></a>

## sort

sort函数可以对列表进行排序，基本用法很简单。它的强大之处在于，我们可以自定义排序规则(比如忽略大小写)，然后交给sort。相当于sort给我们提供了一个接口。

```
sort SUBNAME LIST     # (1).指定使用某个排序子程序来实现排序
sort CODE_BLOCK LIST  # (2).自定义排序代码块
sort LIST             # (3).默认排序方式：ASCII
```

注意，sort并没有定义标量上下文中排序的行为。

另外，sort不是直接作用在原始列表中的，而是返回一个新的有序列表。

### 默认排序

首先看`sort LIST`的排序用法。

它是默认的排序规则，默认是按照ASCII的排序方式进行排序，返回每个元素的从小到大的有序列表。

可以使用`use locale`来指定对应的排序规则，而非使用默认ASCII排序方式。

以下是需要了解的ASCII顺序：从上到小，顺序依次变大
```
最小：空值(\0,undef,null等)

制表符(\t)
换行符(\n)
空格(space)
某些标点符号(主要考虑的是负号 - )
数字(0-9)
大写字母(A-Z)
小写字母(a-z)
```

例如：
```
@str=qw(abc Abc ABc 123);
@sorted=sort @str;
print "@sorted";        # 123 ABc Abc abc
```

### 使用排序子程序进行排序


用法是这样的：
```
sort subname list;
```

subname就是我们自定义的排序子程序。

关于排序子程序，sort处理方式如下：  
- 从列表中取出两个元素，分别赋值给子程序中**固定的已预定义**的两个特殊变量`$a`和`$b`  
- 在子程序的代码中，需要比较`$a`和`$b`：  
    - 如果`$a < $b`，即`$a`在左，`$b`在右，则需要让这段比较代码返回-1  
    - 如果`$a > $b`，即`$a`在右，`$b`在左，则需要让这段比较代码返回1  
    - 如果`$a = $b`，则需要让这段比较代码返回0  
    - 子程序中如果`$a`写在操作符左边，则表示从小到大的排序，如果`$b`写在左边，则表示从大到小的顺序  
- sort根据子程序返回的值(`-1 1 0`)来决定值的前后位置，也就是大小顺序  

必须注意，`$a`和`$b`并非来自列表元素的拷贝，而是直接引用元素，这样可以提高效率。也因此，不建议中途对列表元素进行修改，这可能会产生意想不到的结果。

例如，按照数值大小进行比较，可以定义如下子程序：
```
sub by_num {
    if ($a < $b) {
        -1
    } elsif ($a > $b) {
        1
    } else {
        0
    }
}
```

然后在sort中调用这个排序子程序，注意，需要去掉调用子程序时的`&`符号。
```
@sorted = sort by_num 1,2,44,22,33,10;
print "@sorted";        # 1 2 10 22 33 44
```

在perl中提供了两个对sort排序子程序非常友好的比较操作符：`<=>`和`cmp`，前者用于比较数值，后者用于比较字符串。它们都是三值返回逻辑：  
- 操作符左边的小于右边的，返回-1  
- 大于右边的返回1  
- 等于则返回0  

这正好符合sort子程序的定义规则。

所以，改写上面的排序子程序，会非常的简单：
```
sub by_num { $a <=> $b }
```

sort中使用`<=>`和`cmp`有两点需要注意：  
- `<=>`比较的是两个数值，如果有一方不是数值，将返回undef，在sort中，它们被当作最小值  
    - 如果是正向排序，则非数值排在最前面  
    - 如果是逆序排序，则非数值排在最后面  
- cmp的排序规则和sort的默认规则是一样的，不指定locale时是ascii排序规则，指定locale时则按照对应的顺序

所以，大多数时候我们不需要专门使用cmp操作符，但有时候需要使用它实现特殊需求，如不区分大小写的排序：
```
sub by_caseinsensitive { "\L$a" cmp "\L$b" }
```

如果要实现逆序排序(即从大到小)，只需将子程序中`$a`和`$b`的位置对调即可。

```
sub by_num { $b <=> $a}
```

当然，对sort后的结果使用reverse也可以实现逆序排序，它并不会像想象中那样会降低性能。perl已经将reverse优化为sort的修饰符(例如，将原来`$a`引用的位置，换成`$b`去引用)，不会有多少额外的逆序行为。

### 使用排序代码块进行排序

事实上，应该没人去专门写一个排序子程序，大多数时候都是将排序规则直接内嵌在一个代码块中。当然，如果排序规则比较复杂，则应该写排序子程序。

```
@sorted = sort {$a <=> $b} @list;
```


### 元素相等时的位置问题

当两个元素被判定为相等，它们的顺序如何？例如，忽略大小写对字符串进行排序时，`abc`是在`Abc`的左边还是右边？

sort的方式是保持列表中相等元素的原始位置，无论是正序还是逆序排序。

例如：
```
@str=qw(Abc bbb bde abc);
@sorted1 = sort {"\L$a" cmp "\L$b"} @str;   # 正序排序
@sorted2 = sort {"\L$b" cmp "\L$a"} @str;   # 逆序排序
print "@sorted1","\n";
print "@sorted2","\n";
```

上面`@sorted1`和`@sorted2`中，Abc都将在abc的左边先出现。

### sort使用示例1：按指定字符进行排序

例如，从指定第三个字符处开始排序，第三个相等则排序第四个。

因为要略过前两个字符，所以需要对每个元素都进行子串提取，取出第三个以及后面的字符，再进行排序。

以下是排序代码的实现：
```
@str = qw(Abxx bbcda bdef ab);
@sorted = sort {substr($a,2) cmp substr($b,2)} @str;
print "@sorted";
```

其实很好理解，sort取出两个元素，分别用`$a`和`$b`去引用它们。当排序代码中`$a`所在的表达式(上例中在cmp操作符的左边)小于`$b`所在的表达式时，就返回-1。sort不会管这个-1是怎么来的，它只会根据-1的值对`$a`和`$b`进行排序。

所以，对于元素`Abxx`和`bbcda`，后者第三个元素小于前者第三个元素，sort会将后者排在前者的前面。


### sort使用示例2：对hash进行排序

sort并未提供对hash的排序方式。但是可以采取一些技巧，就像上面的例子一样。

例如，存放姓名和工资的hash，想要按照他们的工资进行排序，但输出的是他们的姓名。也就是说，根据hash的value进行排序，但输出的是对应的key。

可以先使用keys获取key列表，再通过`$hash_name{key_name}`对每个value作比较，从而得到key的顺序。

```
%name_salary = (
                malong => 8000,
                xiaofang => 9000,
                longshuai => 6000,
                woniu => 10000);

@sorted_key = sort { 
                    $name_salary{$a} <=> $name_salary{$b}
                   } keys %name_salary;

print "@sorted_key";  # 输出：longshuai malong xiaofang woniu
```

如果，hash的某两个或几个value值相等，例如工资都是6000，该如何显示？这时可以采取默认的顺序，也需要指定一个决胜属性，明确谁先谁后。

### 值相等时的决胜属性


如下hash：其中wugui的值和longshuai的值相等。
```
%name_salary = (
                malong => 8000,
                wugui => 6000,
                xiaofang => 9000,
                longshuai => 6000,
                woniu => 10000);
```

在按照value进行排序的时候，最终输出时是wugui在前还是longshuai在前？

默认情况下，由于keys获取的key顺序是我们人为无法确定的，所以并不能总是清楚地确定key的先后顺序。但至少，当值相等的时候，先被抓取的元素总是先出现的。
```
@sorted1 = sort {
                    $name_salary{$a} <=> $name_salary{$b}
               } keys %name_salary;

@sorted2 = sort {
                    $name_salary{$b} <=> $name_salary{$a}
               } keys %name_salary;

print "@sorted1","\n";    # longshuai出现在wugui之前
print "@sorted2","\n";    # longshuai出现在wugui之前
```

其实，我们更希望在两个值相等的时候，能自己定义出现的先后顺序。这时就需要指定另一个用于决胜的比较行为(加时赛)。

例如，当工资相等，就按照姓名的大小顺序进行正向排序：
```
@sorted = sort {
                    $name_salary{$a} <=> $name_salary{$b}
                    or 
                    $a cmp $b
               } keys %name_salary;

print "@sorted";
```

注意，or操作符是一个短路操作符，且返回的是表达式的值。所以，当value不等的时候，or前面的返回1或-1，它们都表示true，于是短路直接返回给sort；当value相等的时候，or前面的返回0，它表示false，于是比较or后面的，同样返回1、-1、0给sort。

sort还可以指定更多层次的决胜属性，所以，sort几乎可以实现任何形式的排序。

### sort排序文本数据

unix操作系统中有一个sort命令，它是一个非常完美、且效率非常高的文本排序工具。

在perl中，sort默认只是对列表内元素进行排序。要想用perl对文本数据进行排序，除了调用unix中的sort命令，perl自身的sort也可以实现，但处理大文件时，如果不做一些处理，效率会比较差。而且，unix的sort本身已经足够完美，如果真的有需求，完全可以调用它来排序。

以下是一个简单的写法，将所有数据一次性加载并保存到列表中，然后进行排序。这样的缺点是内存占用较大。

```
print sort { $a cmp $b } <>;
```

