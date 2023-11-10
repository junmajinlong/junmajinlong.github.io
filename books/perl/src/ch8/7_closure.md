## 回调函数和闭包

在Perl中，子程序的引用或匿名子程序常用来做回调函数(callback)、闭包(closure)。

### 回调函数

回调函数的意义是：每触发某种事件就调用一次用户手动指定的函数。这个指定的函数通常通过参数传递。

例如，给子程序传递一个数值，子程序将每0.5秒自增一次该数值，每当数值能被5整除，就执行一次通过参数传递的子程序：

```perl
use Time::HiRes qw(sleep);

# subname(Sub, N)
sub subname{
  my $sub_ref = shift;
  my $cnt = shift;
  my $tmp;
  while(1){
    $tmp = $cnt++;
    $sub_ref->($tmp) if $tmp % 5 == 0;
    sleep 0.5;
  }
}

subname(
  sub {say "lalala: @_";}
  ,3);
```

上面通过参数传递的匿名子程序`sub {say "lalala: @_"}`就是回调函数，每当触发某个条件时它就被执行。

再例如，`File::Find`模块的find函数可用于搜索给定目录下的文件，然后对每个搜索到的文件执行一些操作(通过定义子程序)，这些操作对应的函数要传递给find函数，它们就是回调函数。就像unix下的find命令一样，找到文件，然后print、ls、exec CMD操作一样，这几个操作就是find命令的回调函数(或者称为回调命令)。

```perl
use File::Find;

sub cmd {
  say "$File::Find::name";
};

find(\&cmd, qw(/perlapp /tmp/pyapp));
```

其中`$File::Find::name`代表的是find搜索到的从起始路径(`/perlapp /tmp/pyapp`)开始的全路径名，此外，find每搜索到一个文件，就会赋值给默认变量`$_`。它代表的是文件的basename，和`$File::Find::name`全路径不一样。例如：
```
  起始路径     $File::Find::name       $_
-------------------------------------------
  /perlapp     /perlapp/1.pl         1.pl
  .             ./a.log              a.log
  perlapp       perlapp/2.pl         2.pl
```

回到回调函数的问题上。上面的示例中，定义好了一个名为cmd的子程序，这个子程序不需要手动去调用，而是将其引用作为参数传递给find函数，由find函数自动去调用它。

由于回调函数通常只作为参数传递给子程序，而不手动调用，因此没必要花脑细胞去设计它的名称，完全可以将其设计为匿名子程序，放进find函数中。
```perl
use File::Find;

find(
  sub {
    say "$File::Find::name";
  },
  qw(/perlapp /tmp/pyapp)
);
```

### Perl闭包

从Perl语言的角度来简单描述下闭包：子程序1中返回另一个子程序2，这个子程序2访问子程序1中的变量x，当子程序1执行结束，外界无法再访问x，但因为子程序2还引用着变量x所保存的数据，使得子程序2在子程序1结束后可以继续访问变量x所保存的数据。

所以，**子程序1中的变量x必须是my声明的词法变量(可简单理解为私有变量)**，否则子程序1执行完后，变量x仍可以被外界访问、修改，如果这样，闭包和普通函数就没有区别了。

一个简单的闭包示例：
```perl
sub sub1 {
  my $var1 = 33;
  my $sub2 = sub {
    $var1++;
  }
  return $sub2; # 返回一个闭包
}

# 将闭包函数存储到子程序引用变量
my $my_closure = sub1();
```

子程序sub1内部的子程序`$sub2`可以访问属于`$sub1`但不属于子程序`$sub2`的变量`$var1`，这样一来，只要把调用sub1返回的闭包赋值给`$my_closure`，就可以让这个闭包函数一直引用`$var1`变量所保存的数据。并且，离开了sub1，除了`$my_closure`，没有任何其他方式可以访问`$var1`所保存的数据。当sub1执行完毕后，外界也将没有任何方式去去访问这个数据。

简单来说，sub1退出后，闭包sub2是唯一可以访问sub1中变量的方式。

下面是一个具体的Perl闭包示例：
```perl
sub how_many {       # 定义函数
  my $count=2;     # 词法变量$count
  return sub {say ++$count};  # 返回一个匿名函数，这是一个匿名闭包
}

my $ref=how_many();    # 将闭包赋值给变量$ref

how_many()->();  # (1)调用匿名闭包：输出3
how_many()->();  # (2)调用匿名闭包：输出3
$ref->();        # (3)调用命名闭包：输出3
$ref->();        # (4)再次调用命名闭包：输出4
```
上面将闭包赋值给`$ref`，通过`$ref`去调用这个闭包，即使how_many中的`$count`在how_many()执行完就消失了，但`$ref`指向的闭包函数仍然在引用这个变量，所以多次调用`$ref`会不断修改`$count`的值，所以上面(3)和(4)先输出3，然后输出改变后的4。而上面(1)和(2)的输出都是3，因为两个how_many()函数返回的是独立的匿名闭包。

