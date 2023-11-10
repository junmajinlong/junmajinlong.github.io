## 匿名数组和匿名hash

在此之前，构建数组和hash结构时，都使用列表语法去提供数据。但Perl也是允许使用中括号`[]`构建数组，使用大括号`{}`构建hash的，只不过它们构建的是匿名数组、匿名hash，它们都是引用。

- 使用中括号`[]`构建匿名数组  
- 使用大括号`{}`构建匿名hash  
- 不包含任何元素的`[]`和`{}`分别是匿名空数组、匿名空hash  

### 构造匿名对象

例如，在数组、hash中构建匿名数组：
```perl
# 匿名数组、匿名hash都是引用，因此赋值给标量变量
my $anonymou_array = []; # 空数组
my $anonymou_hash = {};  # 空hash

# 将匿名数组嵌套在其他数组或hash中
my @name=('fairy', ['longshuai','wugui','xiaofang']);
my %hash=('longshuai' => ['male',18,'jiangxi'],
          'wugui'     => ['male',20,'zhejiang'],
          'xiaofang'  => ['female',19,'fujian'],
         );

say "$name[1][2]";
say "$hash{wugui}[1]";
```

如果不想在匿名数组中输入引号，可以使用qw()。
```perl
# 以下等价
my @name=('fairy',['longshuai','wugui','xiaofang']);
my @name=('fairy',[qw(longshuai wugui xiaofang)]);
```

在数组、hash中构建匿名hash：
```perl
my @name=(         # 匿名hash作为数组的元素
  {    # 第一个匿名hash
   'name'=>'longshuai', 'age'=>18,
  },
  {    # 第二个匿名hash
   'name'=>'wugui', 'age'=>20,
  },
  {    # 第三个匿名hash
   'name'=>'xiaofang',  'age'=>19,
  },
);

my %hash=(         # 匿名hash作为hash的value
  'longshuai'=>{   # 第一个匿名hash
                'gender'=>'male', 'age'=>18,
               },
  'wugui'=>{       # 第二个匿名hash
            'gender'=>'male', 'age'=>20,
           },
  'xiaofang'=>{    # 第三个匿名hash
               'gender'=>'female', 'age'=>19,
              },
);
```


### 解除匿名对象的引用

匿名数组或匿名hash是引用，因此可以解除引用，还原得到原始数据对象。

```perl
my $arr_ref = [qw(a b c)];
say "@$arr_ref";  # 解除引用得到匿名数组
```

除了可以通过解除引用变量的方式来得到匿名数据结构，还可以直接解除匿名数据结构：

- 解除匿名数组的引用`@{[]}`  
- 解除匿名hash的引用`%{ {} }`  

之所以可以直接解除匿名数组和匿名hash，是因为构建匿名数组的`[]`和构建匿名hash的`{}`本身就返回引用。

例如，解除匿名数组：

```perl
# 解除匿名数组的引用，得到数组，再将其内插到双引号
say "@{ ['longshuai','xiaofang'] }";
say "@{ [qw(longshuai xiaofang)] }"; 
# 获取匿名对象中的第二个元素
say "@{ [qw(longshuai xiaofang)] }[1]";
```

解除匿名hash：

```perl
say %{   # 解除匿名hash
  { # 构造匿名hash
    longshuai=> ['male',18,'jiangxi'],
    wugui    => ['male',20,'zhejiang'],
    xiaofang => ['female',19,'fujian'],
  }
}{longshuai}->[1];
```

### 解除匿名数组在双引号中内插的技巧

双引号中可以内插标量变量、数组变量，更严谨地说，是双引号中允许内插sigil符号`$ @`符号的取值操作。比如取标量值(包括取数组元素、hash元素)、取列表数据(包括取数组数据、取切片数据)。

```perl
# 内插标量取值
say "$name";
say "$hash{name}";
say "$arr[0]";
say "$hash{name}->{nn}->{nnn}";

# 内插列表值
say "@arr";
say "@arr[0,1,2]";
say "@hash[name,age]";
```

但Perl并不允许在双引号中直接内插hash，也不允许直接内插表达式或语句。

而从前面示例可知，解除匿名数组的语法`@{[]}`是可以直接内插在双引号中的，它会通过中括号将其中的数据构建成匿名数组，然后使用`@{}`解除匿名数组的引用，最终得到一个数组结构。这种语法为双引号中内插表达式提供了解决方案。

