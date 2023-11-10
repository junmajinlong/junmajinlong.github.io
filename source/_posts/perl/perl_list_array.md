---
title: Perl的列表和数组
p: perl/perl_list_array.md
date: 2019-07-06 17:37:44
tags: Perl
categories: Perl
---

# Perl的列表和数组

## Perl列表

- 使用括号包围的元素，括号中的元素使用逗号隔开的是列表。  
- 列表中的元素可以是字符串、数值、undef或它们的混合。
- 列表中的字符串元素需要使用引号包围。
- 空列表是括号中什么都没有的列表，**空列表返回的是undef**。但是赋值给别人时，不会当作undef，而是什么都没有(见稍后的例子)。
```
(1,2,"perl","python")
("var1","var2","var3")
()       # 空列表
```
```
if(defined(())){        # 对空列表()进行判断
    print "defined";
}else{
    print "undefined";  # 返回此行
}
```

- Perl将忽略构造列表时多余的逗号
```
(1,2,3,)      # 等价于(1,2,3)
(1,2,,,4)     # 等价于(1,2,4)
```

- Perl会将多个列表压扁(flatten)成一个大列表。例如：  
  - `((1,2,3),("a","b","c"))`会压扁成`(1,2,3,"a","b","c")`  
  - 这个特性用在很多地方，需注意。

- 列表中的元素支持范围操作，使用(..)即可。如果是小数，则取整。也支持负数。
- 范围操作不支持逆序范围，但不会报错。注意，不是undef，所以也就不是空列表。
- Perl的范围操作符强大的逆天。  
```
(1..5)        # 等价于(1,2,3,4,5)
(1,2,3..5,7)  # 等价于(1,2,3,4,5,7)
(3.5..6)      # 等价于(3,4,5,6)
(3.5..6.5)    # 等价于(3,4,5,6)
(-3..2)       # 等价于(-3,-2,-1,0,1,2)
($m..$n)      # 范围由$m和$n决定
(a..d)        # ("a","b","c","d")
/^one/ .. /^two/ # 匹配从one开头的行到two开头的行
/^one/ .. eof    # 匹配从one开头的行到最后一行
```

其实范围操作符有两种：`..`和`...`，它们只在匹配范围的时候有区别。例如一段文本内容为：
```
start here end here
second line
end
```
如果使用`/start/../end/`，那么在第一行就结束了匹配，因为第一行能同时匹配start和end，如果想要第一行匹配start，不再匹配end，而是匹配第三行的end，那么需要使用`/start/.../end/`。实际上，范围操作符的左边被匹配了称为flip，右边被匹配了称为flop，三个点的范围操作符`...`表示flip之后不要立即做flop评估，而是在本次之后评估flop。

## qw简写列表

使用qw写列表，无需加引号，无需加逗号分隔，而是使用空格分隔。

以下两个列表等价：
```
("perl","python","shell")
qw(perl python shell)
```

- qw里虽然不用写引号，但它相当于单引号。所以qw里面不能变量替换，不能反斜线序列(如`\n`)
```
$perl="perl";
qw($perl python shell\n)   # 不会做任何转换，包含三个元素"$perl"、"python"、"shell\n"
```

- 因为会丢弃所有空白，所以可以直接换行
```
qw(
    perl
    python
    shell
)
```
注意，不能在qw换行内部加上注释`#`。所以，下面`#`和它后面的字符串也被当作列表元素
```
qw(
    perl
    python   # This is python
    shell
)
```

- qw后面的定界符可以换成其它的，只要对应就可以。如果是括号类的，只要能结尾就可以
```
qw/ var1 var2 var3 /
qw! var1 var2 var3 !
qw% var1 var2 var3 %
qw# var1 var2 var3 #      # 这个有点像注释
...  
qw( var1 var2 var3 )
qw< var1 var2 var3 >
qw{ var1 var2 var3 }
qw[ var1 var2 var3 ]
```
如果定界符中间要包含定界符符号，只要转义就可以了。
```
qw/ var1 var2 \/var3 var4 /
```
更好的方法是换成其它定界符。
```
qw{
    /root/xyz.sh
    /etc/xyz.sh
}
```

