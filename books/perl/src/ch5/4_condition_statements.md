## 条件判断：if、unless和三元运算

if和unless都是条件判断语句，它们都支持else子句和任意数量的elsif子句。语法如下；
```perl
if(COND){  # 或者 unless(COND)
  command
}

if(COND1){  # 或者 unless(COND1)
  command1
}elsif(COND2){
  command2
}elsif(COND3){
  command3
} else {
  commandN
}

if(COND){  # 或者 unless(COND)
  command1
}else{
  command2
}
```

注意，`COND`可以是任意一个表示布尔值的值或表达式，它是一个标量上下文。Perl中任何一个需要进行条件判断的地方都是标量上下文。

例如：

```perl
# 如果默认变量$_中有换行符，则去除换行符后使用say输出
if(chomp){  # chomp操作字符串时返回1或0
  say $_;
}

# 如果数组元素数量大于等于5，则输出前5个元素
if(@arr >= 5){
  say "@arr[0..4]";
}
```

unless和if判断方式相反，对于if，条件为真时执行紧跟着的语句块，对于unless，条件为假时执行紧跟着的语句块。所以，unless相当于if的else部分，或者说`unless(cond)`相当于`if(!cond)`。

```perl
# 除非数组元素数量小于5，否则就输出前5个元素
unless(@arr < 5){
  say "@arr[0..4]";
}
```

多数时候，只会用到unless的单分支，不会用到unless的else或elsif子句，因为这样的逻辑可以改写成等价但更易懂的if语句。

### 三元运算符

Perl也支持三元运算符：如果expr返回真，则执行并返回`when_true`的结果，否则执行并返回`when_false`的结果。
```
expr ? when_true : when_false
```

例如，求平均值，如果`$n=0`，则输出`------`。

```perl
$avg = $n ? $sum/$n : "------";
```

注意上面示例中的优先级问题，赋值运算符`=`的优先级只比`not and or`和逗号运算符的优先级高，因此赋值操作几乎总是在最后才执行。

三元运算符是对`if(expr){when_true}else{when_false}`的简写。例如上面示例等价于下面的if逻辑：

```perl
if($n){
    $avg = $sum / $n;
}else{
    $avg = "------";
}
```
三目运算符可以写出更复杂的分支：
```perl
# 如果$score小于60分，则mark为c
# 如果大于等于60小于80，则mark为b
# 如果大于等于80小于90，则mark为c
# 其余情况，mark为d
$mark = ($score < 60) ? "c" : 
        ($score < 80) ? "b" : 
        ($score < 90) ? "a" : 
        "a++";          # 默认值
say $mark;
```