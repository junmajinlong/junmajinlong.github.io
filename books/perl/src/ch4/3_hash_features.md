## 了解hash结构的基本特性

介绍了Perl中如何使用hash类型后，有必要了解一些关于hash的基本特性。

**hash结构中的key是唯一的**。在将键值对存储到hash结构时，会对key进行hash计算，然后根据计算得到的hash值决定该键值对的值存储在何处，由于相同的key总是计算得到相同的hash值，因此先后两次存储key相同的键值对时，后存储的值将覆盖已存储的值。

```perl
my %person = (
  name => "junmajinlong",
  age => 23,
);

$person{name} = "junma";  # 覆盖name键映射的值
```

但是，不同的key也可能会计算出相同的hash值，这时将造成hash冲突(hash碰撞)问题：不同的键值对将存储在同一个位置。虽然hash冲突的计算较低，但仍然需要提供hash冲突时的解决方案，hash冲突的解决方案有多种，不同语言采用不同的策略。

**hash结构不保证键值对的顺序**，比如遍历时的顺序是不可预测的，并且插入新的键值对可能还会改变顺序，因此不要依赖hash的键值对顺序。另外，有些语言实现了按照键值对存储时的先后顺序进行遍历。

```perl
my %person = (
  name => "junmajinlong",
  age => 23,
);
my @keys = keys %person;
say "@keys";   # name age

$person{gender} = "male";
@keys = keys %person;
say "@keys";   # name gender age
```

**hash结构的内存空间利用率不高**。hash结构会划分hash桶(hash bucket)，每个桶都预分配一些空间槽(slot)，槽的数量决定了一个桶中最多能存放多少数据。每次存储键值对时，都将根据对key计算出来的hash值决定value存放在哪个桶以及桶的哪个位置(slot)。

**hash结构的搜索速度和增删键值对的速度很快，且不会随着所存储键值对元素数量的增长而变慢，它由hash桶的大小决定**。而数组的平均搜索速度则会随着元素的增长而逐渐变慢。

但是，**当某次向hash中存储键值对时因空间不够而触发了扩容，速度会很慢**，因为扩容时需要迁移整个hash结构，包括对所有的key进行rehash、拷贝内存数据。因此，一次性存储大量键值对时，会明显感受到长久的耗时。

例如，下面构建1000W个元素的hash耗时6.8秒，构建1000W个元素的数组只需0.7秒。

```perl
use 5.012;
use Time::HiRes qw(time);

my $num = 10_000_000;

my %h;
my $h_start = time;
for my $i (1..$num){ $h{$i} = 1; }
my $h_end = time;
my $h_diff = $h_end - $h_start;
say "time diff: $h_diff";

my @arr;
my $a_start = time;
for my $i (1..$num){ $arr[$i-1] = 1; }
my $a_end = time;
my $a_diff = $a_end - $a_start;
say "time diff: $a_diff";
```

