## 解引用

前面说过，在使用上，可以使用变量名称的地方，都可以替换使用引用变量。理解了这一点，就知道如何去解引用：根据引用获取其指向的原始数据。

标量的解引用、数组的解引用和hash的解引用方式如下：

- 引用的是一个标量，解引用时加上sigil前缀`$`  
- 引用的是一个数组，解引用时加上sigil前缀`@`  
- 引用的是一个哈希，解引用时加上sigil前缀`%`  

以标量`$name`及其引用`$name_ref`、数组`@arr`及其引用`$arr_ref`、hash结构`%hash`及其引用`$hash_ref`为例，分别通过变量和引用变量取得其所指向内存数据的分别方式为：

```perl
$name  ->  $$name_ref
@arr   ->  @$arr_ref
%hash  ->  %$hash_ref
```

也可以使用变量的完全限定语法：

```perl
${name}  ->  ${$name_ref}
@{arr}   ->  @{$arr_ref}
%{hash}  ->  %{$hash_ref}
```

对于数组或hash，取它们的某个元素：

```perl
$arr[0]        ->   $$arr_ref[0]
${arr}[0]      ->   ${$arr_ref}[0]
$hash{name}    ->   $$hash_ref{name}
${hash}{name}  ->   ${$hash_ref}{name}
```

例如：

```perl
my @name=qw(junma jinlong);
my $ref_name=\@name;

say "@{ $ref_name }";
say "@$ref_name";
say "$$ref_name[0]";
say "${$ref_name}[0]";
```

### 瘦箭头->解引用

除了上面介绍的解引用方式，Perl还允许使用瘦箭头`->`来解引用获取数组或hash中的元素。例如`$a_ref->[0]`和`$h_ref->{name}`。

例如：

```perl
my @names = qw(junma jinlong);
my $ref_names = \@names;
say $ref_names->[0];   # 等价于${$ref_names}[0]

my %hash=(
    name => "junmajinlong",
    age  => 23,
);
my $ref_hash =\%hash;
say $ref_hash->{name};  # 等价于${$ref_hash}{name}
```

当数组中嵌套数组或hash，hash中嵌套数组或hash时，优先使用瘦箭头解引用的方式来取元素，这样整个取值过程更清晰。

例如，下面是取复杂数据结构中某元素的三种写法，显然使用瘦箭头方式要比原始的解引用方式更简洁。

```perl
# $ref_Config是一个hash引用，取得其中的urllist值
# urllist值是一个数组，取得第二个元素，该元素仍为数组，
# 再取得第四个元素，依然是数组，最后取得第二个元素
say ${$ref_Config}{urllist}[1][3][1];
say ${${${$ref_Config}{urllist}[1]}[3]}[1];
say $ref_Config->{urllist}->[1]->[3]->[1];
```

并且，连续多个瘦箭头时，从第二个瘦箭头开始可以省略瘦箭头。例如上面最后一种写法可简写为：

```perl
say $ref_Config->{urllist}[1][3][1];
```



