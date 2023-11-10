---
title: Perl的流程控制语句
p: perl/perl_flow_control.md
date: 2019-07-06 17:37:39
tags: Perl
categories: Perl
---

# Perl的流程控制语句

## 布尔值判断

- 如果是数字，0表示假，其它所有数字都是真。
- 如果是字符串，空字符串('')为假，其它所有字符串为真(有例外，见下一条)。
- 如果是字符串'0'，perl是当作数值0来处理的，所以这是唯一的非空但为假的字符串。
- 如果既不是数字，也不是字符串，那么先转换为数字或字符串再做判断(也就是"undef"表示假，其它所有引用表示真)。  
- "!"表示取反。

perl有个技巧，将两个"!"一起用，相当于"负负得正"，所以原来是真的仍然是真的，原来是假的仍然是假的。但实际上，**perl会将它们转换值"1"和"undef"**。
```
$still_true = !!"fred";   # $still_true的值是1
$still_false1 = !!"0";     # $still_false1的值为空(undef)
$still_false2 = !!"";      # $still_false2的值为空(undef)

print "$still_true"."\n";
print "$still_false1"."\n";
print "$still_false2"."\n";
```

## 条件判断：if和unless


它们都是条件判断语句，都支持else、elsif子句。
```
if(CONDITION){  # 或者 unless(CONDITION)
    command
}

if(CONDITION1){  # 或者 unless(CONDITION1)
    command1
}elsif(CONDITION2){
    command2
}elsif(CONDITION3){
    command3
}
    ...
    else{
    commandN
}

if(CONDITION){  # 或者 unless(CONDITION)
    command1
}else{
    command2
}
```

其中CONDITION可以是任意一个标量值。布尔值的判断很简单，方式和bash shell有点类似，但有点相反。

unless和if判断方式相反，if的condition为真则执行后面的代码，否则执行else或者退出if结构。unless则是condition为假时才执行后面的代码，否则执行else或退出unless结构。所以，unless相当于if的else部分，或者相当于`if (!condition)`。

一般来说，不会用到unless的else语句，因为它完全可以改编成if语句。之所以有时候会使用unless而不是if的否定形式，是因为有时候的条件语句用if来写确实不方便。

## 三目运算符`?:`

perl也支持三目运算符：如果expression返回真，则整个表达式返回`if_true`，否则返回`if_false`
```
expression ? if_true : if_false
```

例如，求平均值，如果$n=0，则输出"------"。
```
$avg = $n ? $sum/$n : "------";
```
它等价于：
```
if($n){
    $avg = $sum / $n;
}else{
    $avg = "------";
}
```
三目运算符可以写出更复杂的分支：
```
#!/usr/bin/perl
#
use 5.010;

$score = $ARGV[0];
$mark = ($score < 60) ? "a" : 
        ($score < 80) ? "b" : 
        ($score < 90) ? "c" : 
        "d";          # 默认值
say $mark;
```
执行结果：
```
$ perl test.plx
a
$ perl test.plx 33
a
$ perl test.plx 63
b
$ perl test.plx 83
c
$ perl test.plx 93
d
```

## 逻辑运算符：and(&&)、or(||)、//、not(!)

```
expr1 < and or && || // > expr2
<not !>expr
```

- `&&`运算符只有两边为真时才返回真，且短路计算：expr1为假时直接返回false，不会评估expr2
- `||`运算符只要一边为真时就返回真，且短路计算：expr1为真时直接返回true，不会评估expr2
- `and`和`or`基本等价于对应的`&&`和`||`，但文字格式的逻辑运算符优先级非常低
- `not`和`!`求反，同样文字格式的`not`的优先级很低
- 因为符号格式的逻辑运算符优先级很高，所以往往左边和右边都会加上括号，而文字格式的优先级很低，左右两边不需加括号
- `//`运算符见下文

```
if (($n >=60) && ($n <80)){
    print "...";
}

if ($n >=60 and $n <80){
    print "...";
}
```

