## 控制循环流程

### 标签

Perl允许为循环结构打标签：

```perl
LABEL while (EXPR) BLOCK
LABEL until (EXPR) BLOCK
LABEL for (EXPR; EXPR; EXPR) BLOCK
LABEL for VAR (LIST) BLOCK
LABEL foreach (EXPR; EXPR; EXPR) BLOCK
LABEL foreach VAR (LIST) BLOCK
# 纯语句块也可以打标签，它是执行一次的循环结构
LABEL BLOCK  
```

为循环结构打标签后，last、next等可以控制循环流程的关键字就可以指定要控制哪个层次的循环结构。

### last、next、redo

这三个关键字都用于控制循环的执行流程：

- last相当于其它语言里的break关键字，用于退出循环(包括for/foreach/while/until/纯语句块)  
- next相当于其它语言里的continue关键字，用于跳入下一轮循环  
- redo用于跳转到当前循环层次的顶端，使得本轮循环从头开始再次执行  

下面是一些示例。

last：

```perl
# 当变量i等于5时退出循环
# 该循环将输出1 2 3 4
for my $i (1..10){
  if($i==5){
    last;
  }
  say $i;
}
```

next：

```perl
# 当变量i等于5时进入下一轮循环
# 该循环将输出1 2 3 4 6 7 8 9 10
for my $i (1..10){
  $i == 5 and next;
  say $i;
}
```

redo：

```perl
# 当变量i等于5时，再次执行第5轮循环
# 该循环将输出2 3 4 6 6 7 8 9 10 11
for my $i (1..10){
  $i++;
  $i == 5 and redo;
  say $i;
}
```

如果熟悉sed命令，应该会知道sed里也有类似redo的功能。但其他语言中没有类似redo的功能。可通过下面示例再感受一下redo的效果。

```perl
use open ':std', ':encoding(UTF-8)';
foreach (1..5){
  say "请输入redo";
  chomp($_ = <>);  # 从终端读取输入
  /redo/i and redo;
  say "你没有输入redo";
}
```

默认情况下，last、next和redo控制的都是当前层次的循环结构，可以为它们指定标签来决定要控制哪个外层循环。下面是使用标签控制外层循环的一个示例：

```perl
OUTER:
while (1) {
    print "start outer\n";
    while (1) {
        print "start inner\n";
        last OUTER;
        print "end inner\n";
    }
    print "end outer\n";
}
print "done\n";
```

将输出：

```
start outer
start inner
done
```

### 附加循环代码：continue

Perl中还有一个continue关键字(`perldoc -f continue`)，它可以是一个函数，也可以跟一个代码块。
```
continue              # continue函数
continue BLOCK        # continue代码块
```

如果指定了BLOCK，continue可用于循环结构之后。
```
while (EXPR) BLOCK continue BLOCK
until (EXPR) BLOCK continue BLOCK
for VAR (LIST) BLOCK continue BLOCK
foreach VAR (LIST) BLOCK continue BLOCK
BLOCK continue BLOCK
```

continue表示在循环结构的主循环代码块之后附加了一段额外的代码块，每轮循环中，在执行完主循环代码块后都会执行continue部分的代码块，执行完后才进入下一轮循环。

当循环结构给定continue代码块后，redo、last和next关键字的控制规则为：

- redo、last直接控制整个循环主体  
- next总是跳转到continue代码块开头  

continue语句块中也可以使用redo、last和next关键字，控制规则不变。

以while + continue为例，当在continue中使用next、last、redo时：

```perl
while(){
    # <- redo会跳到这里
    CODE
} continue {
    # <- next总是跳到这里
    CODE
}
# <- last跳到这里
```

实际上，如果没有在循环语句中指定continue语句块，逻辑上等价于给了一个空的continue代码块，这时next可以跳转到空代码而进入下一轮循环。

例如：
```perl
my $a=3;
while($a<8){
  if($a<5){
    say '$a in main if block: ',$a;
    next;
  }
} continue {
  say '$a in continue block: ',$a;
  $a++;
}
```

输出结果：
```
$a in main if block: 3
$a in continue block: 3
$a in main if block: 4
$a in continue block: 4
$a in continue block: 5
$a in continue block: 6
$a in continue block: 7
```

### 通过循环实现一个简单的Perl Shell

Perl自身没有提供交互式的Shell，有时候要测试Perl代码不是很方便。但实现一个简单的交互式的Perl Shell也比较简单：

```perl
while(print ">> ") {  # 无限循环，输出提示符
  # 读取终端输入，去除行尾换行符
  chomp ($_ = <>);
  # 当输入q并回车时，退出循环
  /^q$/ && last;
  # 执行输入的perl代码，并输出执行代码的返回值
  say "=> ", (eval $_) // "undef";
}
```

虽然有很多不足，但做简单的临时测试工具，已经足够了。执行效果如图：

```bash
$ perl perl_shell.pl 
>> 33 + 44
=> 77
>> $a = 33
=> 33
>> print $a;
33=> 1
>> say $a;
33
=> 1
>> my $b;
=> undef
>> if($a){say "$a"}
33
=> 1
>> q  # 退出
```

