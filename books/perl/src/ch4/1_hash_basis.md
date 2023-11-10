## hash基本用法

Perl中hash类型的变量使用`%`前缀表示，由列表创建而成，列表的格式为`(k1,v1,k2,v2,k3,v3)`，即按照一个键一个值的方式构建hash。

例如，构建一个名为person的hash变量：

```perl
my %person = (
  "name"  , "junmajinlong",
  "age"   , 23,
  "gender", "male"
);
```

Perl中，几乎总是可以使用`=>`代替逗号。因此，下面是等价的方式：

```perl
my %person = (
  "name"  => "junmajinlong",
  "age"   => 23,
  "gender"=> "male"
);
```

当key的名称符合标识符命名规范时(只包含下划线、字母和数值，且非数值开头)，可以省略key的引号。因此，构建hash时通常写成下面这种可读性更高的方式：

```perl
my %person = (
  name   => "junmajinlong",
  age    => 23,
  gender => "male"
);
```

可以根据`$hash{key}`的方式来检索hash结构中key对应的value，如果key符合标识符命名规范，则可以省略包围key的引号。

```perl
my %person = (
  name   => "junmajinlong",
  age    => 23,
  gender => "male"
);

say "$person{name}";
say $person{"age"};
```

如果访问hash中不存在的key，则返回undef，而不会报错：

```perl
say $person{class};  # undef
```

hash变量不能内插到双引号。

```perl
say "%person";   # 直接输出：%person
```

**在列表上下文，hash变量会自动隐式转换为`(k1,v1,k2,v2,k3,v3)`格式的列表**。

```perl
my %person = (
  name   => "junmajinlong",
  age    => 23,
  gender => "male"
);

# hash展开成列表，默认所有的key和value紧密相连输出
# 修改内置变量`$,`可设置say/print输出时的列表分隔符
say %person;  # namejunmajinlonggendermaleage23
$, = "-";
say %person;  # name-junmajinlong-gender-male-age-23
```

**在标量上下文，如果hash为空，则转换为数值0，如果hash结构非空，则hash变量会转换为`m/n`格式的标量，m表示当前的键值对数量，n表示hash结构当前的容量**。因此，可以直接将hash变量作为布尔值判断：非空hash为true、空hash为false。

```perl
my %h;     # 空hash
say ~~%h;  # 输出：0
if(%h){say "empty hash"}  # 不输出

$h{k1} = "v1";
say ~~%h;   # 输出：1/8
if(%h){say "not empty hash"}  # 输出
```

将hash变量赋值给另一个hash变量时，由于赋值hash时在列表上下文，因此会先将hash展开为列表，再赋值。

```perl
my %person = (
  name   => "junmajinlong",
  age    => 23,
  gender => "male"
);

my %p = %person;  # %person展开为列表，然后构建%p
```

### 多键组合的hash

在向hash中存储数据时，如果想要用多份数据组合起来作为key，那么可以用字符串相连的方式将它们的值连接起来：

```perl
my %h;
my ($x, $y) = qw(x, y);
$h{$x.$y} = "junmajinlong.com";  # 等价于$h{"$x$y"}
say $h{"$x$y"};
```

但这样的方式不安全。例如`$x=aa,$y=aa`组合作为key时，和使用`$a=a,$b=aaa`组合作为key是一样的。

Perl提供了一种更简便、更安全的逗号分隔`,`方式，当使用逗号分隔多份数据组合为key时，Perl会自动将每份数据使用下标连接符(默认值为`\034`)连接起来，最终得到的字符串作为key。`\034`通常可以认为是安全的连接符，它是一个ASCII中的控制字符，几乎不会出现在文本数据中。

```perl
my %h;
my ($x, $y) = qw(x y);

$h{$x, $y, "name"} = "junmajinlong.com";
say $h{$x, $y, "name"};
say $h{"$x\034$y\034name"};  # 等价形式
```

Perl使用的下标连接符由内置变量`$;`控制，该内置变量的默认值为`\034`。因此，下面这种写法也和上面的写法等价。

```perl
say $h{join($;, $x, $y, "name")};
```

