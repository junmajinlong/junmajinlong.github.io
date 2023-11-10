## 循环控制语句

Perl中支持while循环、until循环、for循环、foreach循环，还支持for迭代遍历、foreach迭代遍历。在循环或迭代时，循环体内可以通过last、next、redo以及continue来控制循环流程。

此外，Perl还支持标签功能、goto语句、纯语句块以及do语句。

### while循环和until循环

while循环和until循环是类似的：  

- 对于while循环，只要条件判断为布尔真，就执行循环体，直到条件判断为假  
- 对于until循环，只要条件判断为布尔假，就执行循环体，直到条件判断为真  

语法如下：

```perl
while(CONDITION){
    commands;
}

until(CONDITION){
    commands;
}
```

注意条件判断处于标量上下文。

例如，循环10次：

```perl
my $i = 0;
while($i<10){
  say $i;
  $i++;
}
```

while也常结合each一起使用来遍历hash数据。例如：

```perl
while(my($k,$v)=each %p){
  say "k: $k, v: $v";
}
```

### for循环和foreach循环

Perl中的for和foreach支持两种语法：类C的for循环语法和迭代时的for迭代语法。for循环和foreach循环等价，for迭代和foreach迭代等价。

以for循环为例。例如：

```perl
for($i=1;$i<=10;$i++){
    print $i,"\n";
}
print $i,"\n";   # 输出11
```
需要注意的是，上面的`$i`默认是全局变量，循环结束后还有效，在开启了strict模式后会报错。可以将其声明为局部变量：
```perl
for (my $i = 1;$i<=10;$i++ ){
    print $i,"\n";
}
```
for循环不仅仅只支持数值递增、递减的循环方式，还支持其它类型的循环，只要能进行条件判断即可。见下面的例子。

for循环和foreach完整的语法为：

```perl
for (expr1; expr2; expr3) {
  ...
}
foreach (expr1; expr2; expr3) {
  ...
}
```

循环的执行流程为：首先执行expr1，这部分是for的初始操作，然后执行expr2进行条件判断，如果expr2为真，则执行一次循环体，执行完循环体后执行一次expr3，然后再执行expr2进行条件判断，为真则执行循环体，然后expr3，然后expr2，如此一直循环，直到expr2为布尔假时退出循环。

for括号中的3个表达式都可以省略，但两个分号不能省略：  
- 如果省略第三个表达式，则表示一直判断，直到退出循环或者无限循环  
- 如果省略第二个表达式，则表示不判断，因此会无限循环  
- 如果省略第一个表达式，则表示不执行初始操作(比如初始赋值)  

例如，下面分别省略第三个表达式和省略所有表达式：
```perl
# 每次删除字符串开头一个字符，直到删除完所有字符
for(my $str="junmajinlong";$str =~ s/(.)//;){
  say $str;
}

# 无限循环
for(;;){
  say "never stop";
}
```

对于无限循环，下面这种while方式更方便：
```perl
while(1){
  command;
}
```

### for迭代和foreach迭代

for迭代语法和foreach迭代语法是一致的。

对于for、foreach迭代遍历语法，其语法为：

```perl
for my $i (LIST) {}
for (LIST){}
foreach my $i (LIST) {}
foreach (LIST){}
```

下面以for迭代语法为例。

for会从列表中不断迭代每一个元素，每次取得一个元素并【赋值】给控制变量`$i`，如果没有指定控制变量，则【赋值】给默认变量`$_`。直到取完列表所有元素，迭代完成。

```perl
# 循环5次
for (1..5){
  say $_;
}

# 遍历数组
my @arr1 = qw(a b c d e f);
for my $i (@arr1){
  say $i;
}

# 带索引的数组遍历
my @arr2 = qw(a b c d e f);
for(0..$#arr2){
  say "index: $_, value: $arr2[$_]";
}
```

需注意，for迭代时，控制变量`$i`指向每个被迭代的元素。因此，修改`$i`也会影响原始数据。例如：

```perl
my @arr = qw(1 2 3 4 5);
for (@arr){
  say $_;
  $_++;
}
say "@arr";   # 2 3 4 5 6
```

另外，迭代过程中改变列表长度，也会影响迭代过程。例如：

```perl
my @arr = qw(1 2 3 4 5);
for (@arr){
  say $_;        # 输出1 3 5
  shift @arr;
}
say "@arr";   # 4 5
```

### 纯语句块

纯语句块即单独一个大括号：

```perl
{...}
```

纯语句块实际上是一个只循环一次的循环结构。

纯语句块常用来创建一个局部作用域，在该语句块内声明的局部变量，退出语句块后失效。

```perl
my $a = 1;
{
  say $a;      # 1
  my $a = 33;  # 将掩盖外部变量a
  say $a;      # 33
}  # 退出语句块，语句块内的变量a失效
say $a;        # 1
```
