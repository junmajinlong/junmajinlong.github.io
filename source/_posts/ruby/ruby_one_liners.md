---
title: Ruby：一行式命令用法总结和技巧
p: ruby/ruby_one_liners.md
date: 2020-11-25 01:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

Ruby一行式和Perl一行式大体类似，但有些用法上它们各有千秋，可以了解了解。

# Ruby一行式常用函数、选项、全局变量

这部分内容可作为参考内容，有需求时再来翻阅。

## ruby一行式常用函数

在Kernel模块中有一些函数在Ruby one-liner中非常实用。它们中大多数都允许直接操作默认变量`$_`：
```
gets
readline
从ARGV文件列表(即ARGF是默认的输入源)中读取一行，并总是自动赋值给$_
gets遇到eof返回nil而不会报错
readline遇到eof时会报错

readlines
使用gets方式自动读取ARGV中所有数据，并保存到一个数组

print
输出到标准输出，当省略参数时，默认输出$_，即print()等价于print($_)
注意，puts()/p()/pp()等没有该行为

sub
gsub
等价于$_=$_.sub和$_=$_.gsub，即直接修改$_
注意，没有sub!()和gsub!()

chomp
chop
等价于$_.chomp
```

例如：**Ruby中，print()省略参数时，表示输出`$_`的内容，等价于`print $_`**。因此，在Ruby一行式中，输出`$_`时，更常使用`print`而非`puts $_`。
```bash
$ printf 'gate\napple\nwhat\n' | ruby -ne 'print'
gate 
apple
what 
```

## ruby一行式常用选项

| 选项        | 选项说明             |
| ----------- | ------------------ |
| `-0[octal]` | 设置`$/`，即指定输入记录分隔符，若未指定参数，则为`\0` |
| `-a`        | 将`$_`根据`-Fpattern`划分成字段，各保存到全局数组`$F`中  |
| `-Cdir`     | 执行ruby命令之前，先进入指定目录    |
| `-e 'cmd'`  | 一行式表达式，可同时指定多个`-e` |
| `-Fpattern` | 设置`$;`，`split()`将根据`$;`分割`$_`为字段并保存到`$F`数组中 |
| `-i[ext]`   | 直接修改ARGV的各个文件，给定ext时则先备份源文件 |
| `-l`        | 读取行时自动去除尾部换行符，输出行时自动加上换行符  |
| `-n`        | 自动读取ARGV的每一行，相当于`while gets(); ... end`  |
| `-p`        | 相当于-n，但总是自动输出`$_`  |
| `-rlib`     | require指定的lib库    |
| `-s`        | 将--后的-x=y注册成全局变量$x=y |

`-0`有两种常用用法：  
- 不指定参数：表示一次性读取整个文件  
- 参数指定为0：表示按段落读取且压缩连续空行  

`-s`选项主要用于从shell注册变量。例如：  
```bash
$ ruby -sne 'p $a' -- -a=3 filename
3
```

> 注意：
>
> 当Ruby一行式命令中使用需要划分字段时(比如使用-a选项)，Ruby处理速度会急剧下降。Ruby总是使用`split`划分字段，而split总是使用正则表达式进行字段的划分，但正则表达式的划分方式性能较低。
>
> awk根据FS变量划分字段，它可以是正则，也可以是精确的字符，如果awk使用正则划分字段，它的速度也会很慢，但如果不是正则表达式划分字段，那么awk的速度会很快很快，远远快于Ruby。(此处所述awk为gawk版本)
>
> 因此在处理大量行数据且需要划分字段时请考虑清楚要不要使用Ruby，如果可以，应测试awk实现相同功能的效率。


## ruby一行式常用全局变量

```
$~
正则匹配的MatchData对象

$&
正则匹配的内容

$`
正则匹配成功时左边的内容

$'
正则匹配成功时右边的内容

$+
正则匹配最后一个分组的内容

$N
正则匹配第N个分组的内容

$/
输入记录分隔符，默认值为换行符，即每次读取一行，一行是一条记录

