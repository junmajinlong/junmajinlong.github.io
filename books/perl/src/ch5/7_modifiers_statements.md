## 流程控制语句修饰符和do语句块

Perl支持单条语句后面加流程控制符。
```perl
command Operator Cond;
```
这样的结构称为流程控制表达式。其中：

- command部分是要执行的语句或表达式  
- Operator部分支持的操作符有if、unless、while、until和foreach，它们称为流程控制语句修饰符  
- Cond部分是条件判断  

整个结果的逻辑是：如果条件判断通过，则根据操作符决定是否要执行command部分。

例如：

```perl
print "true.\n" if     $m > $n;
print "true.\n" unless $m > $n;
print "true.\n" while  $m > $n;
print "true.\n" until  $m > $n;
print "$_"      foreach @arr;
```
这种流程控制表达式结构有几个注意点：  

- 控制符左边只允许一个命令(语句或表达式)，除非使用do语句块，参见下文介绍的do语句块  

- foreach的时候，不能自定义控制变量，只能使用默认的`$_`  

- while或until循环的时候，因为要退出循环，只能将退出循环的条件放进前面的命令中。如： 

    ```perl
    print "abc",($n += 2) while $n < 10;
    print "abc",($n += 2) until $n > 10;
    ```

### do语句块

do语句块结构如下：
```
do {...}
```

do语句块像是匿名函数一样，给定一个语句块，直接执行。且和函数一样，do语句块有返回值，它的返回值是最后一个被执行语句的返回值。

例如，将使用if-elsif-else结构进行赋值的行为改写成do。以下是if-elsif-else结构：
```perl
my $name;
if($gender eq "male"){
  $name="Junmajinlong";
} elsif ($gender eq "female"){
  $name="Gaoxiaofang";
} else {
  $name="RenYao";
}
```

改写成do结构：
```perl
my $name=do{
  if($gender eq "male"){"Junmajinlong"}
  elsif($gender eq "female") {"Gaoxiaofang"}
  else {"RenYao"}
};     # 注意结尾的分号
```

使用流程控制表达式结构的时候，控制符左边只能写一个语句。例如下面的if，左边有了print后，就不能再有其它语句。
```perl
print "..." if(...);
```

使用do结构，可以将多个语句包围，然后执行：
```perl
my $a=3;
do {
  say "statement1";
  say "statement2";
} if $a > 2;
```

但当do语句块结合while和until操作符使用的时候，效果有所改变。这时候Perl将它们特殊对待为其他语言中的`do...while`和`do...until`结构，即do语句块先执行一次，然后才开始进行条件判断。

```perl
do {
  ...
} while cond;

do {
  ...
} until cond;
```

最后，需要区分do语句块和纯语句块：  

- 它们都只执行一次  
- 它们都有自己的代码块作用域  
- do语句块相当于匿名函数，有返回值，它不是循环结构，语句块中不能使用last、redo、next  
- 纯语句块没有返回值(因此不能赋值给变量)，它是只执行一次的循环结构，语句块中可以使用last、redo、next  