## 列表的赋值

- perl中可以通过列表一一对应地给(标量)变量赋值。
```
($perl,$python)=("perl","python");
print $perl,"\n";
print $python,"\n";
```

- 赋值前，总是先计算出等号右边的值，然后再赋给左边。所以交换变量非常容易。
```
($perl,$python)=($python,$perl);
```

- 如果列表赋值时，等号左右两边元素个数不等，则采取如下方案：
    - 如果右边元素更多，则忽略多出来的元素；
    - 如果左边变量元素更多，则多出来的变量赋值为undef；
```
($perl,$python,$shell)     =qw(perl python shell php);   # php被忽略
($perl,$python,$shell,$php)=qw(perl python shell);  # $php被赋值为undef
```

- perl中，列表和数组紧密关联。可以认为将列表赋值给一个变量，这个变量就是数组。数组使用`@`符号开头。
```
print qw{perl python shell};
@subject=qw{perl python shell};
print @subject;
```
```
@subject=("perl","python","shell");
@range=1..5;
@subject=(@subject,"hello",@range);
```

- 空列表和列表中的undef元素：
    - 空列表自身是一个未定义列表，所以它自身返回undef
    - **未定义的数组，或者未定义的列表就是空列表`()`，因为未定义，所以它自身返回undef**
    - 将空列表或者未定义的数组作为元素添加到其它列表、数组中时，等于什么都没做，直接被忽略，因为它是没定义过的  
    - undef元素是一个实实在在的元素，会占用列表、数组空间  

```
@empty=();
if(@empty){
    print "valid\n";
}else{
    print "invalid\n";    # 返回此行
}

@subject1=qw{perl python shell};
@subject2=(@subject,@empty,"php");  # 等价于(@subject,"php")
@subject3=(@subject,undef,"php");   # 不等价于(@subject,"php")
```


## Perl数组

- 数组和列表中的元素，可以是字符串、数值、undef或它们的混合，总之每个元素都必须是标量(即不能嵌套数组、列表)。
- 列表可以单独存在，但数组必须是由列表组成的，虽然它们都允许为空。
- 列表或数组中的元素， **字符串类型不需要使用引号包围** (看下面的例子)。
- perl中的数组使用`@`开头，引用数组中元素的时候，使用`$`开头，并带上数组下标索引，就像变量一样，其实就是引用数组变量(看下面的例子)。
- 数组变量和普通变量的名称属于不通的名称空间(namespace)，所以perl中它们不会冲突，但对于人类来说，容易搞混。
- 双引号包围数组时，会进行数组元素替换，每个元素之间会自带空格。如果不加引号包围，输出的时候，元素之间紧密相连，没有空格。
```
$arr="xyz";
@arr=(var0,var1,var2);    # 这是一个列表，赋值给了数组。
                          # 列表中的元素没有加引号包围
print $arr[0],$arr[1],$arr[2],"\n";   # 引用数组变量
print $arr;             # 输出xyz
print @arr,"\n";        # 输出var0var1var2
print "@arr","\n"       # 输出var0 var1 var2
```

- 引用数组变量时，如果数组越界，不会报错，而是返回undef值。但不会自动填充中间的元素。
- 数组索引可以是小数或负数，如果是小数，则自动取整。如果是负数，则从后向前取，超出数组范围的负数，将返回undef。
- `$#arr`表示最后一个索引值，它比数组元素个数小1(因为有个0索引)。所以`$arr[$#arr]`引用的是最后一个元素(`$arr[-1]`也是最后一个元素)。
```
print $arr[1.3];     # 等价于$arr[1]
print $arr[-1];      # 最后一个元素
print $arr[-2];      # 倒数第二个元素
print $arr[100];     # undef
print $arr[-100];    # undef
print $arr[$#arr];   # 最后一个元素
print $arr[$#arr-1]; # 倒数第二个元素
```

