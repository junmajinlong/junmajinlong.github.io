---
title: Perl函数：List::Util模块用法详解
p: perl/perl_list_utils.md
date: 2019-07-07 17:39:45
tags: Perl
categories: Perl
---

# Perl函数：List\:\:Util模块用法详解

本文介绍Perl标准库`List::Utils`中的列表工具，有时候它们非常好用。比如Perl中测试列表中是否包含某个元素(某个元素是否存在于列表中)没有比较直接比较方便的功能，但使用`List::Utils`中的first或any函数，则非常方便且高效。此外，该模块都有对应的C代码的函数，所以它们的效率也不差。

可以将`List::Utils`模块中的功能大致分为3类：reduce类、key/value类以及其它类：
- `reduce类`：reduce类的函数是依次取(迭代)列表中的元素，并对这些元素做一些操作，直到列表中元素个数减为1或0个。用法和grep/map函数类似  
- `key/value类`：reduce类是每次取一个元素进行操作，key/value类函数是每次取连续的两个元素作为key/value对进行操作  
- 其他类中包括打乱列表顺序的shuffle函数，去重函数uniq、uniqnum和uniqstr  

<a name="blog1548814864"></a>
## Reduce类函数

有不少reduce类的工具，reduce()是这些工具的通用化函数，其它函数都是reduce()的特定情况改编而来。

<a name="blog1548814865"></a>
### reduce()

```perl
$res = reduce {block} @list
```

reduce()函数首先从列表`@list`中取出两个元素赋值给`$a`和`$b`，对这两个变量应用block代码得到一个结果，这个结果赋值给`$a`，再从`@list`中取下一个元素赋值给`$b`，继续与`$a`应用block代码。直到所有元素都应用完了block代码，最后返回block的结果。

说起来有点绕口，举个例子：
```perl
my $max = reduce { $a > $b ? $a : $b} qw(10 5 2 20 14);
```

上面的代码表示从`@list`中取出值最大的元素。上面的处理过程大概是这样的：首先从列表中取出2个元素(即10和5)赋值给`$a`和`$b`，然后执行大括号中的代码逻辑，这段代码逻辑要么返回`$a`，要么返回`$b`，它的返回值(即较大值10)再次赋值给`$a`，然后继续从list中取出另一个元素(即2)赋值给`$b`，`$a`和`$b`再次进行比较取出较大者(即10)，依此类推，最终取出最大的元素(即20)，赋值给`$max`。

再比如计算列表中数值的总和：
```perl
my $sum = reduce { $a + $b } 1..10;
```

**如果`@list`是空列表，则reduce返回undef，如果`@list`是只有一个元素的列表，则直接返回这个元素，不会评估block**。

为了编写健壮的reduce()，经常使用**reduce的一种通用技巧：为了避免列表元素数量的不足，使用一个或多个额外的不影响结果的元素压入到目标列表中**。例如：
```perl
reduce {BLOCK} 0,@list;
reduce {BLOCK} 1,@list;
reduce {BLOCK} undef,@list;
reduce {BLOCK} undef,undef,@list;
```

例如，计算列表中元素的总和：
```perl
my $sum = reduce {$a+$b} 0,@list;
```

<a name="blog1548814866"></a>
### any

any是reduce类的一个工具，用于判断列表中是否存在某个符合条件的元素，如果存在则立即退出。它有点像grep，但是它比grep在元素判断上更优化，因为grep总会迭代所有元素，而any在得到结果时会立即退出。

语法：
```perl
any {BLOCK} @list
```

any()将依次迭代列表中的元素并赋值给`$_`，然后应用BLOCK，如果某个元素能使BLOCK返回true，则any()直接返回true并退出any。

如果所有元素都不能使得block返回true，则any返回false，或者`@list`为空列表则也返回false.

在Perl中没有比较直接的方式来判断某个元素是否存在于列表中，但使用any会非常简便且高效(后面介绍的first()也可以判断)。例如，判断`xyz`是否存在于数组中：
```perl
if(any {"xyz" eq $_} @list){
    print qw('"xyz" exists in @list')
}
```

与之等价的grep()判断为:
```perl
if (grep {"xyz" eq $_} @list){
    print qw('"xyz" exists in @list')
}
```

前面说了，grep判断的效率要低一些，不管元素是否存在，它总会迭代完列表所有元素才退出，而any在第一次判断存在的时候就退出。

<a name="blog1548814867"></a>
### all

```perl
all {BLOCK},@list;
```

如果`@list`中所有元素都应用于BLOCK都返回true，或者`@list`为空，则all()返回true，否则返回false。

<a name="blog1548814868"></a>
### none和notall

语法：
```perl
none {BLOCK} @list;
notall {BLOCK} @list;
```

none()是any()的反意，notall()是all()的反意。

none()表示如果`@list`中所有元素应用于BLOCK时都不返回true，则none()返回true。

notall()表示`@list`中并非所有元素应用于BLOCK时都返回true，则notall()返回true。如果没有元素返回True，则也返回true。所以，notall是all的反意。

