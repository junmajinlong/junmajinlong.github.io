## 操作数组

在Perl中，数组、列表应用非常广泛，也经常需要对数组和列表进行一些操作。

列表常见操作包括：grep、join、map、reverse、sort、unpack、x操作符执行列表重复，等等。另外，标准库`List::Utils`中也提供了很多常见的列表操作，如reduce、first、any、sum、uniq、shuffle等。

多数时候，数组可以当作列表来使用，原因是操作列表的地方期待一个列表，即处在列表上下文，perl会隐式地将数组转换为列表。因此上述列表操作多数也适用于数组。

但数组是Perl的内置数据结构，是直接暴露给编程人员的数据类型，它可以转换为列表，但它不是为了转换成列表而存在的，它有自己的角色，有一些操作只能应用于数组而不能应用于列表。

数组常见的操作包括：each、pop、push、shift、unshift、keys、values、splice。

本节将介绍数组的常见操作，列表相关操作将留给后文。

### keys、values

- keys函数在列表上下文返回数组的所有索引或hash的所有key，在标量上下文返回数组或hash的元素数量  

- values函数在列表上下文返回数组的所有元素或hash的所有value，在标量上下文返回数组或hash的元素数量  

以数组为例：

```perl
my @arr = (11, 22, 33, 44);
# 列表上下文
my @arr_keys = keys @arr;
my @arr_values = values @arr;

say "@arr_keys";    # 0 1 2 3
say "@arr_values";  # 11 22 33 44
```

可以通过keys或values函数来迭代数组、hash的索引或值。以数组为例：

```perl
my @arr = (11, 22, 33, 44);
for(keys @arr){say $_;}   # 0 1 2 3
for(values @arr){ say $_; } # 11 22 33 44
```

每次调用keys和values都会重置Perl内部维护的迭代指针，因此使用keys或values来迭代数组、hash是安全的，它们总能取得数组或hash的所有元素。

另外，values获取的值是对元素的引用，因此修改values获取的值，也将修改源数据。

```perl
my @arr = (11, 22, 33, 44);
my @arr_values = values @arr;
$arr_values[0] = 111;
say "@arr_values";   # 111 22 33 44
```

### pop push shift unshift

- pop从数组中移除并返回最后一个元素，数组为空则返回undef  
- push向数组尾部追加一个元素或一个列表，返回追加完成后数组长度  
- shift移除并返回数组第一个元素，数组为空则返回undef  
- unshift向数组头部添加一个元素或一个列表，返回追加完成后数组长度  

对于pop、shift：

```perl
# pop
my @arr1 = (11,22,33);
say pop @arr1;  # 33
say pop @arr1;  # 22
say pop @arr1;  # 11
say pop @arr1;  # undef，警告模式下会给出警告

# shift
my @arr2 = (11,22,33);
say shift @arr2;  # 11
say shift @arr2;  # 22
say shift @arr2;  # 33
say shift @arr2;  # undef，警告模式下会给出警告
```

对于push、unshift：

```perl
# push
my @arr1 = (11,22,33);
push @arr1, 44;      # 追加单个元素
push @arr1, 55, 66;  # 追加列表，列表的小括号被省略
say push @arr1, (77,88); # 输出8，push返回数组长度

say "@arr1";  # 11 22 33 44 55 66 77 88

# unshift
my @arr2 = (11,22,33);
unshift @arr2, 'a', 'b';
unshift @arr2, qw(aa bb);
say "@arr2"; # aa bb a b 11 22 33
```

push追加在效果上等价于如下代码，但效率更高：

```perl
my @arr;
for my $v (@arr) {
  # 向数组尾部追加一个元素
  $arr[++$#arr] = $v;  # 或者：$arr[~~@arr]=$v;
}
```

注意，Perl的数组结构和其他语言的数组结构有所不同，其他语言中，这四个函数的效率通常会比使用索引操作方式的效率要更低，但在Perl中，这四个函数的效率比使用索引的处理方式更高，这可能会让人难以置信。原因在于perl已经将这几个函数转换为opcode的方式，它们直接通过C数组索引访问底层的C数组，而如果使用Perl的索引，则先在Perl层面找到P数组的索引位置，然后再访问对应的C数组。不过，多数时候不需要考虑这些操作的效率问题。

