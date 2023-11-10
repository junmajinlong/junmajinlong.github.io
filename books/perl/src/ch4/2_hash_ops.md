## 操作hash

### hash切片

Perl也支持对hash变量的切片。

例如：

```perl
my %phone_num = (
  longshuai =>"18012345678",
  xiaofang =>"17012345678",
  tun_er =>"16012345678",
  fairy =>"15012345678"
);
my ($a,$b,$c) = @phone_num{qw(xiaofang fairy xiaofang)};
```

需要注意的几点是：  

- hash切片使用`@`前缀，而不是`%`前缀，因为它代表访问多个内存数据空间  
- hash切片的索引部分是一个列表上下文，是表示键的列表  
  - 因此`@h{qw(a b)}`或`@h{"a","b"}`是有效的，但省略双引号`@h{a,b}`是错的  
- hash不可内插至双引号，但hash的切片可以内插到双引号，因为hash切片的结果是一个列表  
- hash切片可以作为左值，从而修改hash中对应的键值对数据  

```perl
my %phone_num = (
  longshuai =>"18012345678",
  xiaofang =>"17012345678",
  tun_er =>"16012345678",
  fairy =>"15012345678"
);

# 双引号中内插hash切片
say "@phone_num{qw(fairy longshuai)}";

# hash切片作为左值
@phone_num{qw(fairy longshuai)} = qw(155555555 188888888);
say "@phone_num{qw(fairy longshuai)}"; # 输出：155555555 188888888
```

### hash类内置函数

perl提供了几个基本的hash内置函数：delete、each、exists、keys、values。

#### keys和values

keys和values分别用来获取hash的key列表和value列表。注意，hash各元素的出现顺序是不可预测的。

```perl
my %h = qw(k1 v1 k2 v2 k3 v3);
my @keys = keys %h;
my @values = values %h;
say "keys: @keys";     # 输出：keys: k3 k2 k1
say "values: @values"; # 输出：values: v3 v2 v1
```

#### each和遍历hash

each常和while结合用来遍历数组和hash。每次each迭代时，都获取索引和对应值，并作上位置标记，下次从标记处开始继续迭代。需小心使用each，原因可参考[小心使用each](./ch3/5_iterator.md#perl_each)。

例如：

```perl
while(my ($k, $v) = each %hash){
  say "key: $k, v: $v";
}
```

for、foreach也可以遍历hash，但只能通过keys函数来遍历Key，通过values函数来遍历Value。 

```perl
my %myhash = qw(k1 v1 k2 v2 k3 v3);

# foreach迭代遍历key
foreach my $k (sort keys %myhash){
 say $k, $myhash{$k};
}

# 迭代遍历value
foreach my $v (values %myhash){
  say $v;
}
```

此外，还需注意，each、keys、values这三个函数共用一个迭代指针，每次调用keys、values时都会重置each的迭代指针。因此，`while + each`遍历hash时，循环体内要小心使用keys、values函数。

例如，下面的代码将无限循环。

```perl
while(my ($k, $v) = each %p){
  say "k: $k, v: $v";
  keys %p;
  # keys在空上下文中只重置迭代指针而不返回key列表，因此效率不受影响
}
```

#### exists和判断键是否存在

exists用来判断hash结构中是否存在某个key，如果存在则返回表示布尔真的结果(数值1)。由于Perl中访问hash结构中不存在的键值对时不会报错，而是返回undef，因此有时候也会直接使用hash索引的方式来测试。但注意，如果key存在于hash结构中，但其对应的值表现为布尔假(即value为undef、0或空字符串等值)时，两种方式测试结果将不同。

```perl
my %h;
if(exists $h{k1}){...}
if($h{k1}){...}

$h{k1} = undef;
if(exists $h{k1}){say "1"}  # 输出
if($h{k1}){say "2"}  # 不输出
```

#### 删除键值对和清空hash

delete用来删除hash结构中的键值对，可以使用hash切片方式一次性删除多个键值对。

```perl
my %hash = (
  a => "aa",  b => "bb",
  c => "cc",  d => "dd"
);

delete $hash{a};         # 删除单个键值对
delete @hash{qw(b c)};   # 根据hash切片删除
$, = "-";
say %hash;   # d-dd
```

作为技巧，`delete @HASH{keys %HASH}`会清空hash，但效率很低。清空hash效率更高的是下面这两种方式：

```perl
%HASH = ();     # 直接清空hash
undef %HASH;    # 注销hash变量
```

delete还有一个特殊的用法：`delete local`，它只会在当前作用域内删除hash键值对，退出该作用域后，被删除的键值对仍然存在。

```perl
my %hash = (
  a => "aa",  b => "bb",
  c => "cc",  d => "dd"
);

$, = '-';
{
  delete local $hash{a};
  say %hash;   # 输出：b-bb-c-cc-d-dd
}
say %hash;     # 输出：d-dd-c-cc-a-aa-b-bb
```