`or`运算符往往用于连接两个"成功执行，否则就"的子句。例如，打开文件，如果打开失败，就结束该perl程序：
```
open LOG '<' "/tmp/a.log" or die "Can't open file!";
```
更常见的，还会分行缩进：
```
open LOG '<' "/tmp/a.log"
    or die "Can't open file!";
```
同样，`and`运算符也常用于连接两个行为：左边为真，就执行右边的操作(例如赋值)。
```
$m < $n and $m = $n;   # 以$m的姿态取出$m和$n之间较大值
```
以下是3个语句是等价语句：
```
if ($m<$n){$m=$n}
$m=$n if $m<$n;
$m=($m < $n) ? $n : $m;
```

### 关于perl的短路计算和`//`

perl的短路计算非常特别，它返回的是最后运算的表达式的值：
- 如果这个返回值对应的布尔值为真，则整个短路计算自然为真
- 如果这个返回值对应的布尔值为假，则整个短路计算自然为假

这个返回值有时候很有用，往往通过逻辑或的操作来设置默认值，所以，这个返回值既保证短路计算的结果不改变，又能得到返回值。

例如
```
my $name = $myname || "malongshuai"
```

当`$myname`变量存在时，它返回真(0是例外，见下面)，并将`$myname`赋值给`$name`；当`$myname`变量不存在时，它返回假，于是评估"malongshuai"，因为是个字符串，所以返回真，于是将`malongshuai`赋值给`$name`。这样一来，`$myname`变量就有了默认值。

但是，如果`$myname`变量存在，但值为0(字符串的或数值的)，由于它也返回假，导致`$name`被赋以"malongshuai"。

这种行为显然并非我们所需要的。于是，改用下面这种先判断，再赋值的行为：
```
my $name = defined($myname) ? $myname : "malongshuai";
```
但是这样的写法比较复杂，perl 5.10版本提供了更方便的"逻辑定义或"(logical defined-or)操作符`//`：当发现左边的值是已经定义过的，就直接进行短路计算，而不管该左边的值评估后是真是假。
```
use 5.010;
my $name = $myname // "malongshuai";
```
这个操作符应对开启了`use warnings`功能的perl程序很有用，免得warnings发出烦人的警告。

## while循环和until循环

```
while(CONDITION){
    commands;
}

until(CONDITION){
    commands;
}
```

until和其它某些语言的until循环有所不同，perl的until循环，内部的commands主体可能一次也不会执行，因为Perl会先进行条件判断，当条件为假时就执行，如果第一次判断就为真，则直接退出until。

## for循环

Perl中的for循环采取C语言的for风格，例如：
```
for($i=1;$i<=10;$i++){
    print $i,"\n";
}
print $i,"\n";   # 输出11
```
需要注意的是，上面的`$i`默认是全局变量，循环结束后还有效。可以使用my关键字将其声明为局部变量：
```
for (my $i = 1;$i<=10;$i++ ){
    print $i,"\n";
}
```
for循环不仅仅只支持数值递增、递减的循环方式，还支持其它类型的循环，只要能进行判断即可。见下面的例子。

for关键字后面括号中的3个表达式都可以省略，但两个分号不能省略：
- 如果省略第三个表达式，则表示一直判断，直到退出循环或者无限循环
- 如果省略第二个表达式，则表示不判断，所以会无限循环
- 如果省略第一个表达式，则表示不做初始赋值

例如，下面分别省略第三个表达式和省略所有表达式：
```
for(my $str="malongshuai";$str =~ s/(.)//;){
    print $str,"\n";
}

for(;;){
    print "never stop";
}
```

对于无限循环，Perl中更好更优化的方式是使用：
```
while(1){
    command;
}
```

Perl中的for也支持成员测试性的遍历，就像shell中的`for i in ...`的操作一样，它期待一个列表上下文，表示遍历整个列表。如果省略控制变量，表示使用`$_`。例如：
```
my @arr = qw(Shell Python Perl PHP);
for $i (@arr){ print "$i\n" }
for (@arr) {print "$_\n"}
```

像for遍历列表元素的操作，可以使用foreach来替代，大多数迭代列表的时候它们可以互换。

## foreach循环

(因为还没介绍数组和hash，所以这里能看懂就看，不能看懂等到介绍数组的时候再看里面的foreach示例)