另外，**shift会导致数组元素向前挪位，unshift会导致数组元素先后挪位。在其他语言中，这样的挪位操作效率非常低，但是在Perl中，这两个操作没有任何效率损失**。所以，请放心使用它们。

### each

在上一小节已经详细介绍each的使用，参考：[小心使用each](./5_iterator.md#perl_each)。

### splice

pop/push、unshift/shift操作的都是数组的开头或者末尾。splice则可以指定操作数组中的哪个位置。

```
splice ARRAY
splice ARRAY,OFFSET
splice ARRAY,OFFSET,LENGTH
splice ARRAY,OFFSET,LENGTH,LIST
```

splice在移除元素时，在列表上下文返回被移除的元素列表，标量上下文返回最后一个被移除的元素  

**1.一个参数时，即`splice ARRAY`，表示清空ARRAY**。

```perl
use 5.010;
@arr=qw(perl py php shell);

@new_arr=splice @arr;
say "original arr: @arr";   # 输出：空
say "new arr: @new_arr";    # 输出原列表内容
```

如果splice在标量上下文，则返回最后一个被移除的元素：
```perl
use 5.010;
@arr=qw(perl py php shell);

$new_arr=splice @arr;
say "$new_arr";    # 输出：shell
```

**2.两个参数时，即`splice ARRAY,OFFSET`，表示从OFFSET处开始删除元素直到结尾**。  

注意，OFFSET可以是负数。

例如：
```perl
use 5.010;
@arr=qw(perl py php shell);

@new_arr=splice @arr,2;
say "original arr: @arr";   # 输出：perl py
say "new arr: @new_arr";    # 输出：php shell
```

如果offset为负数，则表示从后向前数第几个元素，-1表示最后一个元素。
```perl
use 5.010;
@arr=qw(perl py php shell);

@new_arr=splice @arr,-3;
say "original arr: @arr";    # 输出：perl
say "new arr: @new_arr";     # 输出：py php shell
```

**3.三个参数时，即`splice ARRAY,OFFSET,LENGTH`，表示从OFFSET处开始向后删除LENGTH个元素**。  

注意，LENGTH可以为负数，也可以为0，它们都有奇效。

例如：
```perl
use 5.010;
@arr=qw(perl py php shell ruby);

@new_arr=splice @arr,2,2;
say "original arr: @arr";   # 输出：perl py ruby
say "new arr: @new_arr";    # 输出：php shell
```

**如果length为负数，则表示从offset处开始删除，直到尾部还保留-length个元素**(例如length为-3时，表示尾部保留3个元素)。例如：

```perl
use 5.010;
@arr=qw(perl py php shell ruby java c c++ js);

@new_arr=splice @arr,2,-2;   # 从php开始删除，最后只保留c++和js两个元素
say "original arr: @arr";    # 输出：perl py c++ js
say "new arr: @new_arr";     # 输出：php shell ruby java c
```

如果正数length的长度超出了数组边界，则删除至结尾。如果负数length超出了边界，也就是保留的数量比要删除的数量还要多，这时保留优先级更高，也就是不会删除。例如，从某个位置开始删除，后面还有2个元素，但如果length=-2"，则这两个元素不会被删除。

**如果length为0，则表示不删除**，这个在有第4个参数LIST时有用。

**4.四个参数时，即`splice ARRAY,OFFSET,LENGTH,LIST`**，表示将LIST插入到删除的位置，也就是替换数组的部分位置连续的元素。

例如：
```perl
use 5.010;
@arr=qw(perl py php shell ruby);
@list=qw(java c);

@new_arr=splice @arr,2,2,@list;
say "original arr: @arr";   # 输出：perl py java c ruby
say "new arr: @new_arr";    # 输出：php shell
```

如果想原地插入新元素，而不删除任何元素，可以将length设置为0，它会将新列表插入到offset的位置。
```perl
use 5.010;
@arr=qw(perl py php shell ruby);
@list=qw(java c);

@new_arr=splice @arr,2,0,@list;
say "original arr: @arr";   # 输出：perl py java c php shell ruby
say "new arr: @new_arr";    # 输出：空
```

注意上面php在新插入元素的后面。



