## 逻辑运算：and(&&)、or(||)、//、not(!)

Perl包含以下几种逻辑运算操作符：

```
not expr          # 逻辑取反
expr1 and expr2   # 逻辑与
expr1  or expr2   # 逻辑或

! expr            # 逻辑取反
expr1 && expr2    # 逻辑与
expr1 || expr2    # 逻辑或

expr1 // expr2    # 逻辑定义或
```
其中：  
- `&&`运算符只有两边为真时才返回真，且短路计算：expr1为假时直接返回false，不会评估expr2  
- `||`运算符只要一边为真时就返回真，且短路计算：expr1为真时直接返回true，不会评估expr2  
- `not`运算符对expr取反，expr为真，则取反后为假，expr为假，则取反后为真  
- `not and or`基本等价于对应的`! && ||`，但文字格式的逻辑运算符优先级非常低，而符号格式的逻辑运算符优先级则较高  
- `//`运算符见下文  

因为符号格式的逻辑运算符优先级很高，所以往往左边和右边都会加上括号，而文字格式的优先级很低，左右两边不需加括号

```perl
if (($n >=60) && ($n <80)){ print "..."; }
if ($n >=60 and $n <80){ print "..."; }
```

`or`运算符往往会用于连接两个【成功执行，否则就】的子句。例如，打开文件，如果打开失败，就报错退出perl程序：

```perl
open LOG '<' "/tmp/a.log" or die "Can't open file!";
```
有时候还会分行缩进：
```perl
open LOG '<' "/tmp/a.log"
    or die "Can't open file!";
```
同样，`and`运算符也常用于连接两个行为：左边为真，就执行右边的操作(例如赋值)。
```perl
$m < $n and $m = $n;   # 将$m和$n之间较大值保存到变量m
```

### 逻辑运算的返回值

在Perl中，还需关注逻辑运算的返回值：返回最后计算的那个表达式的结果值。

对于`expr1 && expr2`，如果expr1计算结果为假，则短路计算，直接返回expr1的计算结果。如果expr1计算结果为真，将会继续计算expr2，于是返回的是expr2的计算结果。

对于`expr1 || expr2`，如果expr1计算结果为真，则段落计算，直接返回expr1的计算结恶果。如果expr1计算结果为假，将会继续计算expr2，于是返回的是expr2的计算结果。

对于`! expr`，如果expr计算结果为真，则返回undef(在不同上下文可转换为数值0或空字符串)，如果expr计算结果为假，则返回代表布尔真的数值1。

作为技巧，可以将两个`!`一起用，如`!!a`，它会【负负得正】：如果原来a代表布尔真值，负负得正后会得到代表布尔真的1，如果原来a代表布尔假值，负负得正后得到代表布尔假的undef(在不同上下文可转换为数值0或空字符串)。也就是说，`!!`在布尔判断效果上不会变化，但会将值转换为undef或1。

```perl
say !!"abc";   # 1
say !!"";      # 空
say ((!!"") + 2);  # 2
```

### 关于||和//

`||`会短路计算，且有返回值。结合这两点，可以为变量做默认赋值。

例如：
```perl
my $name = $myname || "junmajinlong"
```

当变量myname未定义时，将`"junmajinlong"`赋值给变量name，当变量myname已定义时，将myname的值赋值给变量name。因此，这样的赋值方式可以让变量name有默认值。

但是，这样的方式不严谨，因为有两种情况都会将`junmajinlong`作为默认值赋值给变量name：

- 变量myname处于未定义状态  
- 变量myname已定义，但其值为undef、空字符串、数值0等代表布尔假的值  

因此，为了确保在myname处于未定义状态或值为undef时才将`junmajinlong`作为变量name的默认值，Perl v5.10提供了另一种逻辑运算符`//`，它也称为【逻辑定义或】(logical defined-or)：如果左边的值不是undef(包括未定义变量)，则短路运算且返回左边的值，如果左边的值为undef，则计算并返回右边表达式的值。

现在，可以使用`//`为变量赋以默认值：
```perl
use 5.010;
my $name = $myname // "junmajinloing";
```
当开启了warnings时，`//`可以免除使用undef值的警告，但仍然无法避开`use strict`模式下使用未定义变量的编译错误。

```perl
use warnings;
# 无警告，尽管使用了undef
my $name = undef // "junmajinloing";

use strict;
# 报错，使用了未定义变量myname
my $name = $myname // "junmajinloing";
```