foreach更适合用于遍历，所有foreach都能直接修改关键字为for而转换成for循环。当写成for格式的时候，perl通过判断括号中的分号来决定这是foreach循环还是for的普通循环。但for能实现的循环功能，foreach不一定能实现，因为for中有初始变量，有条件判断，而foreach则是简单版的for循环。

先解释下foreach的用法：

例如，迭代从1到10的列表：
```
foreach $i (1..10){
    print $i,"\n";
}
```

其中`$i`称为控制变量，每迭代一次都会从迭代列表中取出一个元素赋值给控制变量。可以省略控制变量，这时将采用默认的变量`$_`：
```
foreach (1..10){
    print $_,"\n";
}
```

foreach可以改写为for：
```
for(1..10){
    print $_,"\n";
}

for ($i=1;$i<=10;$i++){
    print $i,"\n";
}

@arr=qw(malongshuai gaoxiaofang xiaofang longshuai wugui fairy);
for (@arr){
    print $_,"\n";
}
```

关于for循环和foreach循环，如果在遍历过程中修改了元素的值，它会直接修改原始值。换句话说，迭代时赋值给控制变量的元素的引用，而不是赋值元素再赋值给控制变量。
```
@arr=qw(perl python shell);
foreach $subject (@arr) {
    $subject .= "aaa";
}
print @arr;         # 输出perlaaapythonaaashellaaa
```

**当foreach/for遍历结束后，控制变量将复原为foreach/for遍历前的值**(例如未定义的是undef)。
```
$subject="php";
foreach $subject (qw(perl python shell)){
}
print $subject;     # 输出"php"
```

## each遍历

```
each HASH
each ARRAY
```

each用来遍历hash或数组，每次迭代的过程中，都获取hash的key和value，数组的index(数值，从0开始)和元素值。

each放在列表上下文，会返回key/value或index/element，放在标量上下文则只返回key或index。

遍历hash：
```
#!/usr/bin/perl -w
use strict;

my %hash = (
    name1 => "longshuai",
    name2 => "wugui",
    name3 => "xiaofang",
    name4 => "woniu",
);

while(my($key,$value) = each %hash){
    print "$key => $value\n";
}
```

输出结果：
```
name4 => woniu
name3 => xiaofang
name2 => wugui
name1 => longshuai
```

遍历数组：
```
#!/usr/bin/perl -w
use strict;

my @arr = qw(Perl Shell Python PHP Ruby Rust);

while(my($key,$value) = each @arr){
    print "$key => $value\n";
}
```
输出结果：
```
0 => Perl
1 => Shell
2 => Python
3 => PHP
4 => Ruby
5 => Rust
```

each放在标量上下文：
```
#!/usr/bin/perl -w
use strict;

my %hash = (
    name1 => "longshuai",
    name2 => "wugui",
    name3 => "xiaofang",
    name4 => "woniu",
);
my @arr = qw(Perl Shell Python PHP Ruby Rust);

while(my($key) = each %hash){
    print "$key\n";
}

while(my($key) = each @arr){
    print "$key\n";
}
```
输出结果：
```
name2
name4
name3
name1
0
1
2
3
4
5
```

## 表达式修饰符(改写流程控制语句)

perl支持单条表达式后面加流程控制符。如下：
```
command OPERATOR CONDITION;
```
例如：
```
print "true.\n" if $m > $n;
print "true.\n" unless $m > $n;
print "true.\n" while $m > $n;
print "true.\n" until $m > $n;
print "$_" foreach @arr;
```
很多时候会分行并缩进控制符：
```
print "true.\n"        # 注意没有分号结尾
    if $m > $n;
```

改写的方式几个注意点：