例如：

```perl
my $age = 23;
say "age+10: @{[$age+10]}";

my @arr = qw(Perl Go JavaScript Shell Ruby Python);
say "[len >= 5]: @{[grep {length $_ >=5} @arr]}";
```

最后注意，`@{[]}`的上下文环境是列表上下文。

### 大括号：区分匿名hash和代码块

Perl中大括号被用在多个地方：一次性语句块、if/for/while等的语句块、grep/map等的语句块、构建匿名hash。

大多数时候，Perl会根据使用`{}`的上下文环境自动判断大括号的作用，但有时候会判断错误或无法判断。例如：

```perl
# Perl无法判断
my %hash = map {  "\L$_" => 1  } @array;

# Perl推断出大括号是语句块
my %hash = map { +"\L$_" => 1  } @array;
my %hash = map {; "\L$_" => 1  } @array;
my %hash = map { ("\L$_" => 1) } @array;
my %hash = map {  lc($_) => 1  } @array;

# Perl推断出大括号是构建匿名hash的大括号
my @hashes = map +{ lc($_) => 1 }, @array;
```

因此，有时候需要显式地告诉Perl，这个大括号是语句块的大括号还是构建匿名hash的代码块：  
- 大括号前面加上`+`符号，即`+{...}`，表示这个大括号是用来构造匿名hash的  
- 大括号内第一个语句前，使用一个`;`，即`{;...}`，表示这个大括号是语句块  

`+`不仅仅可以加载匿名hash的大括号前，还可以加在匿名数组的中括号前，以及hash引用变量、数组引用变量前，如`+$ref_hash`、`+[]`、`+$ref_arr`。

```perl
@{ +[qw(longshuai wugui)]}   # 匿名数组中括号前
@{ +$ref_arr }           # 数组引用变量前
%{ +$ref_hash }          # hash引用变量前
```

### autovivification特性

这个单词是perl自造的词，应用到了多种语言中：[Wiki Autovivification](https://en.wikipedia.org/wiki/Autovivification)。

它的功能大致为：**当解除引用时，如果解除目标不存在，Perl会自动创建一个空目标，且会自动递归补齐上层缺失目标**。注意，autovivification的作用体现在解除引用时。

这有点像Unix下的`mkdir -p`一样，当创建某个目录的时候，如果其父目录不存在，则自动补齐创建缺失的父目录。


例如：
```perl
push @{ $config{path} },'/usr/bin/perl';

say keys %config;        # 输出：path
say $config{path};       # 输出：ARRAY(0x...)
say $config{path}[0];    # 输出：/usr/bin/perl
```

执行到push的时候，perl首先会发现`@{}`在解除一个匿名数组引用，这个引用来自于`$config{path}`，因此`$config`是一个hash引用，但是perl发现这个hash目前还不存在，hash里的key(即path)也不存在。于是根据autovivification特性，perl首先会构建一个空的hash对象`%config={}`，然后创建hash里的一个key：path，其值为空列表，即`$config{path}=[]`，最后将`"/usr/bin/perl"`字符串push到对应的列表中，即`$config{path}=['/usr/bin/perl']`。

在上面的示例中，perl在解除引用时，自行创建了几个部分：  

- 自建一个hash对象  
- 自建hash对象中的一个元素  
- 自建hash对象中某个元素的value部分  

必须注意，perl的autovivification功能只在解除引用的时候才自建缺失的结构，从解除引用的操作动机上看，每当要解除引用，说明可能要操作引用对象中的数据了，那么缺少的部分应该要补齐。

如果不是在解除引用，那么Perl将根据**语法特性**决定是否自建对象。例如下面将自建数组`@name`和hash对象`%person`以及它的一个元素`$person{name}`。但这不是autovivification的特性，而是perl的语法特性，且是未处在strict模式下的特性。
```perl
push @name,"longshuai";
$person{name}="longshuai"

say "$name[0]";
say keys %person;
```

紧跟着上面的示例：
```perl
@{ $config{path} }[2]='/usr/bin/perl';

say $config{path};       # 输出：ARRAY(0x5571664403c0)
say $config{path}[0];    # 输出：空
say $config{path}[1];    # 输出：空
say $config{path}[2];    # 输出：/usr/bin/perl
say scalar @{$config{path}};   # 输出元素个数：3
```