- 数组的标准引用方式是带上大括号。例如`@{arr}`、$#{arr}，这样的引用方式可以避免歧义。
```
@{arr1}=(var0,var1,var2)
```

- perl中的数组在定义新元素的时候，会 **自动填补中间缺少的元素** ，这些元素被定义为undef。
```
@arr2=(var0,var1,var2);
print $arr2[100];      # undef
print $#arr2;          # 返回2
$arr2[5]="var5"        # 将填充$arr2[3],$arr2[4]为undef
print $#arr2;          # 返回5
```

- 可以直接引用数组，如`@arr`。
- perl中有很多种上下文环境，最常出现的是 **标量上下文和列表上下文 。perl中会根据不同上下文进行转换，从而获取不一样的值**。后面会解释这两种上下文。

注意下面的例子中`print @arr,"\n"`和`print @arr."\n"`，仅仅是一个逗号和点号的区别，返回结果却完全不同。这就是不同上下文环境中的执行结果。
```
@arr=(var0,var1,var2);  # 数组，里面的元素不用引号包围
$arr[4]="var4";         # 直接对数组变量赋值，会自动填充$arr[3]:undef
print @arr,"\n";        # 引用整个数组：var0var1var2var4
print @arr."\n";        # 输出的是5，表示的是数组中元素个数
                        # 因为这里的数组从列表上下文切换成了标量上下文，
                        # 数组将返回元素个数
```

<a name="blog111"></a>
## 增、删数组元素

### pop、push

push函数：向数组尾部(最右边)插入元素；
pop函数：弹出(移除并返回)数组尾部的元素；当没有元素可弹时，则pop返回undef。

pop和push用于实现堆栈，像堆盘子一样，不断往上堆，也从上面开始取。

```
@arr = 5..9;                   # (5,6,7,8,9)
$var1 = pop(@arr);
print @arr,"\n",$var1,"\n";    # 返回5678和9

$var2 = pop @arr;
print @arr,"\n",$var2,"\n";    # 返回567和8

pop @arr;                      # 直接丢弃弹出的元素
print @arr,"\n";               # 返回56
```

倒数第二行的`pop @arr`没有做任何赋值，perl中有很多这种用法，**当有返回内容时，却不对其赋值，表示仅执行操作，但丢弃操作后的返回结果**，在perl中，这种称为【空上下文】(void context)。所以，这里可以表示直接删除数组的最后一个元素。

```
@arr=(5,6);
push(@arr,0);     # @arr现在是(5,6,0)
push @arr,1..10;  # @arr现在有13个元素：5,6,0,1,2,3,4,5,6,7,8,9,10
@other=qw{1 3 4 5};
push @arr,@other; # @arr现在17个元素：5,6,0,1,2,3,4,5,6,7,8,9,10,1,3,4,5
```

### shift和unshift

push和pop是操作数组尾部，shift和unshift则是操作数组头部：
    - shift弹出(移除并返回)数组的第一个元素(最左边)，当没有元素可弹时，shift将返回undef；
    - unshift向数组最左边插入一个元素。

```
@arr = qw{perl python shell};
$m   = shift(@arr);   # $m变成"perl",@arr现在是("python","shell")
$n   = shift @arr;    # $n变成"python",@arr现在是("shell")
shift @arr;           # @arr变成空了
%i   = shift @arr;    # $i为undef，@arr还是空
unshift @arr,5;       # @arr现在是(5)
unshift @arr,4;       # @arr现在是(4,5)
@other=1..3;
unshift @arr,@other;  # @arr现在变为(1,2,3,4,5)
```

### splice()函数

shift-unshift和pop-push操作的是数组首尾部。如果想要操作数组的中间某个元素，则需要使用splice()函数。

splice()函数有4个参数，后两个参数可选。
- 第一个参数指定要操作的数组；
- 第二个参数指定从哪个索引位置开始操作；
- 第三个参数指定要操作的长度；
- 第四个参数指定替换原数组的数据；

