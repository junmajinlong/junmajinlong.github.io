---
title: Perl的hash类型
p: perl/perl_hash.md
date: 2019-07-06 17:37:45
tags: Perl
categories: Perl
---

# Perl的hash类型

hash类型也称为字典、关联数组、映射(map)等等，其实它们都是同一种东西：键值对。每一个Key对应一个Value。

- hash会将key/value散列后，按序放进hash桶。**散列后的顺序和存放数据的顺序无关**
- hash类型的key只能是字符串，value可以是字符串、数值、undef或其它类型的标量
- hash的key必须唯一，不能重复
- perl中使用符号`%`表示hash类型，如`%myhash`。使用`$hashname{index}`访问hash中的元素
- perl中可以单独对每一个hash元素赋值，也可以以列表的方式一次性赋多个值(初始化时可用)
- 一次性赋值多个时，每个value跟在自己的key后面，用逗号分隔，每个key/value对之间也使用逗号分隔
    - 也就是说，它们的顺序是：`{键1,值1,键2,值2...}`
- perl中可用使用"胖箭头"(`=>`)替代逗号，让整个hash看起来很清晰
- 访问hash中不存在的元素，会返回undef
```
$phone_num{longshuai}="18012345678";
$phone_num{xiaofang}="17012345678";
$phone_num{fairy}="16012345678";
```
等价于以下几种方式：
```
%phone_num1=("longshuai","18012345678",   # 注意是括号，不是大括号
             "xiaofang","17012345678",
             "fairy","16012345678");

my %phone_num1=("longshuai" => "18012345678",  # 将hash声明为局部hash
                "xiaofang"  => "17012345678",
                "fairy"     => "16012345678");
```
- 使用胖箭头赋值的时候，如果key命名够规范(字母、数字、下划线)，可以省略key部分的引号，perl会自动加上。在引用hash中的元素时，也一样可省略引号。如`$phone_num{"longshuai"}`和`$phone_num{longshuai}`都有效
```
%phone_num1=(longshuai =>"18012345678",
             xiaofang  =>"17012345678",
             fairy     =>"16012345678");
```
如果key命名不够规范，则不会自动加上引号。有时候，这可能会当作一个表达式进行计算：
```
$myhash{foo.bar}    # $myhash{foobar}
```

- 可以将hash赋值给另一个hash
```
%hash_name1 = %hash_name2;
```

Perl中的这个赋值过程和一般语言不太一样，它会**先将`%hash_name2`展开成列表**，然后再将这个列表赋值给新列表`%hash_name1`。

- 可以直接输出hash，如`print %myhash`，但不能加上引号，例如`print "%myhash"`不会输出hash里的元素
```
%myhash = (key1,value1,key2,value2,key3,value3);
print %myhash,"\n";
print "%myhash","\n";
```

- perl中的ENV：perl可以通过ENV这个hash直接访问操作系统的环境变量
```
print $ENV{PATH};   # 输出操作系统的PATH环境变量
```
如果perl想访问操作系统中某个变量，可以直接在操作系统中设置，然后通过perl访问：
```
$ myvar=2;export myvar;

print $ENV{myvar};
```



## Perl hash相关函数

主要有reverse()、keys()、values()、exists()和delete()。

- 可以用reverse函数反转hash。它会将hash当作列表一样反转，然后再将其当作hash。所以，原hash的key会变成后来的value，原value会变成后来的key
```
(key1,value1,key2,value2,key3,value3)
```
反转过程中：
```
(value3,key3,value2,key2,value1,key1)
```
反转后新的hash可能之一：
```
(value2,key2,value1,key1,value3,key3)
```

因为反转为新的hash时，是以原来的value当作新的key，所以可能会有重复的新key，perl采取的是覆盖生效：后存储的覆盖先存储的。

再者，反转为新的hash时，会对新的key重新hash计算存储到hash桶里，所以反转后的顺序不一定真的是反序的。这里的reverse更注重key/value的反转。

- keys函数和values函数，分别返回key列表和value列表
- keys函数和values函数在标量上下文中返回的是列表元素的个数

```
%myhash = (key1,value1,key2,value2,key3,value3);
@keys   = keys %myhash;
@values = values %myhash;
$keys_num = keys %myhash;
print @keys,"\n";
print @values,"\n";
print $keys_num,"\n";    # 返回3
```
显然，key列表和value列表的顺序和存储的顺序可能是不一致的，但至少keys函数返回的列表中，如果key1排在最前，那么values函数返回的列表中，value1也肯定排在最前

- 只要hash中包含任何一个键值对，在于布尔值判断上就返回真
```
if(%hash){
    print "True\n";
}
```
- exists()函数判断hash中是否存在某个key
- delete()函数用于删除某个key/value，如果要删除的key/value不存在，则直接返回，不会报错



## Perl遍历hash

- each可以遍历hash。each可以遍历数组和hash，它会获取索引和对应的值
- each每次都获取一个键值对，并作上位置标记，以便下次从此开始继续遍历。换句话说，数组和hash有内部的迭代器
- foreach也可以遍历hash，但它只能通过keys函数来遍历Key，间接遍历hash
```
%myhash = (key1,value1,key2,value2,key3,value3);

# each迭代遍历
while (($key,$value) = each %myhash){
    print "$key: $value","\n";
}

# foreach迭代遍历
foreach my $key (sort keys %myhash){
    print $key,$myhash{$key},"\n";
}
```

需要注意的是each遍历，是不保证顺序的，foreach可以按照一定keys的顺序进行遍历。另外，在上面while each迭代的过程中，有几个过程：
1. `each %myhash`首先迭代第一个键值对；
2. 将获取到的第一个键值对赋值给`($key,$value)`；
3. 判断`while`的条件真假，因为赋值后得到的是一个包含键、值的列表，在while的标量上下文中，它返回列表中元素数量2，所以为真；
4. 迭代到最后一个，each迭代不到key/value，所以列表元素数量为0，while的条件返回false，不会继续执行下去。