<a name="blog1548814869"></a>
### first

语法：
```perl
first {BLOCK} @list;
```

迭代list中的每个元素，返回第一个应用于BLOCK返回true的元素。注意，只要发现了一个满足条件的元素，就立即停止迭代，这和any()是类似的。

如果所有元素都返回false，或者list为空，则first返回undef。

```perl
# 返回list中第一个已定义的元素
$foo = first {defined($_)} @list;

# 返回list中第一个大于某值的元素
$foo = first { $_ > 23 } @list;

# 判断list中是否存在某个元素
if (first { $_ eq "abc" } @list) {...}
```

<a name="blog1548814870"></a>
### max和maxstr

max和maxstr分别用来取出列表中的最大数值和最大字符串。如果列表为空，则返回undef。

```perl
max @list
maxstr @list
```

例如：
```perl
max qw(1 3 10 5 11 12);
max @list1, @list2;
```

<a name="blog1548814871"></a>
### min和minstr

min和minstr分别用来取出列表中的最小数值和最小字符串。如果列表为空，则返回undef。

```perl
min @list
minstr @list
```

<a name="blog1548814872"></a>

### product

返回列表中所有元素的乘积。
```perl
product 3,4,5  # 60
product 1..10  # 3628800
```

<a name="blog1548814873"></a>

### sum和sum0

sum和sum0都计算列表中所有元素的和，不同的是前者在列表为空时返回undef，而后者在列表为空时返回0。

<a name="blog1548814874"></a>
## key/value类函数

以key/value成对的方式迭代列表，所以列表元素的个数需要是偶数。

<a name="blog1548814875"></a>
### pairs和unpairs

pairs用法：
```perl
@each = pairs @list;
print "@{$each[0]}";
print "@{$each[1]}";
print "@{$each[2]}";
print "@{$each[3]}";
...
```

这个函数从一个列表中每次取两个元素，分别作为key/value(Perl中key/value也一样是列表)放进一个数组中，并返回这个数组的引用。所以，每次迭代时返回的数组引用中都只包含两个元素。

例如：
```perl
my @arr = qw(one 1 two 2 three 3);
my @pairs = pairs @arr;
```

上面的数组`@pairs`中包含了3个数组的引用：
```perl
$" = "\n";
print "@pairs";
```
结果：
```perl
List::Util::_Pair=ARRAY(0x64ad30)
List::Util::_Pair=ARRAY(0x6549a8)
List::Util::_Pair=ARRAY(0x654a20)
```

这里的每个数组引用所指向的数组都只包含2个元素，这两个元素来自于`@arr`中每次取出的两个元素。例如，解除每个数组引用：
```perl
print "@{$pairs[0]}";  # one 1
print "@{$pairs[1]}";  # two 2
print "@{$pairs[2]}";  # three 3
```

因为pairs每次都返回一个数组引用，可以去迭代pairs的结果。且每个数组引用对象都可以使用key和value方法从这个2元素的数组中取出key和value：
```perl
foreach my $pair (pairs @arr) {
    print "@{$pair}";
}

foreach my $pair (pairs @arr) {
    print $pair->key;     # 从每个数组引用中取出key
    print $pair->value;   # 从每个数组引用中取出value
}
```

unpairs()和pairs()作用正好相反，pairs()是将一个列表中的每两个元素组成一个key/value对的数组引用，unpairs()则是将一个包含每两个元素的数组引用的数组解除成列表。它在结果上等价于：
```perl
my @arr = map { @{$_}[0,1] } @pairs;
```

对于`qw(one 1 two 2 three 3)`列表，pairs()的结果是构造下面的数组，unpairs()则是将`@pairs`这样的结构还原成一个list。
```perl
@pairs = ( ["one", 1], ["two", 2], ["three", 3] );
```

在pairs()转换和unpairs()还原的过程中间，可以加上一些函数来操作key/value对。例如：
```perl
# 类似于后面的pairgrep
my @arr = unpairs grep {FUNC} pairs @arr;

# 类似于后面的pairmap
my @arr = unpairs map {FUNC} pairs @arr;
```

还可以将每个key/value对使用sort进行排序。例如按照key排序：
```perl
my @arr = unpairs sort {$a->key cmp $b->key} pairs @arr;
```

<a name="blog1548814876"></a>
### pairkeys和pairvalues

分别返回pairs得到的key/value对的key和value列表，也就是分别取得pairs操作列表的第奇数个元素和第偶数个元素。

例如：
```perl
use List::Util qw(pairs pairkeys pairvalues);
my @arr = qw(one 1 two 2 three 3);

my @keys = pairkeys @arr;  # one, two, three
print "@keys";

my @values = pairvalues @arr;  # 1 2 3
print "@values";
```

<a name="blog1548814877"></a>
### pairgrep

```perl
@arr = pairgrep {BLOCK} @list;
$count = pairgrep {BLOCK} @list;
```