如果只给两个参数，则从索引位置开始，一直删除到数组结尾。
```
# 两个参数
@arr  = qw{perl python shell php java};
@arr1 = splice @arr,2;             # @arr现在是qw{perl python}
                                   # @arr1现在是qw{shell php java}

# 三个参数
@arr  = qw{perl python shell php java};
@arr1 = splice @arr,2,2;           # @arr现在是qw{perl python java}
                                   # @arr1现在是qw{shell php}

# 四个参数
@arr  = qw{perl python shell php java};
@arr1 = splice @arr,2,2,qw{ruby};    # @arr现在是qw{perl python ruby java}
                                     # @arr1现在是qw{shell php}
```

在使用四个参数的时候，如果将第三个参数设置为0，则可以不用做任何删除就向数组中插入新元素。

```
@arr  = qw{perl python shell php java};
@arr1 = splice @arr,2,0,qw{ruby};   # @arr现在是qw{perl python ruby shell php java}
                                    # @arr1现在是qw()
```

## foreach遍历

foreach可以遍历列表或数组。

```
foreach $subject qw(perl python shell){
    print $subject,"\n";
}
```

其中$subject是控制变量，每个迭代过程中，都会从列表或数组中取出一个元素赋值给控制变量。

迭代数组：
```
@arr=qw(perl python shell);
foreach $subject (@arr){
    print $subject,"\n";
}
```

迭代过程中，赋值给控制变量的其实是列表或数组中的引用，并非是直接取出元素的值赋值给控制变量的(也就是说，不是变量复制)。因此，**在foreach过程中修改控制变量的值也会修改列表/数组中元素的值**。
```
@arr=qw(perl python shell);
foreach $subject (@arr) {
    $subject .= "aaa";
}
print @arr;         # 输出perlaaapythonaaashellaaa
```

**当foreach遍历结束后，控制变量将复原为foreach遍历前的值**(例如未定义的是undef)。
```
$subject="php";
foreach $subject (qw(perl python shell)){
}
print $subject;     # 输出"php"
```


## 默认变量`$_`

在foreach遍历结构中也可以省略控制变量部分，这时会使用默认的变量`$_`作为控制变量，每次迭代都会从数组中取出一个列表/数组的索引赋值给`$_`。
```
foreach (qw(perl python shell)) {
    print $_,"\n";
}
```

其实，perl中很多地方没有【需求数据】时，都会自动使用`$_`。例如下面的print函数就使用了`$_`。
```
$_="hello world!\n";
print;
```

## sort()和reverse()

- sort()用于对列表/数组进行升序排序，默认按照unicode(包含ascii)顺序：`数字<大写字母<小写字母`。在以后，还可以自定义排序规则。
- reverse()则是对列表/数组的当前顺序进行反转。
- sort()和reverse()的排序/反转的效果是体现在返回值中的，它不会改变原始列表/数组中元素的顺序。

```
print reverse 11,6..8,20,"\n";    # 输出顺序：20 8 7 6 11

@arr=qw(perl python shell);
reverse @arr;            # 反转了顺序，但返回结果丢弃了，@arr不变
print @arr,"\n";         # 返回顺序：perl python shell

@arr1=reverse @arr;      # @arr1中元素的顺序：shell python perl
print '@arr1: ',@arr1,"\n";

@sort_arr=sort @arr;     # 排序，但因为@arr中元素顺序本就是升序的
print '@sort_arr: ',@sort_arr,"\n";  # 返回顺序：perl python shell

sort @arr;               # 虽排序，但丢弃了排序结果
```

## each()

each操作符可以获取一个键值对(key/value)。可以操作数组，也可以操作hash。each操作hash的内容见后文。

each对数组操作时，返回的是数组中元素的索引号和该元素的值。
```
my @arr=qw(perl python shell);
while (my($index,$value) = each(@arr)){
    print "$index: $value","\n";
}
```
其中my关键字表示该变量是局部变量。以下是执行结果：
```
0: perl
1: python
2: shell
```

如果不使用each，则需要遍历数组，并获取数组的索引以及元素的值。
```
@arr=qw(perl python shell);
foreach $index (0..$#{arr}) {
    print "$index: $arr[$index]","\n";
}
```