$\
print()输出时的输出记录分隔符(也是IO#write的输出分隔符)，默认值为nil

$,
输出字段分隔符，print()和Array#join()的字段分隔符，默认值为nil

$;
输入字段分隔符，String#split()的字段分隔符，默认以连续的空白符作为分隔符，且忽略前缀后缀空白

$.
当前行号。ARGV中如果有多个文件时，$.不会重置

$<
ARGF的别名

$>
Kernel#print和Kernel#printf的默认输出流。默认值为$stdout

$_
保存gets()和readline()最近所读取的一行

$0
脚本名称

$*
ARGV的别名

$$
脚本的PID

$FILENAME
ARGF当前正在处理的文件名，等价于ARGF#filename

$stderr
当前的标准错误输出IO流，默认值为STDERR

$stdin
当前的标准输入IO流，默认值为STDIN

$stdout
当前的标准输出IO流，默认值为STDOUT

STDIN
标准输入

STDOUT
标准输出

STDERR
标准错误

ENV
保存了ruby进程运行时的环境变量

ARGF
ruby命令行中要处理的虚拟文件

ARGV
命令行参数数组
```

# ruby一行式的常用技巧

无论是Ruby一行式还是Perl一行式，一行式命令的灵魂都是追求精简、追求技巧、追求骚操作、追求代码高浓缩。掌握各种常用技巧，无疑会在一行式命令上锦上添花。

## ruby一行式常用正则匹配方式

**当Ruby正则表达式处于匹配上下文时，可省略匹配目标，此时表示匹配`$_`**。等价于`~/pat/`和`/pat/ =~ $_`。
```bash
$ printf 'gate\napple\nwhat\n' | ruby -ne 'print if /at/'
$ printf 'gate\napple\nwhat\n' | ruby -ne 'print if ~/at/'
$ printf 'gate\napple\nwhat\n' | ruby -ne 'print if /at/=~$_'
gate
what

# 正则不在匹配上下文时，将直接返回定义的正则对象而不是去匹配$_
$ ruby -ne 'puts /at/'
(?-mix:at)(?-mix:at)(?-mix:at)(?-mix:at)

# 定义并返回正则，然后丢弃
$ printf 'gate\napple\nwhat\n' | ruby -ne '/at/;print $&'

# 定义正则并返回正则，返回值当作true对待
$ printf 'gate\napple\nwhat\n' | ruby -ne '/at/ && print'
gate
apple
what

$ printf 'gate\napple\nwhat\n' | ruby -ne 'print if /at/ && /ga/'
gate
```

如果只是为了确定是否匹配成功，而不需要匹配后的其他信息，可使用`match?()`，它会更高效一点。
```bash
$ ruby -ne 'print if $_.match?(/on\b/)' a.file
```

如果要获取正则匹配的内容，可使用`Str[/reg/]`语法：
```bash
$ echo junmajinlong.com | ruby -ne 'puts $_[/[^.]*/]'
junmajinlong
```

## ruby一行式常用技巧

**在一行式中，常使用更简短的写法来执行Array的join操作**：
```bash
$ ruby -e 'p %w(a b c)*"-"'
"a-b-c"

$ ruby -e 'p %w(a b c)*"::"'
"a::b::c"
```

**有时候也会用上字符串的tr()方法，在一行式中，更多的是用`$_.tr!()`**。
```bash
$ # rot13
$ echo 'Uryyb Jbeyq' | ruby -pe '$_.tr!("a-zA-Z", "n-za-mN-ZA-M")'
Hello World

# 除0-9和换行符外的其他字符都替换为-，第一个参数开头使用 ^ 表示补集替换
$ echo 'foo:123:baz' | ruby -pe '$_.tr!("^0-9\n", "-")'
----123----

# 第二个参数设置为空字符串，表示删除匹配的字符
$ echo 'foo:123:baz' | ruby -pe '$_.tr!("^0-9\n", "")'
123
```

ruby一行式经常会使用`xxx if yyy`格式的if判断，**如果xxx超出一条语句，常使用括号包围多条语句**。
```ruby
# 及时使用exit是很有必要的，可以避免花时间处理不必要的数据
$ seq 1 10 | ruby -ne'(print;exit) if /3/'
3
```

**ruby一行式中，`$*`是ARGV的别名，ARGV保存ruby脚本程序(不是ruby程序自身)的所有选项和参数。当开始处理某参数时，该参数会先从ARGV中移除。当处理多个文件时，可以通过ARGV.size()来判断还有几个文件未处理，这间接地表示当前ruby正在处理第几个文件**。
```bash
$ ruby -e 'p ARGV' a.txt b.txt c.txt
["a.txt", "b.txt", "c.txt"]

$ ruby -ne 'p ARGV;exit' a.file b.file c.file
["b.file", "c.file"]

# 第一行时手动添加一个文件到ARGV中等待被处理
$ ruby -ne 'ARGV.push("b.file") if $.==1;print' a.file

# 处理第一个文件时，将该文件内容保存到hash结构中，以便和第二个文件对比
# 下面的if $*.size==1表示处理a.file
# print if h[$_]则是在处理b.file
$ ruby -ne 'BEGIN{h={}};
            (h[$_]=1;next) if $*.size==1;
            print if h[$_]' a.file b.file
```

**ruby一行式中，`$<`是ARGF的别名，它代表的是所有待处理文件组成的虚拟IO流。`$.`表示已经处理的记录号(行号)，处理下一个文件时，`$.`行号不会重置为1。`ARGF.eof`可判断当前正在处理的文件是否遇到了eof**(注意是每个文件的eof，而不是所有文件的eof)。

```bash
# 输出最后一行
$ seq 3 8 | ruby -ne '$<.eof && print'
8 

# 输出每个文件的最后一行
$ ruby -ne 'print if $<.eof' <(seq 3 5) <(seq 7 9)
5
9

# 每个文件重置行号：$.=0即可
# 重置行号的另一种方式是调用ARGF.close
$ ruby -ne '(puts $.;$.=0) if $<.eof' <(seq 1 5) <(seq 3 9)
$ ruby -ne '(puts $.;$<.close) if $<.eof' <(seq 1 5) <(seq 3 9)

# 每个文件输出第一行后立即退出
$ ruby -pe 'ARGF.close' a.file b.file
$ ruby -ne 'ARGF.close;print ARGF.filename,": ",$_' a.file b.file
```

**一行式中经常会使用到flip-flop，ruby支持两种flip-flop，写法和意义都和Range类似**。

```ruby
# 匹配第1到第3行
$ seq 3 10 | ruby -ne 'print if $.==1..$.==3'

# 匹配第1到第3行，但不包括第3行
$ seq 3 10 | ruby -ne 'print if $.==1...$.==3'

# 匹配第5行到最后一行
$ seq 3 10 | ruby -ne 'print if $.==5..'
```

如果flip-flop在条件语句中，可有如下简写方式(对比sed的写法)：
- `if 3..5`等价于`if $.==3..$.==5`  
- `if /a/../b/`等价于`if ~/a/..~/b/`等价于`if /a/=~$_../b/=~$_`  
- `if 3../a/`等价于`if $.==3..~/a/`  
- `if !(/a/..$<.eof)`  

```bash
# 匹配第1到第3行
$ seq 3 10 | ruby -ne 'print if 1..3'
$ seq 3 10 | ruby -ne 'print if(1..3)'
```

**ruby中不会自动转换数据类型，例如nil不会转换为数值0执行加减法，也不允许对不存在的变量调用方法(报错：NilClass NoMethodError)。但在一行式中经常需要对变量执行一些操作，有几种比较简洁的处理方式**：
```bash
# `||`初始化转换：h[$_]=(h[$_]||0)+1
# 或者引用类型时 `(h[$_]||=[]).push("hello")`
$ ruby -lne 'BEGIN{h={}};h[$_]=(h[$_]||0)+1;END{pp h}' a.file

# 三元运算符转换：h[$_] = h[$_]?h[$_]+1:0
$ ruby -lne 'BEGIN{h={}};(h[$_]=h[$_]?h[$_]+1:0);END{pp h}' a.file

# 对于hash来说，可指定key不存在时的默认值
# 但这样设置的默认值如果是引用类型(比如数组)，则所有使用默认值的操作都会得到或影响那一个默认值
# 所以，这样设置默认值的方式，只建议设置不可变类型
$ ruby -lne 'BEGIN{h=Hash.new(0)};h[$_]+=1;END{pp h}' a.file

# 安全调用运算符`var&.method`，避免NilClass NoMethodError
```

## ruby一行式字段处理常用技巧

处理输入字段分三类方式：  
- 指定`-a`选项(内部调用`split()`)，按字符或按正则将输入记录划分为各字段保存到`$F`数组  
- 手动使用`split()`划分字段  
- 使用`String#scan()`将正则匹配到的内容保存到数组，而不是像split一样将匹配到的内容作为分隔符  
- 使用`unpack()`，按字符数量划分字段  

使用`-a`选项时，ruby会将每一条输入记录使用split()划分为各个字段保存到`$F`数组中，split()划分字段的依据由`-F`选项或`$;`变量指定。

`split(pattern=nil [,limit])`划分字段时，遵循如下规则：  
- pattern是单个字符时，该字符作为字段分隔符  
   - 但如果是单个空格时，则以连续空白为分隔符，且忽略前缀后缀空白  
- pattern是正则时，以该正则匹配到的内容作为字段分隔符  
- pattern为nil(默认)时，采用`$;`的值作为分隔符。`$;`默认值为nil，此时等价于单个空格的分割方式`split(" ")`  
- 如果给定了limit参数，其为数值，表示最多划分为这么多个字段  

因此，对于一行式来说，如果不指定`-F`或`$;`，将以`split(" ")`方式划分字段，即以一个或多个连续的空格作为分隔符，且忽略前缀和后缀空白。

当指定`-F`时，其值将当作正则表达式作为split()的第一个参数。而通过`$;`，则可以设置字符或正则表达式的分隔方式。

```bash
$ echo -e '  a    b   c  ' | ruby -ane 'p $F * "-"'
"a-b-c"

$ echo 'a b c d e' | ruby -ane 'p $F[1]'
"b"
$ echo 'a b c d e' | ruby -ane 'p $F[-1]'
"e"
$ echo 'a b c d e' | ruby -ane 'p $F[1..3]'
["b", "c", "d"]
$ echo 'a b c d e' | ruby -ane 'p $F[-3..-1]'
["c", "d", "e"]
$ echo 'a b c d e' | ruby -ane 'p $F[1,2]'
["b", "c"]

# 注意，-F" "将等价于split(/ /)而不是split(" ")
$ echo 'a b c d e   ' | ruby -F' ' -ane 'p $F[-1]'
"\n"
$ echo 'a b c d e   ' | ruby -F' ' -lane 'p $F[-1]'
"e"
```

使用`print()`输出字段时：  
- 往往结合`-l`选项来处理行尾换行符  
- 可通过`$,`设置`print()`输出字段分隔符  
- `Array#join()`也会使用`$,`作为默认的字段分隔符  
- 未设置`$,`时，其默认值为空字符串，因此默认情况下，print输出的各字段是紧密相连的  

```bash
$ echo 'a b c d e' | ruby -alne 'BEGIN{$,="-"};print $F[1],$F[2]'
b-c

$ echo 'a b c d e' | ruby -ane 'p $F[1..3].join("-")'
"b-c-d"

$ echo 'a b c d e' | ruby -ane 'p $F[1..3]*"-"'
"b-c-d"
```

使用`scan()`可以将匹配到的内容保存到数组：
```bash
$ echo "hello12world23HELLO" | ruby -lne 'print $_.scan(/\d+/)[1]'
23

$ echo 'eagle,"fox,42",bee,frog' | ruby -lne 'puts $_.scan(/"[^"]*"|[^,]+/)[1]'
"fox,42"
```

使用`unpack()`可以按字符数量来处理字段。
```bash
$ cat a.file
ID  name    gender  age  email          phone
1   Bob     male    28   abc@qq.com     18023394012
2   Alice   female  24   def@gmail.com  18084925203
3   Tony    male    21   aaa@163.com    17048792503
4   Kevin   male    21   bbb@189.com    17023929033
5   Alex    male    18                  18185904230
6   Andy    female  22   ddd@139.com    18923902352
7   Jerry   female  25   exdsa@189.com  18785234906
8   Peter   male    20   bax@qq.com     17729348758
9   Steven  female  23   bc@sohu.com    15947893212
10  Bruce   female  27   bcbd@139.com   13942943905

# 第一个字段2个字符，然后跳过两个字符: a2x2
# 第二、三字段均占用6字符，然后跳过两个字符: a6x2
# 第四字段占用3字符，然后跳过2字符: a3x2
# 第五字段占用13字符，然后跳过2字符: a13x2
# 第六字段占用剩下所有字符: a*
$ ruby -ne 'puts $_.unpack("a2x2a6x2a6x2a3x2a13x2a*") * ","' a.file
ID,name  ,gender,age,email        ,phone
1 ,Bob   ,male  ,28 ,abc@qq.com   ,18023394012
2 ,Alice ,female,24 ,def@gmail.com,18084925203
3 ,Tony  ,male  ,21 ,aaa@163.com  ,17048792503
4 ,Kevin ,male  ,21 ,bbb@189.com  ,17023929033
5 ,Alex  ,male  ,18 ,             ,18185904230
6 ,Andy  ,female,22 ,ddd@139.com  ,18923902352
7 ,Jerry ,female,25 ,exdsa@189.com,18785234906
8 ,Peter ,male  ,20 ,bax@qq.com   ,17729348758
9 ,Steven,female,23 ,bc@sohu.com  ,15947893212
10,Bruce ,female,27 ,bcbd@139.com ,13942943905
```

## 输入和输出记录的处理技巧

- 设置`$/`可修改ruby读取输入记录时的分隔符，默认值为`\n`表示每次读取一行  
- 设置`$\`可修改print输出时的记录分隔符，默认值为空  

默认情况下，`$/`的输入记录分隔符会保留在`$_`中，可以使用`chomp()`或`-l`选项，它们默认都会消除输入记录分隔符。
```bash
$ printf 'abc ABC\ndef ABCdef\nDEF ABC' | ruby -lne 'BEGIN{$/="ABC"};print "#{$.})#{$_}"'
1)abc 
2)
def
3)def
DEF

