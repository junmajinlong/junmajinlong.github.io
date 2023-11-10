---
title: Perl函数：数组和hash相关函数
p: perl/perl_arr_hash_funcs.md
date: 2019-07-07 17:39:43
tags: Perl
categories: Perl
---

# Perl函数：数组和hash相关函数


Perl数组和hash相关函数

内置的数组函数有：
```
each, keys, pop, push, shift, splice, unshift, values
```

内置的hash函数有：
```
delete, each, exists, keys, values
```

有些是重复的。所以放在一起解释。

数组相关函数：  
- push：将单元素或一个列表追加到数组的尾部，返回追加后的数组长度具体示例[push](#blogpush)  
- pop：删除数组中的最后一个元素，返回被pop掉的元素，具体示例[top](#blogpop)  
- unshift：将单元素或一个列表追加到数组的首部，返回追加后的数组长度具体示例[push](#blogpush)  
- shift：删除数组中的第一个元素，返回被shift掉的元素，具体示例[top](#blogpop)  
- splice：从指定位置处删除、插入元素，具体示例[splice](#blogsplice)  

hash相关函数：(比较简单，所以不介绍)  
- delete：删除给定key  
- exists：测试key是否存在于hash中  

共同函数：(比较简单，所以不介绍)  
- keys：获取hash或数组(5.12版本才开始提供)的key，对于数组，则是返回索引位置。对于hash，返回的key的顺序是不确定的  
- values：返回hash或数组(5.12版本才开始提供)的值。values返回的元素顺序和keys返回的顺序是一致的  
- each：遍历hash或数组  

<a name="blogpush"></a>

## push & unshift

将一个列表追加到一个数组的尾部，返回追加后数组的长度。
```
push ARRAY,LIST
unshift ARRAY,LIST
```

- push是将一个列表(可能只有一个元素)追加到数组的最尾部  
- unshift则是将一个列表(可能只有一个元素)追加到数组的最前面  
- 它们返回追加后数组的长度  
- 追加列表的时候，不是每次追加一个元素，而是一次追加一整个列表，所以新追加到数组中的元素顺序保持不变  

注意，unshift在数组开头追加一个列表，会导致数组中原有的元素索引整体后移。例如原数组内容为`a b c d`，unshift一个新元素后，a就变成第二个元素，索引位置从0变成1，b的索引位置从1变成2,c的索引位置从2变成3，d的索引位置从3变成4。在数组较小时，unshift并不会有什么影响，但数组较大，会严重影响效率。

例如：

```
@arr=qw(python shell php);
push @arr,"perl","Ruby";
print "@arr";         # 输出：python shell php perl Ruby
```
push的返回值为追加元素成功后，数组的长度：
```
@arr=qw(python shell php);
print push @arr,"perl","Ruby";  # 输出：5
```

pop操作，在结果上等价于：
```
for my $value (@LIST){
    $ARRAY[++$#ARRAY] = $value;
}
```
但push效率更高，因为它是一次追加一整个列表，而非一次追加一个列表中的元素。


<a name="blogpop"></a>

## pop & shift

```
pop ARRAY
pop
shift ARRAY
shift
```

- pop从数组中移除并返回最后一个元素
- shift从数组中移除并返回第一个元素  
- 如果数组已空，则pop/shift返回undef  
- 如果省略ARRAY，则pop/shift操作的是`@ARGV`，但如果是在子程序中，则操作的是`@_`

注意，shift删除第一个元素会导致数组的索引位置整体向前移动。例如数组内容为`a b c d`，但shift一次后，b就变成了第一个元素，它的索引位置从1变成0，c的索引位置从2变成1，d的索引位置从3变成2。在数组较小时，shift并不会有什么影响，但数组较大，会严重影响效率。

例如：循环删除数组的最后一个元素
```
@arr=qw(python shell php);
while(my $poped = pop @arr){
    print $poped,"\n";     # 依次输出：php shell python
}
```

当省略ARRAY时，且不在子程序中时，pop将操作`@ARGV`。
```
while(my $poped=pop){
    print $poped,"\n";
}
```
执行它：
```
$ ./test.pl a.txt b.txt c.txt
c.txt
b.txt
a.txt
```

同理，在子程序中，则操作默认的参数数组`@_`。

```
sub mysub() {
    while(my $poped=pop){
        print $poped,"\n";
    }
}

&mysub(qw(a.txt b.txt c.txt));    # 将依次输出c.txt b.txt a.txt
```

<a name="blogsplice"></a>

## splice

pop/push、unshift/shift操作的都是数组的开头，或者末尾。splice(中文译为：粘接)则可以指定操作数组中的哪个位置。

```
splice ARRAY
splice ARRAY,OFFSET
splice ARRAY,OFFSET,LENGTH
splice ARRAY,OFFSET,LENGTH,LIST
```

- splice在移除元素时，在列表上下文返回被移除的元素列表，标量上下文返回最后一个被移除的元素  

**1.一个参数时，即`splice ARRAY`，表示清空ARRAY**。

```
use 5.010;
@arr=qw(perl py php shell);

@new_arr=splice @arr;
say "original arr: @arr";   # 输出：空
say "new arr: @new_arr";    # 输出原列表内容
```

如果splice在标量上下文，则返回最后一个被移除的元素：
```
use 5.010;
@arr=qw(perl py php shell);

$new_arr=splice @arr;
say "$new_arr";    # 输出：shell
```

**2.两个参数时，即`splice ARRAY,OFFSET`，表示从OFFSET处开始删除元素直到结尾**。  

注意，OFFSET可以是负数。

例如：
```
use 5.010;
@arr=qw(perl py php shell);

@new_arr=splice @arr,2;
say "original arr: @arr";   # 输出：perl py
say "new arr: @new_arr";    # 输出：php shell
```

如果offset为负数，则表示从后向前数第几个元素，-1表示最后一个元素。
```
use 5.010;
@arr=qw(perl py php shell);

@new_arr=splice @arr,-3;
say "original arr: @arr";    # 输出：perl
say "new arr: @new_arr";     # 输出：py php shell
```

**3.三个参数时，即`splice ARRAY,OFFSET,LENGTH`，表示从OFFSET处开始向后删除LENGTH个元素**。  

注意，LENGTH可以为负数，也可以为0，它们都有奇效。

例如：
```
use 5.010;
@arr=qw(perl py php shell ruby);

@new_arr=splice @arr,2,2;
say "original arr: @arr";   # 输出：perl py ruby
say "new arr: @new_arr";    # 输出：php shell
```

**如果length为负数(假设为-3)，则表示从offset处开始删除，直到尾部还保留-length个元素**(-3时，即表示尾部保留3个元素)。例如：
```
use 5.010;
@arr=qw(perl py php shell ruby java c c++ js);

@new_arr=splice @arr,2,-2;   # 从php开始删除，最后只保留c++和js两个元素
say "original arr: @arr";    # 输出：perl py c++ js
say "new arr: @new_arr";     # 输出：php shell ruby java c
```

如果正数length的长度超出了数组边界，则删除结尾。如果负数length超出了边界，也就是保留的数量比要删除的数量还要多，这时保留优先级更高，也就是不会删除。例如，从某个位置开始删除，后面还有2个元素，但如果`length=-2`，则这两个元素不会被删除。

**如果length为0，则表示不删除**，这个在有第4个参数LIST时有用。

**4.四个参数时，即`splice ARRAY,OFFSET,LENGTH,LIST`**，表示将LIST插入到删除的位置，也就是替换数组的部分位置连续的元素。

例如：
```
use 5.010;
@arr=qw(perl py php shell ruby);
@list=qw(java c);

@new_arr=splice @arr,2,2,@list;
say "original arr: @arr";   # 输出：perl py java c ruby
say "new arr: @new_arr";    # 输出：php shell
```

如果想原地插入新元素，而不删除任何元素，可以将length设置为0，它会将新列表插入到offset的位置。
```
use 5.010;
@arr=qw(perl py php shell ruby);
@list=qw(java c);

@new_arr=splice @arr,2,0,@list;
say "original arr: @arr";   # 输出：perl py java c php shell ruby
say "new arr: @new_arr";    # 输出：空
```

注意上面php在新插入元素的后面。


splice功能非常好用，在不少语言中，要实现类似的功能，需要借助链表的方式来实现，比较复杂。splice一个函数即可，方便的多，但链表毕竟增、删元素的效率高，所以对于大数组又要频繁增删改的时候，还是是现成链表比较好。