和Perl的grep函数类似，不同的是pairgrep筛选的对象是key/value对，每次迭代key/value对的时候，都会分别设置变量`$a`和`$b`为key和value。

在列表上下文中，它返回的是满足条件key/value对，它会直接还原成列表中的元素，所以返回的列表中包含连续的key和value。

在标量上下文中，则是返回满足条件的key/value对的数量。所以标量上下文的返回数量比列表上下文中元素的数量少一半。

具体看下面的示例。

例如，筛选列表中key为大写字母的key/value对：
```perl
use List::Util qw(pairgrep);
my @testarr = qw(ab cd EF gh AB CD ef GH);

my @upp = pairgrep { $a =~ m/^[[:upper:]]+$/ } @testarr;
print "@upp";   # EF gh AB CD

my $count = pairgrep { $a =~ m/^[[:upper:]]+$/ } @testarr;
print "$count";  # 2
```

<a name="blog1548814878"></a>
### pairfirst

```perl
my @pair = pairfirst {BLOCK} @arr;
my ($key, $val) = pairfirst {BLOCK} @arr;
my $found_or_not = pairfirst {BLOCK} @arr;
```

类似于first()函数，但是pairfirst操作的是key/value对。它和pairgrep一样，同样会设置`$a`和`$b`为每个key/value对中的Key和value。

在列表上下文，它返回list中的第一个满足条件的key/value对，它是一个2元素的数组。如果没有满足条件的则返回一个空列表。在标量上下文中，它简单地返回true/false而不是第一个key/value对的值，只要找到了就返回true。
```perl
use List::Util qw(pairgrep pairfirst);
my @testarr = qw(ab cd EF gh AB CD ef GH);

my @upp = pairfirst { $a =~ m/^[[:upper:]]+$/ } @testarr;
print "@upp";

my $found_or_not = pairfirst { $a =~ m/^[[:upper:]]+$/ } @testarr;
print "$found_or_not";
```

<a name="blog1548814879"></a>
### pairmap

```perl
my @list = pairmap {BLOCK} @arr;
my $count = pairmap {BLOCK} @arr;
```

类似于Perl的map函数，但是它操作的是key/value对。同样地，它会将每个key/value对中的key和value设置为`$a`和`$b`。

在列表上下文，它返回block对每个key/value的评估结果，在标量上下文，它直接返回对应于列表上下文列表元素的个数。


```perl
use List::Util qw(pairgrep pairfirst pairmap);
my @testarr = qw(ab cd EF gh AB CD ef GH);

# 每次迭代key/value对时，新列表中都有两个新构成的元素
my @upp = pairmap { "$a: $b","$a$b" } @testarr;
print "@upp";

# 上面的结果列表中有8个元素，所以这里返回8
my $count = pairmap { "$a: $b","$a$b" } @testarr;
print "$count";
```

返回结果：
```perl
ab: cd abcd EF: gh EFgh AB: CD ABCD ef: GH efGH
8
```

<a name="blog1548814880"></a>
## 其它函数

<a name="blog1548814881"></a>
### shuffle

打乱给定列表中元素的顺序。
```perl
my @values = shuffle @values;
```

例如：
```perl
$ perl -MList::Util=shuffle -e '$,=" "; print shuffle 1..9'
4 3 9 7 6 5 8 2 1
```


<a name="blog1548814882"></a>
### uniq

去除列表中的重复元素。只有连续的相同的两元素才认为是重复的，如果有连续的重复元素，则保留第一个重复元素。

标量上下文下返回去重后的元素个数。

例如：
```perl
my @tsuniq = qw(ab ab AB 1 1 2 2.0 3 );
my @subset = uniq @tsuniq;  # ab AB 1 2 2.0 3
my $count = uniq @tsuniq;   # 6
```

需要注意的是，undef和空字符串是不同的，但是连续的undef被认为是相同的，所以只会保留一个undef。

<a name="blog1548814883"></a>
### uniqnum

对列表中的数值进行去重操作。1.0和1被认为是相同的数值元素。同样的，只有连续的重复元素才算是重复。

同样的，在标量上下文返回的是去重后的元素个数。

对于undef元素，它与0比较是重复的元素，如果0在undef的前面，保留0，如果undef在0的前面，则保留undef。如果开启了警告信息(`use warnings 'uninitialized';`)，会对undef给出警告。

```perl
use List::Util qw(uniqnum);
my @tsuniq = qw( 1 1 2 2.0 3 undef undef 0 );
my @subset = uniqnum @tsuniq;  # 1 2 3 undef
my $count = uniqnum @tsuniq;   # 4
```

如果是对`qw(1 1 2 2.0 3 0 undef undef)`或`qw(1 1 2 2.0 3 undef 0 undef)`进行去重，则去重时保留undef而不是0。

<a name="blog1548814884"></a>

### uniqstr

类似于uniqnum，只不过undef元素和空字符串被认为是相等的，保留undef还是空字符串的规则同uniqnum。