$ printf 'abc ABC\ndef ABCdef\nDEF ABC' | ruby -ne 'BEGIN{$/="ABC"};print "#{$.})#{$_}"'
1)abc ABC2)
def ABC3)def
DEF ABC
```

`$/`的值只能是字符串，可以是空串或单字符或多字符，不能是正则表达式。特别地，当`$/=""`时，表示按段落读取。

按段落读取表示一次读取一段，至少有连续一个空行分割的，称为段落。按段落读取时，会忽略第一个段落的前缀空行，且自动压缩多个空行。

还可以设置`-0`选项来设置单个字符作为输入记录分隔符，该选项的值只能是8进制数值表示方式。特别地：  
- 当`-0`不接参数时，表示以`\0`作为输入记录分隔符，这很可能代表一次性读取整个文件(因为文件中出现`\0`的几率较小)  
- 当`-00`时，表示按段落读取，等价于`$/=""`  

```bash
$ cat c.file


aaaaa


BBBBBBBB

CCCCCCCCCC
$ ruby -l00pe '' c.file
aaaaa
BBBBBBBB
CCCCCCCCCC
```

注意`-l`和`-0`的顺序会影响print()的输出结果：`-l`在后面，会覆盖`-0`的效果。

设置`$\`可修改print()的输出分隔符。默认输出记录分隔符为空。经常地，需要按条件设置输出记录分隔符，这时可使用三元运算符`$\ = cond ? val1 : val2`。

```bash
# 等价于：ruby -pe 'sub(/\n/, "-") if $. % 3 != 0'
$ seq 6 | ruby -lpe '$\ = $. % 3 != 0 ? "-" : "\n"'
1-2-3
4-5-6
```

