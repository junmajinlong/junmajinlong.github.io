## 检查引用的类型

有时候可能会需要检查引用是什么类型的(主要是在定义函数时使用)，从而确保期待数组引用时，传递的确实是数组引用，而不是hash引用。

`ref`函数可用来检查引用的类型，并返回类型。perl中内置了如下几种引用类型，**如果检查的不是引用，则返回undef**。

```
SCALAR
ARRAY
HASH
CODE
REF
GLOB
LVALUE
FORMAT
IO
VSTRING
Regexp
```

例如：
```perl
my @name=qw(longshuai wugui);
my $ref_name=\@name;

my %myhash=(
    longshuai => "18012345678",
    xiaofang  => "17012345678",
    wugui     => "16012345678",
    tuner     => "15012345678"
);
my $ref_myhash =\%myhash;

say ref $ref_name;     # ARRAY
sya ref $ref_myhash;   # HASH
```

因此，可以对传入的引用进行判断：
```perl
my $ref_type = ref $ref_hash;
say "expect HASH reference" unless $ref_type eq 'HASH';
```

上面的判断方式中，是将HASH字符串硬编码到代码中的。如果不想硬编码，可以让ref对空hash、空数组等进行检测，然后对比。
```perl
ref []   # 返回ARRAY
ref {}   # 返回HASH
```

例如：
```perl
my $ref_type=ref $ref_hash;
say "expect HASH reference" unless $ref_type eq ref {};
```

或者，将HASH、ARRAY这样的引用类型名定义为常量：
```perl
use constant HASH => ref {};
use constant ARRAY => ref [];

say "expect HASH reference" unless $ref_type eq HASH;
say "expect Array reference" unless $ref_type eq ARRAY;
```

除了`ref`函数，Perl模块`Scalar::Util`提供的`reftype`函数也用来检测类型，它还适用于对象相关的检测。

