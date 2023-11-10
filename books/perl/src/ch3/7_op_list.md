## 列表常见操作

列表常见操作包括：grep、join、map、reverse、sort、unpack、x操作符执行列表重复，等等。另外，标准库`List::Utils`中也提供了很多常见的列表操作，如reduce、first、any、sum、uniq、shuffle等。

多数时候，数组可以当作列表来使用，原因是操作列表的地方期待一个列表，即处在列表上下文，perl会隐式地将数组转换为列表。因此上述列表操作多数也适用于数组。

限于篇幅问题，本文只介绍Perl内置的列表操作函数，`List::Utils`中提供的操作可自行查阅手册或查看我的博客文章[List::Util模块用法](https://junmajinlong.com/perl/perl_list_utils)进行了解。

### 列表重复：x

使用小写字母`x`可以重复列表指定次数：  

```perl
# @arr = (1,2,1,2,1,2)
my @arr = (1, 2) x 3;
```

列表重复通常用于初始化构建一个特定大小的数组，也常用于生成测试数据。例如：  

```perl
# 创建包含100个undef元素的数组
# 等价于$arr[99] = undef;
my @arr = (undef) x 100;

# 生成一个大数组(20W个元素)，用于某些测试
my @test_data = (11,22) x 100000;
```

一定要注意，不能将操作符`x`用于数组，因为`x`会被解析成字符串重复操作，使得数组处于标量上下文，然后进行字符串重复。例如：  

```perl
my @arr = (11,22);
say @arr x 3;  # 输出：222
```

如果需要对数组进行重复，将它放进小括号转换为列表即可：

```perl
my @arr = (11,22);
@arr = (@arr) x 3;
```

### join

用给定字符将列表中各元素连接起来，返回连接后的字符串。

join语法：

```
join $sep,$list
```

例如：

```perl
say join "-",qw(a b c d e);   # 输出："a-b-c-d-e"
```

<a name="perl_split"></a>

### split

使用给定分隔符将字符串划分为列表，分隔符支持使用正则表达式。

在列表上下文，返回划分后得到的列表，在标量上下文，返回划分后列表的元素数量。

语法：

```
split /PATTERN/,EXPR,LIMIT
split /PATTERN/,EXPR
split /PATTERN/
split
```

例如：

```perl
my $str="abc:def::123:xyz";
my @list = split /:/,$str;
say join ',', @list;   # abc,def,,123,xyz

my $str="abc:def::12:xyz";
my @list = split /::/,$str);     # 返回："abc:def","12:xyz"
my @list = split /[:]+/,$str);   # 返回："abc","def","12","xyz"
my @list = split /[:0-9]/,$str); # 返回："abc","def","","","","","xyz"
```

可以加上一个limit参数，限制最多分隔为多少个元素。

例如，指定limit=2，表示只分隔一次：

```perl
my $str="abc:def::123:xyz";
my @list = split /:/,$str,2;   # 返回"abc","def::123:xyz"两个元素
```

省略limit时，默认limit=0，表示尽可能多地划分元素，且忽略后缀空元素，但会保留前缀空元素。limit为负数时，几乎等价于limit=0，但不忽略后缀空元素。例如：

```perl
my $str=":::abc:def:123:xyz::::";
my @new_list1=join(".",split /:/,$str);
my @new_list2=join(".",split /:/,$str, -1);

say "@new_list1";   # ...abc.def.123.xyz
say "@new_list2";   # ...abc.def.1234.xyz....
```

省略字符串参数时(意味着也必须省略limit)，split默认对`$_`进行划分：

```perl
split /:/;   # 等价于 split /:/, $_;
```

对于split，除了常规用法，更重要的是要记住它的特殊用法：

- 将pattern指定为空格`" "`时(注意，不是正则里的空格`/ /`)，和awk的行为一样：忽略前缀空白，且将一个或多个空白作为分隔符  

  ```perl
  my $str = "  a  b    c   ";
  my @arr = split " ", $str;
  say join ",", @arr;   # a,b,c
  ```

- 省略pattern时(意味着后面其他参数也被省略)，即不带任何参数的split，默认pattern为空格`" "`，对`$_`变量进行划分  

- 将pattern指定为`//`时(空正则表达式)，字符串的各字符都被划分  

  ```perl
  my $str = "abc";
  my @arr = split //, $str;
  say join ",", @arr;   # a,b,c
  ```

### grep

从列表中筛选符合条件的元素，在列表上下文返回符合条件的元素列表，在标量上下文中返回符合条件的元素数量。

```
grep BLOCK LIST
grep EXPR, LIST
```

例如，筛选列表中的偶数和奇数：

```perl 
my @nums = (11,22,33,44,55,66);
my @odds = grep {$_ % 2} @nums;   # 取奇数
my @evens = grep {$_ % 2 == 0} @nums;  # 取偶数
say "@odds";
say "@evens";
```

grep会迭代列表中的每一个元素，并将这些元素逐次【赋值】给默认变量`$_`，在给定的语句块BLOCK中可以使用该默认变量，当BLOCK中的代码评估结果为布尔真，则将本次迭代的元素放进返回值列表中等待被返回。

当BLOCK中只有一条语句或一个表达式时，可以使用`grep expr,list`语法。例如，上面示例的等价写法：

```perl
grep $_ % 2, @nums;
grep $_ % 2 == 0, @nums;
```

注意，grep在迭代列表各元素时，`$_`是各元素的别名引用，在代码块中修改`$_`，也将影响到源列表，也因此会影响返回值列表。

```perl
my @nums = (11,22,33,44,55,66);
my @arr = grep {$_++; $_ % 2} @nums;
say "@arr";     # 23 45 67
say "@nums";    # 12 23 34 45 56 67
```

### map

语法：

```
map BLOCK LIST
map EXPR, LIST
```

map迭代列表的每个元素，并将表达式或语句块中返回的值放进一个列表中，最后返回这个列表。

例如：

```perl
my @chars = map(chr, (65..70));
say "@chars";  # A B C D E F

my @arr = map { $_ * 2 } (1..5);
say "@arr";  # 2 4 6 8 10
```

当语句块中只有一条语句时，可使用表达式写法。如

```perl
my @arr = map $_*2, (1..5);
say "@arr";  # 2 4 6 8 10
```

同grep一样，map迭代每个元素时，`$_`是这些元素的别名引用，修改`$_`将会修改元素原始数据。

注意，Perl map不是完全等量映射，不一定会返回和原列表元素数量相同的列表。特别地，**如果语句块中返回空列表`()`，相当于没有向返回列表中追加元素**。例如：

```perl
my @arr = (11,22,33,44,55);
# @evens = (undef,22,undef,44,undef)
my @evens = map {$_ if $_%2==0} @arr;

# @evens = (22,44)
my @evens = map {$_%2==0 ? $_ : ()} @arr;
# 等价于 map {$_} grep {$_%2==0} @arr;
```

并且，map允许在一个迭代过程中保存多个元素到返回列表中。

```perl
my @name=qw(ma long shuai);
my @new_names=map {$_,$_ x 2} @name;
say "@new_names";  # ma mama long longlong shuai shuaishuai
```

正因为map可以一次向返回列表中添加多个元素，因此可以每次迭代生成两个元素并将map返回值赋值给hash：

```perl
my @name=qw(ma long shuai gao xiao fang);
my %new_names = map {$_, $_ x 2} @name;

while (my ($key,$value) = each %new_names){
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

当map的BLOCK返回两个元素时，map的大括号可能会和构建匿名hash结构的大括号产生歧义。Perl会尽量根据规则取猜测大括号是map的语句块还是用于构建匿名hash的。

```perl
# 下面的大括号被猜错了，当作了匿名hash的构建大括号
# 等价于 map \%hash @array
my %hash = map {  "\L$_" => 1  } @array

# 给点提示，在第一个元素前使用`+`，使其不能作为hash的key
my %hash = map { +"\L$_" => 1  } @array
my %hash = map {; "\L$_" => 1  } @array # 这也可以
my %hash = map { ("\L$_" => 1) } @array # 这也可以
my %hash = map {  lc($_) => 1  } @array # 这也可以
my %hash = map +( lc($_) => 1 ), @array # 这也可以
my %hash = map  ( lc($_), 1 ),   @array # 评估为(1, @array)
```

### sort

sort用于对列表元素进行排序，返回排序后的列表。

```
sort SUBNAME LIST
sort BLOCK LIST
sort LIST
```

对于`sort LIST`语法，表示按照默认的字符串顺序进行排序(比如ASCII码顺序)。需要了解的几个顺序是：

```perl
最小：空值(0,undef,""等)

制表符(\t)
换行符(\n)
空格(space)
某些标点符号(主要考虑的是负号 - )
数字(0-9)
大写字母(A-Z)
小写字母(a-z)
```

例如：

```perl
my @str=qw(abc Abc ABc 123);
my @sorted=sort @str;
say "@sorted";     # 123 ABc Abc abc
```

对于`sort BLOCK LIST`语法，sort首先会从列表中取出两个元素，分别赋值给两个特殊的变量`$a`和`$b`(仍然是引用别名的关系，修改这两个变量将会影响原始元素)：  

- 若语句块返回-1，则表示`$a`对应的元素小于`$b`对应的元素，`$a`将排在`$b`的前面  
- 若语句块返回1，则表示`$a`对应的元素大于`$b`对应的元素，`$a`将排在`$b`的后面  
- 若语句块返回0，则表示`$a`对应的元素等于`$b`对应的元素，`$a`和`$b`的位置不变  

因此，可以编写如下代码对一串数字进行排序。

```perl
my @nums = (11,33,4,55,7,12);

# 升序排序
my @sorted_nums = sort {
  if($a<$b){
    -1
  } elsif($a > $b) {
    1
  } else {
    0
  }
} @nums;

say "@sorted_nums";  # 4 7 11 12 33 55
```

Perl提供了两个非常好用的运算符：

- `<=>`：用于比较数值，如果左边的数值小于右边的数值，则返回-1，大于则返回1，相等则返回0  
- `cmp`：用于比较字符串，规则和`<=>`相同  

因此，使用比较运算符来改写上面的升序排序：

```perl
my @nums = (11,33,4,55,7,12);

# 升序排序
my @sorted_nums_asc = sort {$a<=>$b} @nums;
# 降序排序
my @sorted_nums_desc = sort {$b<=>$a} @nums;
say "@sorted_nums_asc";  # 4 7 11 12 33 55
say "@sorted_nums_desc"; # 55 33 12 11 7 4
```

但是，在使用`<=>`时需要小心，因为`<=>`比较的是两个数值，如果有一方不是数值，将返回undef。而在sort中，它们被当作最小值

- 如果是正向排序，则非数值排在最前面  
- 如果是逆序排序，则非数值排在最后面  

下面是几个sort排序示例。

sort排序示例1：排序一串字符串，从字符串的第3个字符开始排序。  

```perl
my @str = qw(Abxx bbcda bdef ab);
my @sorted = sort {substr($a,2) cmp substr($b,2)} @str;
say "@sorted";
```

sort排序示例2：对hash进行排序，排序依据是按照数值大小比较value。

例如，存放姓名和工资的hash，想要按照他们的工资进行排序，如果工资相同，则按照名字的大小顺序进行排序。最后输出排序后的姓名。

可以先使用keys获取key列表，再通过`$hash{key}`对每个value作比较，从而得到key的顺序。

```perl
my %name_salary = (
  malong => 8000,
  wugui => 6000,
  xiaofang => 9000,
  longshuai => 6000,
  woniu => 10000
);

my @sorted_key = sort {
   # 先对工资按数值进行排序
   $name_salary{$a} <=> $name_salary{$b}
   or
   # 如果工资相同，则按照姓名大小排序
   $a cmp $b
  } keys %name_salary;

say "@sorted_key";  # 输出：longshuai wugui malong xiaofang woniu
```

注意，上面的or操作符，当比较的两个工资不等的时候，or前面的`<=>`比较返回1或-1，它们都表示true，于是短路直接返回给sort；当两个工资不等的时候，or前面的`<=>`比较返回0，它表示false，于是比较or后面的`cmp`，同样返回1、-1、0给sort。

### reverse

reverse用于反转列表：在列表上下文中返回元素被反转后的列表，在标量上下文中，返回原始列表各元素组成的字符串的反转字符串。

```perl
my @arr1 = qw(aa bb cc dd);

say "@{[reverse @arr1]}";  # dd cc bb aa
say ~~(reverse @arr1);     # ddccbbaa，返回aabbccdd的反转
```

reverse可以在标量上下文中直接反转一个字符串。

```perl
say ~~reverse "hello";  # olleh
```

reverse也常结合sort一起使用，用来反转sort排序后的结果。但注意，reverse结合sort并不会二次排序，perl会在sort排序时自动将reverse效果应用在sort排序期间，因此不会带来效率的下降。

```perl
my @arr = qw(Abxx bbcda bdef ab);
my @r_sorted = reverse sort {length $a <=> length $b} @arr;
say "@r_sorted";
```