## 子程序的参数

Perl子程序不需要定义形参，在调用子程序时，传递参数的地方是一个列表上下文，传递的所有实参会依次放入名为`@_`的数组中。因此，通过`$_[0]`可获取第一个参数的值，通过`$_[1]`可获取第二个参数的值，依次类推。

```perl
sub area{
  my $length = $_[0];
  my $width = $_[1];
  return $length * $width;
}

# 传递两个参数，子程序内部：`@_ = (2, 5)`
say area(2, 5);
```

> Perl的默认变量总结：
>
> 1.默认标量变量`$_`  
>
> 2.在子程序内部，默认列表变量是`@_`  
>
> 3.在子程序外部，默认列表变量是`@ARGV`  

在子程序内部，列表操作的默认对象是`@_`，因此，在子程序内部，下面两个操作是等价的：

```perl
shift @_;
shift;
```

它们都表示获取第一个参数，且从参数列表`@_`中移除第一个参数。


在定义子程序时，经常会使用`shift`来获取所传递的参数并修改参数列表：

```perl
sub subname{
  my $arg1 = shift;
  my $arg2 = shift;
}
```

### 处理多个参数

调用子程序时，可直接传递多个参数：

```perl
subname('a', 'b', 'c', 'd');
```

上面传递的四个参数，均存储在`@_`中，可分别通过`$_[0]`、`$_[1]`、`$_[2]`、`$_[3]`获取。

有时候可能会期望将前几个参数赋值给指定变量，剩余参数放进一个数组。这种需求可以这样实现：

```perl
my ($a, @arr) = @_;
```

当然，也可以将传递的参数组合成hash：

```perl
sub subname {
  my %hash = @_;
  say keys %hash;
  say values %hash;
}

subname(
  "name", "junmajinlong",
  "age", 23,
);
# 或者
subname(
  name=>"junmajinlong",
  age=>23,
);
```

上面第二种调用subname子程序时传递参数的方式也是传递命名参数的方式。但是，这种方式处理命名参数比较容易出错，这要求传递的参数必须能够转换成hash数据，如果传递的参数不合理，将会给出警告或报错。

例如，只传递单个参数，将无法合理地构建成hash，于是会给出警告信息：

```perl
subname("junmajinlong");
```

解决这种问题的方案是传递一个hash引用。参考下文。

注意，重新构建的数组变量、hash变量必须放在最后面。例如，下面处理参数的方式是不合理的：

```perl
my (@arr, $a, $b) = @_;
```

 放在最前面的`@arr`会吞掉`@_`的所有元素，使得`$a`和`$b`都是空变量。

### 传递数组和hash

实际上，调用子程序的参数部分是一个列表上下文，传递的所有参数都会转换为列表数据然后放入`@_`中。

因此，直接将数组变量或hash变量作为参数传递给子程序，将得不到期待的结果，数组变量或hash数据都会被压平(flatten)。

```perl
my @arr = qw(a b c);
my %h = (name=>"junma", age=>23,);
subname(@arr);  # 等价于subname('a', 'b', 'c');
subname(%h);  # 等价于subname('name', 'junma', 'age', 23);
```

某些情况下，可以将它们压平后的元素重新构建成数组或hash：

```perl
sub subname{
  my @arr = @_; # 将所有参数重新构建成数组
  my %h = @_;  # 将所有参数重新构建成Hash
  ...
}
```

但这种重新构建数组或hash的方式不能适用于所有场景，有些情况下也不方便，比如传递的参数数量是奇数时不能合理地构造hash，比如要传递非常多参数时，传递所有参数数据不如传递一个包含它们的引用更高效。

如果确实需要传递数组或hash，建议的方式是传递它们的引用。

```perl
sub subname{
  my $arr_ref = shift;
  say $$arr_ref[0];
}

my @arr = qw(a b c);
subname(\@arr);
```

当然，也可以传递匿名数组、匿名hash，它们本质上仍然是引用。

```perl
sub subname{
  my $arr_ref = shift;
  my $hash_ref = shift;
  my $scalar = shift;
  
  say $$arr_ref[0];
  say $$hash_ref{name};
  say $scalar;
}

subname([qw(a b c)], {name=>'junma', age=>23}, 3333);
```

### 参数是别名引用

调用子程序时传递的参数，放入`@_`中的都是原始数据的别名引用：修改`@_`各元素的值，也将影响原始数据。

例如：

```perl
sub subname{
  $_[0]++;
  say $_[0];
}

my $a=33;
subname($a);
say $a;
```

这将输出：

```
34
34
```

也就是说，`@_`中的数据和原始数据是完全相同的，它们是别名关系。

这种别名关系也存在于容器的元素中。

```perl
sub subname{
  $_[0]++;
  say $_[0];
}

my @arr = (11,22,33);
subname(@arr);
say $arr[0];
```

实际上，`subname(@arr)`会将`@arr`的各元素压平后传入子程序内的`@_`，但是压平后的是各元素的别名，而不是各元素的值。因此上面的示例输出：

```
12
12
```

如果想要让子程序内部的修改不影响原始数据，则应该将参数数据保存到变量中。

```perl
sub subname{
  my $a = shift;
  $a++;
  say $a;
}

my $x=33;
subname $x;  # 34
say $x;      # 33
```

数组也类似：

```perl
sub subname{
  my @arr = @_;
  $arr[0]++;
  say $arr[0];
}

my @arr=(11,22,33);
subname @arr;  # 12
say $arr[0];   # 11
```





