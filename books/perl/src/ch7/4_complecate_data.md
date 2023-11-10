## 复杂数据结构

复杂数据结构是指数组中嵌套数组或hash，hash中嵌套数组或hash，甚至它们存在多层嵌套关系。

由于Perl中数组或hash是根据列表提供的数据来构建的，如果直接将数组或hash放进列表上下文，它们将自动转换为列表数据，这样的行为可能会使得最终构建的结果并非预期结果。例如：

```perl
my @arr = qw(a b c);

# 嵌套数组，但@arr的元素直接被插入到了@arr1
# 等价于@arr1 = qw(a a b c b)
my @arr1 = (a, @arr, b);

my @arr2 = qw(a b c);
# %hash的某个value设置为数组，但
# 该数组不会作为值，而是展开为列表再构建Hash
# 等价于: (aa=>'a',b=>'c',bb=>1)
my %hash = ( aa => @arr2, bb => 1);
```

因此，在需要嵌套数组或hash的时候，应当使用它们的引用。

例如：

```perl
my @arr = qw(a b c);
my %h = ( aa => \@arr, bb => 1);
```

现在hash变量h中嵌套保存了一个数组，要获取到这个数组的第一个元素，方式为：

```perl
my $aa_value = $h{aa};
say ${$aa_value}[0];
# 或者可读性非常差的：say ${$h{aa}}[0];
```

如果改用Perl的瘦箭头解引用方式，则取值过程非常清晰：

```perl
say $h{aa}->[0];
say $h{aa}[0];  # 可省略瘦箭头
```

也可以将一个复杂的hash结构的引用嵌套在一个数组或其他hash中，例如：

```perl
my @arr = qw(a b c);
my %h = ( aa => \@arr, bb => 1);
my @outer = ( 'x', 'y', \%h, 'z' );

say $outer[2]->{aa}->[0];
say $outer[2]->{aa}[0];
say $outer[2]{aa}[0];
```

下面是一个更为复杂的取值示例：

```perl
my @more_urllist=qw(http://mirrors.shu.edu.cn/CPAN/
  http://mirror.lzu.edu.cn/CPAN/
);
my @my_urllist=('http://mirrors.aliyun.com/CPAN/',
  'https://mirrors.tuna.tsinghua.edu.cn/CPAN/',
  'https://mirrors.163.com/cpan/',
  \@more_urllist   # 将数组more_urllist引用作为元素
);
my %Config = (
  'auto_commit' => '0',
  'build_dir' => '/home/fairy/.cpan/build',
  'bzip2' => '/bin/bzip2',
  'urllist' => [
    'http://cpan.metacpan.org/',
    \@my_urllist  # 将数组my_urllist作为元素
  ],
  'wget' => '/usr/bin/wget',
);

my $ref_Config=\%Config;
```

现在想要通过`$ref_Config`取得`@more_urllist`中的第二个元素：

```perl
say ${$ref_Config}{'urllist'}[1][3][1];
say ${${${$ref_Config}{'urllist'}[1]}[3]}[1];
say $ref_Config->{urllist}->[1]->[3]->[1];
say $ref_Config->{urllist}[1][3][1];
```

显然，除了第二种写法非常伤眼睛之外，其他写法都比较简洁。

### 打印输出数据结构

默认情况下，Perl使用print、say、printf进行输出，但有些变量数据不适合使用它们进行输出。比如想查看一个hash结构却又不想遍历hash。

可以使用Perl模块`Data::Dump`提供的dump函数进行输出，需要先安装：

```bash
cpan install Data::Dump
```

安装之后，导入使用：

```perl
use Data::Dump qw(dump);

my @arr = qw(a b c d);
my %hash = ( name=>"junma", age=>23, arr => \@arr);

# 传递待输出数据的引用
dump(\@arr);
dump(\%hash);
```

输出：

```
["a" .. "d"]
{ age => 23, arr => ["a" .. "d"], name => "junma" }
```

更适合查看数据结构的是`Data::Printer`模块的p方法。先安装：

```bash
cpan install Data::Printer
```

使用p()输出数据结构：

```perl
use Data::Printer;

my @arr = qw(a b c d);
my %hash = ( name=>"junma", age=>23, arr => \@arr);

p(@arr);
p(%hash);
```

输出结果：

```
[
    [0] "a",
    [1] "b",
    [2] "c",
    [3] "d"
]
{
    age    23,
    arr    [
        [0] "a",
        [1] "b",
        [2] "c",
        [3] "d"
    ],
    name   "junma"
}
```

上面的输出结果中，数组元素带了索引，hash元素的key和value之间使用空白符号分隔。这些输出行为都可以控制。例如：

```perl
use Data::Printer {
  index => 0,     # 不要输出数组索引
  hash_separator => ': ',  # hash元素的key和value分隔符
  quote_keys     => 'auto',  # 自动加引号
};

my @arr = qw(a b c d);
my %hash = ( name=>"junma", age=>23, arr => \@arr);

p(@arr);
p(%hash);
```

输出结果：

```
[
    "a",
    "b",
    "c",
    "d"
]
{
    age : 23,
    arr : [
        "a",
        "b",
        "c",
        "d"
    ],
    name: "junma"
}
```

`Data::Printer`还支持查看(面向对象的)对象结构，支持其他很多种输出控制行为，可参考模块手册：<https://metacpan.org/pod/Data::Printer>。