- 控制符左边只能用一个命令。除非使用do语句块，参见[do语句块](https://www.cnblogs.com/f-ck-need-u/p/9695536.html)  
- foreach的时候，不能自定义控制变量，只能使用默认的`$_`  
- while或until循环的时候，因为要退出循环，只能将退出循环的条件放进前面的命令中  
    - 例如：`print "abc",($n += 2) while $n < 10;`  
    - `print "abc",($n += 2) until $n > 10;`  
- 改写的方式不能满足需求时，可以使用普通的流程结构  

## 执行一次的语句块

使用大括号包围一段语句，这些语句就属于这个语句块，这个语句块其实是一个循环块结构，只不过它只循环一次。语句块也有自己的范围，例如可以将变量定义为局部变量。
```
{
    print "Enter a Num","\n";
    chomp(my $n = <STDIN>);
    $res = sqrt $n;
    print "$res","\n";
}
print $res,"\n";
```

## 循环控制：last、next、redo、LABEL(标签)

- last相当于其它语言里的break关键字，用于退出当前循环块(for/foreach/while/until/执行一次的语句块都属于循环块)，注意是只退出当前层次的循环，不会退出外层循环
- next相当于其它语言里的continue关键字，用于跳入下一次迭代。同样只作用于当前层次的循环
- redo用于跳转到当前循环层次的顶端，所以本次迭代中曾执行过的语句可能会再次执行
- 标签用于为循环块打上标记，以便那些循环块控制关键字(last/next/redo)可以指定操作的循环层次

```
use 5.010;
use strict;

foreach (1..10){
        say "startline...: $_";
        say "enter a word: last, next, redo?";
        chomp(my $choice = <STDIN>);
        last if $choice =~ /last/i;
        next if $choice =~ /next/i;
        redo if $choice =~ /redo/i;
        say "endline...: $_";
}
say "outside loop...";
```

以下是打标签的示例(标签建议采用大写)：
```
/usr/bin/perl
use 5.010;
use strict;

LINE: while(<>){
    foreach(split){
        last LINE if /error/i;
        say "$_";
    }
}
```
上面的标签循环中，首先读取一行输入，然后进入foreach遍历，因为split没有参数，所以使用默认参数`$_`，这个`$_`所属范围是while循环，split以空格作为分隔符分割这一行，同时foreach也没有控制变量，所以使用默认的控制变量`$_`，这个`$_`所属范围是foreach循环。当foreach的`$_`能匹配字符串"error"则直接退出while循环，而不仅仅是自己的foreach循环。这里if语句后采用的匹配目标是属于foreach的默认变量`$_`。

例如，这个perl程序读取a.txt文件，其中a.txt文件的内容如下:
```
$ cat a.txt
hello world
hello world Error
hello world Error heihei
```
执行这个perl程序：
```
$ perl -w test.plx a.txt
hello
world
hello
world
```
可见，只输出了a.txt中第二行Error前的4个单词。


## 附加循环代码：continue

perl中还有一个continue关键字(http://perldoc.perl.org/functions/continue.html)，它可以是一个函数，也可以跟一个代码块。
```
continue              # continue函数
continue BLOCK        # continue代码块
```

如果指定了BLOCK，continue可用于while和foreach之后，表示附加在循环结构上的代码块。
```
while(){
    code
}continue{
    attached code
}

foreach () {
    code
} continue {
    attached code
}
```

每次循环中都会执行此代码块，执行完后进入下一循环。

在continue代码块内部，也可以使用redo、last和next控制关键字。所以，这几个流程控制关键字更细致一点的作用是：redo、last直接控制循环主体，而next是控制continue代码块。所以：
```
while(){
    # redo jump to here
    CODE
} continue {
    # next jump to here
    CODE
    # next loop
}
# last jump to here
```

实际上，while和foreach在没有给定continue的时候，逻辑上等价于给了一个空的代码块，这时next可以跳转到空代码而进入下一轮循环。

例如：
```
#!/usr/bin/env perl
use strict;
use warnings;

$a=3;
while($a<10){
    if($a<6){
        print '$a in main if block: ',$a,"\n";
        next;
    }
} continue {
    print '$a in continue block: ',$a,"\n";
    $a++;
}
```

输出结果：
```
$a in main if block: 3
$a in continue block: 3
$a in main if block: 4
$a in continue block: 4
$a in main if block: 5
$a in continue block: 5
$a in continue block: 6
$a in continue block: 7
$a in continue block: 8
$a in continue block: 9
```
