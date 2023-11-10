## 字符串替换：s///

`m//`模式用来匹配文本，即用于搜索。Perl还支持和sed类似的`s///`用法，它用来查找并替换文本。

```
$str =~ s/PATTERN/Replacement/FLAGS;
```
它表示用指定的正则表达式PATTERN去搜索`$str`中的内容，并将搜索出来的内容替换为Replacement。

FLAGS指定s替换时的行为，`m///FLAG`支持的FLAG也都可以用于`s///`。例如，`s/reg//i`表示PATTERN正则匹配的时候不区分大小写。

例如：

```perl
my $str = "junmajinlong";
$str =~ s/(jin.*)$/ \1/;
say $str;   # junma jinlong
```

`s///`的斜线可以替换为其他对称的符号(括号类)或相同的符号。例如，左右相同的分隔符`s!!! s### s%%%`，对称的括号分隔符`s()() s{}{} s<><>`等，或者混用这些分隔符`s{}## s{}()`等。

如果不指定替换目标，则默认操作`$_`。因此，下面两种写法是等价的：

```perl
s///;
$_ =~ s///;
```


默认情况下，`s///`返回替换的次数。因此，如果发生了替换，那么替换次数等于1或次数大于1(使用了g修饰符)，它们都可以代表真值，如果未发生替换，则返回0，可代表布尔假。

```perl
# 如果替换成功，执行say
say "substituted" if s///;

# 不断将空格替换为逗号，每轮循环替换一次，直到所有空给都替换完成
while(s/ /,/){
  ...
}
```

如果FLAGS处使用了r修饰符，则拷贝`$str`，并对拷贝后的数据进行替换，原变量`$str`的内容不变，此时返回值是替换后的结果而不是替换次数。

在PATTERN中使用分组捕获时，由于捕获成功后会立即将分组捕获的结果保存到特殊变量`\1`和`$1`中，所以在Replacement部分可以使用这些变量。

```perl
say $str = "gao xiao or ma long";
$str =~ s/(gao)(.*)(ma)(.*)/\3$2\1$4/;
say $str;     # 输出ma xiao or gao long
```

在PATTERN和Replacement部分，可以使用变量，它们会替换为对应的变量值：

```perl
my $foo = "foo";
my $bar = "BAR";

my $str="hello foo";
$str =~ s/$foo/$bar/; # hello BAR
```

还有一种操作符`$str !~ s///`，用法和`=~`是相同，只不过它转换了布尔逻辑：替换成功时返回0，未发生替换时返回1。

需要注意，有时候将所有替换操作放在单个`s///`中，效率可能不如使用多个`s///`分次替换，这和正则表达式的效率有关。例如，移除前缀空白和后缀空白：

```perl
# 一次性替换，效率低
s/^\s*(.*?)\s*$/$1/; 

# 分两次替换，效率高
s/^\s+//;
s/\s+$//;
```

## 修饰符

`m//FLAG`可用的修饰符在`s///`中基本都可用，最常用的修饰符还是`gimsx`。

```perl
my $str = "Gao xiao or Ma long";
# 全局替换，且搜索时不区分大小写
$str =~ s/(gao).* or (ma).*/$2 xiao or $1 long/ig;
```

再例如，压缩空白：
```perl
s/(\s)+/\1/g;   # 将多个空白缩减成一个
s/^\s+//;       # 去除行首空白
s/\s+$//;       # 去除行尾空白
s/^\s+|\s+$//;  # 去除行首空白和行尾空白
```

此外，`s///`还有自己的修饰符r和e。注：还有一个ee修饰符，和e类似，但多一层eval评估的效果。

### r修饰符

默认情况下，`s///`的返回值是替换成功的次数，使用r修饰符，可以让这个替换操作返回替换后的字符串。几个注意点：  
1. r修饰符实际上是在替换前先拷贝一份待替换数据，然后在副本上进行替换，所以原始数据不会发生改变  
2. r修饰符实际上是返回拷贝后的数据，如果替换成功，则返回替换后的字符串，如果替换失败，则直接返回这个副本  
3. r修饰符的替换返回结果一定是纯文本字符串，即使它操作的是一个对象  

```perl
my $str = "ma xiao or ma long";

say $str =~ s/Ma/gao/igr;        # 输出替换后的内容
say $str;                        # 原始内容不变
my $copy = $str =~ s/Ma/gao/igr; # 替换后的内容赋值给新的变量
say $copy;                       # 输出替换后的内容
```

如果不使用r修饰符，想要将替换的内容输出，只能先将其保存到一个新的变量中，然后输出这个变量：
```perl
my $str = "ma xiao or ma long";
(my $copy = $str) =~ s/Ma/gao/ig;
say $str;        # 原始数据不变
say $copy;       # 替换后的数据
```

使用r修饰符的时候，可以将多个`s///r`串联起来：

```perl
# bar数据不变，foo保存两次替换之后的结果
my $foo = $bar =~ s/this/that/r
               =~ s/that/other/r;
```

r修饰符在map函数中非常好用，它可以替换一个列表中的某些元素。例如，下面的map将`@list`中首字母大写的单词替换为小写。

```perl
my @list = qw(Ma long Gao xiao);
my @new_list = map {s/([A-Z])([a-z]+)/\L\1\E\2/rg} @list;
say "@new_list";
```


### e修饰符


e可以让Replacement部分当作一个表达式执行，然后将表达式返回结果替换到PATTERN匹配成功的地方。


```perl
my $str="ma long or ma xiao";

# 不使用e修饰符
$str =~ s/ma/$& x 2/g;
say $str;  # ma x 2 long or ma x 2 xiao

# 使用e修饰符
$str =~ s/ma/$& x 2/eg;
say $str;  # mama long or mama xiao
```

e修饰符使得能够直接在Replacement处使用Perl表达式，这个功能非常好用。例如：

```perl
s/PATTERN/++$num/e;     # 执行自增操作后替换
s/PATTERN/@{[EXPR]}/e;  # 直接执行一个内插表达式
s/PATTERN/sprintf()/e;  # 格式化字符串后再替换
s/PATTERN/print$&;$&/e; # 输出$&，丢弃返回值，然后替换为$&
```

