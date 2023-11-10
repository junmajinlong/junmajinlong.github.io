## 子程序引用和匿名子程序

子程序也有引用，创建子程序引用的方式为`\&subname`。

例如：

```perl
my $sub_ref = \&subname;
```

通过子程序引用调用子程序的方式包括：  
- (1).`&$sub_ref([ARGS])`或完整格式的`&{$sub_ref}([ARGS])`，某些时候必须使用完整格式
- (2).`$sub_ref->([ARGS])`  
- (3).`&$sub_ref`或完整格式的`&{$sub_ref}`，某些时候必须使用完整格式   

注意，使用`&`前缀的副作用，它会跳过原型检测，且不手动传递参数时，将会把当前所在环境的`@_`作为参数传递给子程序。

例如：

```perl
sub subname{
  say "@_";
}

my $sub_ref = \&subname;
&$sub_ref(1,2,3);
$sub_ref->(11,22,33);

sub f{
  &$sub_ref;
}

f(111,222,333);
```

输出结果：

```
1 2 3
11 22 33
111 222 333
```

由于子程序的引用是一个标量变量，因此子程序的引用可以保存到变量、保存到数组或hash。

例如：

```perl
sub subname{say "@_"}
my $sub_ref = \&subname;

# 子程序引用保存到数组
my @arr = ($sub_ref);
$arr[0]->(1,2,3);  # 调用子程序
&{$arr[0]}(1,2,3); # 必须使用完整格式，不能使用&$arr[0](1,2,3)

# 子程序引用保存到hash
my %h = (
  sub1 => $sub_ref,
);
$h{sub1}->(1,2,3);
&{$h{sub1}}(1,2,3); # 必须使用完整格式
```

不仅可以将子程序引用保存起来，还可以将子程序引用作为参数传递给函数、作为函数返回值返回。

例如：

```perl
sub subname{say "@_"}
my $sub_ref = \&subname;

# 子程序引用作为参数传递给函数
# 第一个参数是子程序引用，剩余参数传递给该子程序
sub call_sub_ref{
  my $sub_ref = shift;
  $sub_ref->(@_);  # 或者直接 &$sub_ref;
} 
call_sub_ref($sub_ref, 11, 22, 33);

# 子程序引用作为函数返回值
sub return_sub_ref{
  sub inner_sub{ say "@_"; }
  return \&inner_sub;
}
my $sub = return_sub_ref;
$sub->(111,222,333);
```

### 匿名子程序

和匿名数组、匿名hash类似，匿名子程序也是引用。

```perl
# 创建匿名子程序的方式
sub {
  ...
}
```

一般情况下，会将匿名子程序赋值给某个变量，那么这个变量就是一个子程序引用。

```perl
my $sub = sub {
  say "@_";
}

$sub->(1,2,3);
```

也可以将匿名子程序放进数组、hash：

```perl
# 匿名子程序放进hash
my %h = (
  sub1 => sub { say "sub1: @_"; },
  sub2 => sub { say "sub2: @_"; },
);

$h{sub2}->(11,22,33);
```

还可以将匿名子程序作为参数传递给子程序，或者当作子程序返回值被返回：

```perl
# 作为参数被传递
sub subname {
  my $sub = shift;
  $sub->(11,22,33);
}
subname(sub{say "@_";});

# 返回匿名子程序
sub subname {
  return sub {
    say "@_";
  }
}
my $sub = subname();
$sub->(11,22,33);
```

### Perl子程序的引用计数

Perl中的子程序和标量、数组等类似，它也是保存在内存中实实在在的数据。这和其他语言可能有所区别。

实际上，Perl子程序也有引用计数值:

```perl
sub sub1{say "hello";}  # 内存中子程序的引用计数为1
my $s = \&sub1;         # 内存中子程序的引用计数为2
{
  my $ss = $s;          # 内存中子程序的引用计数为3
}  # 内存中子程序的引用计数为2
```

当子程序的引用计数值减为0，该子程序将被销毁。

例如：

```perl
{
  my $sub = sub { say "hello"; }
}
$sub->();   # 报错
```

当离开大括号语句块时，my声明的私有变量`$sub`被销毁，使得匿名子程序的引用计数减为0，它被销毁。

需要注意的是，具名子程序的名称是非私有的变量名称，它相当于没有使用my声明的全局子程序名称：

```perl
{
  sub subname1{say "hello";}
}
subname1();  # 不报错

sub sub1{
  sub subname2{say "hello";}
}
subname2();  # 不报错
```

即使离开了作用域，subname1和subname2子程序仍然有效。