Perl语言有自己的特殊性，它的某些语句块具有独立的作用域环境。例如，只执行一次的语句块(即用一对大括号`{}`包围)。这使得Perl中的闭包并非一定要嵌套在一个子程序中，也可以将闭包函数放在一对大括号中：
```perl
my $closure;
{
  my $count=1;   # 随语句块消失的词法变量
  $closure = sub {print ++$count,"\n"};  # 闭包函数
}

$closure->();  # 调用一次闭包函数，输出2
$closure->();  # 再调用一次闭包函数，输出3
```

在上面的代码中，`$count`所指向内存数据的引用计数在赋值时为1，在sub中使用并赋值给`$closure`时引用计数为2，当离开大括号代码块的时候，`$count`被销毁，该内存数据的引用计数减1，此时闭包`$closure`仍在引用该数据。

值得注意的是，Perl中的子程序也保存在内存中，它也是一份实实在在的数据，这一点和其他语言可能不太一样。上面的示例中，在大括号语句块中，`$closure`引用这个子程序数据，当离开大括号时，`$closure`仍是有效变量，它依然引用这个子程序数据，使得该子程序不被销毁。

如果确实想要定义只在某范围内有效的子程序，可开启Perl的特性`use feature 'lexical_subs'`。此时可定义`my sub`或`state sub`，这种方式定义的子程序只在某作用域内有效。例如：

```perl
no warnings "experimental::lexical_subs";
use feature 'lexical_subs';
sub whatever {
  my $x = shift;
  my sub inner {
    ... do something with $x ...
  }
  inner();
}  # 退出后，inner()失效
```

关于`my sub`和`state sub`相关的细节，参考`perldoc perlsub`中的*Lexical Subroutines*段落。

### 闭包的注意事项

对于下面的示例：

```perl
{
  my $count=10;
  sub one_count{ ++$count; }
  sub get_count{ $count; }
}

one_count();
one_count();
say get_count();  # 输出12
```

但如果将调用子程序的语句放在代码块前面呢？
```perl
one_count();  # 1
one_count();  # 2
say get_count();  # 输出：2

{
  my $count=10;
  sub one_count{ ++$count; }
  sub get_count{ $count; }
}
```

上面输出2，这暗示了`$count=10`的赋值行为尚未进行。这是因为my声明的词法变量和它的初始化过程是在编译期间完成的，而赋值操作是在执行到赋值语句时进行的。所以，当编译完成后进入运行期间，在执行到`one_count()`这条语句时，将调用已编译好的子程序one_count，但这时`$count`的赋值还没有执行。

可以将上面的语句块加入到BEGIN块中：
```perl
one_count();  # 11
one_count();  # 12
say get_count();  # 输出：12

BEGIN{
  my $count=10;
  sub one_count{ ++$count; }
  sub get_count{ $count; }
}
```

### state修饰符替代简单的闭包

闭包的作用是为了让my声明的词法变量不能被外部访问，但却让子程序持续访问它。

Perl v5.10提供了一个state修饰符，它和my完全一样，都用于声明私有的词法变量，唯一的区别在于state修饰符使得变量持久化，且state修饰的变量只会初始化赋值一次。

注意：  
- state修饰符不仅仅只能用于子程序中，在其他语句块中也可以使用，例如find、grep、map、循环中的语句块  
- 只要没有东西在引用state变量所指向的数据，它指向的数据就会被回收  
- 目前state只能修饰标量，可以修饰数组、hash的引用变量，因为引用就是个标量  

例如，将state修饰的变量从外层子程序移到内层子程序中。下面两个子程序等价：
```perl
use 5.010;  # for state
sub how_many1 {
  my $count=2;
  return sub { say ++$count };
}

sub how_many2 {
  return sub {state $count=2;ay ++$count};
}

my $ref=how_many2();  # 将闭包赋值给变量$ref
$ref->();             # (1)调用命名闭包：输出3
$ref->();             # (2)再次调用命名闭包：输出4
```

需注意，虽然`state $count=2`，但同一个闭包多次执行时不会重新赋值为2，而是在初始化时赋值一次。

而且，将子程序调用语句放在子程序定义语句前面是可以如期运行的(前面分析过，闭包不会如期运行)：
```perl
my $ref=how_many2();   # 将闭包赋值给变量$ref
$ref->();           # (1)调用命名闭包：输出3
$ref->();           # (2)再次调用命名闭包：输出4

sub how_many2 {
  return sub {state $count=2;say ++$count};
}
```

这是因为`state $count=2`是闭包函数的一部分，无论在哪里调用到它，都会执行它，只不过会它只被初始化一次。

再例如，state用于while循环的语句块内部，使得每次迭代过程中都持续访问这个变量，而不会每次迭代都初始化：
``` perl
use v5.10;   # for state

while($i<10){
  state $count;
  $count += $i;
  say $count;   # 输出：0 1 3 6 10 15 21 28 36 45
  $i++;
}
say $count;       # 输出空
```