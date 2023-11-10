---
title: Perl和OS交互(一)：system、exec和反引号
p: perl/perl_system_exec.md
date: 2019-07-07 17:38:39
tags: Perl
categories: Perl
---

# Perl和OS交互(一)：system、exec和反引号

## 调用操作系统命令：system函数

system函数可以直接让perl调用操作系统中的命令并执行。

### system入门示例

例如：
```
#!/usr/bin/perl

system 'date +"%F %T"';
system 'echo hello world';
system 'echo',"hello","world";
```
执行结果：
```
2018-06-21 18:32:50
hello world
hello world
```

注意system的参数可以被单个引号包围，也可以用多个引号分隔成多个参数，如果分隔开，system会将它们用空格的方式连接起来。

另外，上面使用了单引号、双引号，都能正确执行，但注意，双引号会解析perl中的特殊符号。例如：

```
$myname="Malongshuai";
system "echo $myname";   # 输出：Malongshuai
system 'echo $USER';     # 输出当前登录的用户：root
```

可见，双引号中的变量`$myname`被perl解析了，而单引号中的变量`$USER`不被perl解析，perl将其交给bash，由shell负责解析，所以会输出当前用户名。

在system中，还可以使用shell的重定向、管道等功能。
```
$myname="Malongshuai";
system "echo $myname >/tmp/a.txt";
print "==============================\n";
system "cat <1.plx";
print "==============================\n";
system 'find . -type f -name "*.pl" -print0 | xargs -0 -i ls -l {}';
system 'sleep 30 &';
```

### 深入system

system有两种语法：
```
system LIST
system PROGRAM LIST
```

这里忽略第二种，因为它是一种以欺骗的防止执行命令的：LIST中的第一个参数作为命令，但欺骗自己说自己执行的是PROGRAM命令。

下面将详细讨论第一种语法。

#### 基础知识

在讨论之前，先解释一下bash命令行执行命令时的引号解析问题。例如：
```
awk -F ":" 'NR<=3{username=$1;print "username:",username}' /etc/passwd
find /root -type f -name "*.log"
```

shell命令行中执行命令时，包含两部分：一个是程序名，一个是程序的参数部分。在真正执行之前，shell的词法分析行为会解析程序名称、参数部分。但有些时候命令行中会使用一些shell的特殊符号来实现shell的特殊功能。例如shell的星号通配符`*`、管道功能`|`、重定向功能`> < >> << <<<`、命令替换功能`$()`等。但有些程序自身，其用法规则中可能也会使用一些特殊符号(如`find -name "*.log"`的星号)，这会和shell的特殊符号冲突。由于shell的解析行为在命令执行之前，为了保留特殊符号给程序自身来解释，需要使用引号来保护这些特殊符号以避免被shell解析。

正如上面awk中的`":"`和`'{}'`以及find中的`"*.log"`，它们都使用引号包围特殊符号，使得这些符号"逃过"shell的解析过程，从而让程序自身解析。

更通俗一点，如果不是执行命令要依赖于shell环境的存在，如果能直接在最纯粹的环境中执行命令，那么特殊符号是无需加引号保护的。例如，awk如果能脱离shell单独执行，下面的第一条命令才是正确的，第二条命令却是错误的。
```
awk -F : NR<=3{username=$1;print "username:",username} /etc/passwd
awk -F ":" 'NR<=3{username=$1;print "username:",username}' /etc/passwd
```

#### system参数细节

`system LIST`中的system要求的是列表上下文参数LIST，就像print函数一样。所以，当LIST是一个标量字符串，它其实也是一个列表，只不过是只包含一个元素的列表。

例如：
```
system 'find /perlapp -type f -name "*.pl"';   # 是一个标量字符串构成的LIST

system "ls","-lh","/root";    # 包含多元素的列表参数

@cmd_arg=qw(-lh /root);
system "ls",@cmd_arg;       # 包含多元素的列表参数
```

对于`system LIST`语法，perl在执行LIST中的命令之前，会先检查LIST：  
1. 当system的参数是一个只有单元素的列表(即上面第一个例子)，它将检查这个参数整体中是否有需要shell解析的特殊元字符(如shell中的通配符`* ？ []`，shell中的重定向`< > >> <<< <<`，shell中的管道`|`，shell的后台任务符号`&`，命令替换`$()`等等)：  
    - 如果有这些需要shell解析的特殊元字符，则调用`/bin/sh -c STRING`的方式来执行LIST，其中LIST就是STRING部分  
    - 如果没有需要shell解析的特殊元字符，则perl将其分割成一个一个单词，并传递给`execvp`系统函数(man execvp)来执行，它的效率比unix的system()更高  
2. 当system的参数是一个包含多元素的列表：  
    - 它将认为列表中的第一个元素是待执行的命令，并直接执行它，而不会先调用shell,再通过shell来解析并执行它。  
    - 所以，使用多元素的列表参数时，将失去shell中重定向、管道、命令替换等等功能  
    - 但如果第一个元素作为命令spawn失败(和语法、参数等无关，而是权限或其它系统层面的失败)，将降级回使用shell来执行  

注：`bash -c STRING`的c选项会从STRING中读取命令并执行。

几个示例：
```
@arg1=qw(-lh /root);
system "ls",@arg1;          # 1.可正确执行

system "ls -lh /root/*.log"; # 2.可正确执行

@arg2=qw(-lh /root/*.log);
system "ls",@arg2;           # 3.将执行失败

system "ls -lh","/root";     # 4.执行失败，更准确的是spawn过程就失败
system "ls","-lh /root";     # 5.执行失败
system "ls","-l -h","/root"; # 6.执行失败
```

上面第二个system能执行成功，而第三个system会执行失败，是因为：  
- 第二个system的参数是一个单元素的列表，而且有需要解析的通配星号字符，所以它等价于`/bin/sh -c ls -lh /root/*.log`命令  
- 第三个system的参数是多个元素构成的列表，所以它会直接spawn一个ls进程，由于不在shell环境中执行，ls程序又不认识星号字符，所以执行失败  

第四个system也执行失败，因为不止一个参数，于是取第一个参数作为命令来spawn新的进程，但这第一个参数是`ls -lh`整体，而不是ls，这等价于`"ls -lh" /root`，所以spawn失败，找不到这个命令。

第5个system执行失败，因为"-lh /root"作为列表的第二个元素，它是一个整体。所以它等价于`ls "-lh /root"`，这显然是错误的。

第6个system执行失败，原因同上。

所以可以稍微总结下，**如果使用多个参数的system，每个原本在unix shell命令行中需要空格分开的选项和参数，都需要单独作为列表的独立元素**。

正如：
```
system "ls","-lh","/root";

@args=qw(-lh /root);
system "ls",@args;
```

更复杂一点的示例：
```
@cmd_arg1=qw(/perlapp -type f -name *.pl);
system "/usr/bin/find",@cmd_arg1;        # 1.正确

@cmd_arg2=qw(/perlapp -type f -name "*.pl");   # 加上了双引号
system "/usr/bin/find",@cmd_arg2;        # 2.错误

$prog="/usr/bin/awk";
@arg3=("-F",":",'NR<=3{username=$1;print "username: ",username}','/etc/passwd');
system $prog,@arg3;      # 3.正确
```

上面第二个system中，是多参数的system，不会调用shell来解析，而`*.pl`使用了引号包围，但对于find来说，引号不可识别的字符，它会将其当作要查找文件名的一部分，所以执行失败。之所以在shell命令中的find要加上引号，是为了防止`*`被shell解析。

第三个system中，没有使用`qw()`的方式生成列表，因为awk的表达式部分存在空格，使用qw生成列表的方式无法保留空格，所以这里采用最原始的生成列表的形式。当然，也可以实现split来生成：
```
@arg3=split /%/,q(-F%:%NR<=3{username=$1;print "username: ",username}%/etc/passwd);
```

### 使用单个参数还是多参数？

关于使用单个参数的system还是使用多参数的system。

如果对shell解析熟悉，使用单个参数比较好，能比较直接地使用shell相关的功能(重定向、管道等)。但使用单个参数，引号引用和转义引用方面毕竟比较复杂，容易出错，可能需要多次调试。

多个参数也有好处，不用担心太多引号问题，但却失去了使用shell功能的能力。如果想要在多参数的system中使用管道、重定向等特殊符号带来的shell功能，可以将`'/bin/sh','-c'`作为system的前两个参数，使得system强制调用shell来执行命令。

`/bin/sh -c STRING`执行命令的方式是shell从STRING中读取命令来执行。所以，为了保证完整性，STRING部分建议全都包含在一个引号中。例如：

```
shell> bash -c 'find . -type f -name "*.pl" | xargs ls -l'
```

回到system的调用`/bin/sh -c`的用法，例如：

```
$arg1=q(find . -type f -name "*.pl" -print0);    # 1
$arg2=q( | xargs -0 -i ls -l {});                # 2
system '/bin/sh','-c',"$arg1 $arg2";             # 3
```

上面3行，每行都有关键点：  
- 第一行：  
    - 不能使用数组、列表，而是标量的字符串  
    - 因为要给shell解析，所以`*.pl`还是要加上引号包围  
- 第二行：  
    - 同样，不能使用数组、列表，而是标量字符串  
    - 即使是特殊的管道符号(或其它符号)，也可以直接放在标量字符串中  
- 第三行：  
    - 前两个参数是`/bin/sh`和`-c`  
    - 第三个参数必须是字符串STRING，强烈建议使用引号包围，保证参数的完整性  
    - 如果不加引号包围STRING，而是将arg1和arg2作为参数列表的两个元素，将割裂两者，导致只执行到`$arg1`中的命令，甚至有时候会因为`$arg1`不完整或有多余字符而报错

看上去规则很多，而且书写必须十分规范，失之毫厘，结果将差之千里。如非必须，还不如直接写成单个参数的system。例如，上面的3行等价于：
```
system '/bin/sh','-c','find . -type f -name "*.pl -print0 | xargs -0 -i ls -l {}"';
system 'find . -type f -name "*.pl -print0 | xargs -0 -i ls -l {}"';
```

### 捕获system的错误状态

system执行命令时的返回值为`$?`，它和bash的`$?`不太一致。当最后一个管道关闭时、反引号执行命令、wait()或waitpid()成功执行时或system()，都会返回`$?`。在Perl中，`$?`包含两部分共16字节，低8位是信号信息，高8位才是所执行的命令的状态码。也就是说，perl中的`$?`的高8位才对应bash中的`$?`。

因此，要获取退出状态码，需要使用`$?>>8`。
```
#!/usr/bin/perl

system '(exit 4)';
print $?>>8,"\n";    # 输出4
```

如果，想要直接在执行的命令上判断命令是否正确执行，然后决定是否die。可以在system的前面加上一个`!`取反。这是因为在shell中，非0的状态码表示命令错误执行，0状态码才表示执行正确。这和perl的布尔值正好相反，所以加上感叹号取反：

```
!system '(exit 4)' or die "command return error num: ",$?>>8;
```

需要注意，这里不能使用`$!`，在perl中有多种不同的错误捕获变量，`$!`捕获的是perl在发起系统调用层面的错误，而system执行的命令的错误发生在命令执行时。对于system函数来说，perl只要成功执行system，不管里面的命令是否执行成功，perl发起的系统调用都已经结束了。


关于如何获取信号信息，参见官方手册。或者：
```
The “low” octet combines several things. The highest bit notes if a core dump happened.The hexadecimal and binary representations (recall them from Chapter 2) can help mask out the parts you don’t want:

my $low_octet = $return_value & 0xFF; # mask out high octet
my $dumped_core = $low_octet & 0b1_0000000; # 128
my $signal_number = $low_octet & 0b0111_1111; # 0x7f, or 127
```

### system的内部细节

在Perl中，除了system，还有exec、fork、pipe、IPC等进程操作方式，它们的细节，都可man system、man exec、man fork等等来获取。在后文会一一解释，此处先解释system执行的细节。

在执行到system时，system会直接拷贝一份当前perl进程(称为子进程)，然后自己进入睡眠态，并使用waitpid()等待子进程执行完毕。

unix系统中的system()用来调用一个shell解释器来执行命令，用来启动一个新的程序，是`fork+execl("/bin/sh -c COMMAND")+waitpid()`的结合，因为多一层shell的调用，效率相比于fork+exec来说较低，且需要waitpid()的等待，无法控制子进程也无法并发。

perl的system()和unix的system()不太一样，多了一层判断来决定是使用`fork+execl("/bin/sh -c COMMAND")+waitpid()`还是直接使用`fork+execvp(COMMAND)+waitpid()`。

因为是直接拷贝的，所以子进程初始时和perl父进程是完全一致的。所以，标准输入(STDIN)、标准输出(STDOUT)、标准错误输出(STDERR)都是和父进程共享的。
```
system 'read -p "enter your name: " name;echo "your name is: " $name';
```

在system中的命令执行之前，perl首先会解析system的参数列表，关于解析的方式，在前文已经详细解释过了。如果命令是直接执行的，则命令所在进程就是perl进程的子进程。如果命令需要通过通过调用`/bin/sh -c`来执行，则shell进程是子进程，真正执行的命令则是孙进程(grandchild)或者是下一代。

例如，在参数中放入shell的for循环，因为这是bash内置属性，它会直接在当前bash进程中完成。
```
system 'for i in {1..10};do echo $i;done';
```

这些内容比较复杂，可参见：[bash内置命令的特殊性，后台任务的"本质"](https://www.cnblogs.com/f-ck-need-u/p/9183819.html)

当命令执行完毕后，将回到perl进程，perl进程会执行wait()，然后结束system。

## 调用操作系统命令：exec

exec和system除了一种行为之外，其它用法和system完全一致。exec和system的区别之处在于：

- system会创建子进程，然后自己进入睡眠，去等待子进程执行完毕，最后执行wait()  
- exec不会创建子进程，而是在当前Perl进程自身去执行命令，相当于用命令去覆盖当前进程，所以没有睡眠  
- 当exec执行的命令结束后，将直接结束当前perl进程，没有wait()行为

由于exec执行完命令后，立即退出当前perl进程，所以命令执行的正确与否，无法被捕获。但如果exec启动待执行命令过程就出错了，这属于perl的系统调用过程出错，可以使用`$!`捕获。

```
exec 'date';
die "date couldn't run: $!";
```

一般来说，很少直接使用exec，而是fork+exec同时使用。关于fork，见[perl和操作系统交互：fork](/perl/perl_fork)。


## 调用操作系统命令：反引号和qx()

perl中的system()和exec()执行命令时，都是直接执行命令，并将执行结果输出到某个地方(比如屏幕)。但是反引号(`` `COMMAND` ``)可以将执行的结果插入到某个地方或者进行赋值，而不是直接输出。就像shell中的反引号一样。

例如，将操作系统中date命令的执行结果赋值给一个变量。

```
#!/usr/bin/perl

my $date=`date +"%F %T"`;
print $date;
```

由于反引号是将命令的输出结果捕获起来并插入到某个地方或赋值，如果反引号单独成一个语句，也即是在空上下文(void)中，它的结果会丢弃。一般来说，这是多此一举或者是浪费的行为，除非是要通过执行命令临时做出某些设置：
```
`date +"%F %T"`;   # 命令的结果将直接丢弃
```

qx()和反引号执行命令是一样的，只不过写法不同，使得某些特殊符号的处理变得更容易，就像shell中也有一个`$()`的方式替换`` ` ` ``。特别地，由于perl反引号是以双引号的方式解释反引号内部的内容，如果反引号中间有perl可以解释的特殊符号，就会被perl先解释，再传递给shell去执行。如果使用qx并使用单引号作为定界符(即`qx'COMMAND'`)，perl将使用单引号的方式去解释COMMAND，使得perl不再解释一些特殊符号。

例如下面的例子中，在shell环境中导出了一个环境变量name，值为`Gaoxiaofang`，而perl程序内部正好也定义了一个变量`$name`，这时使用反引号`` `COMMAND` ``和`qx'COMMAND'`就不再一样。

以下是shell中执行的命令：
```
export name="Gaoxiaofang"
```
以下是perl程序中的内容
```
#!/usr/bin/perl

$name="Malongshuai";
my $new_name1=`echo $name`;
print $new_name1;             # 输出Malongshuai

my $new_name2=qx'echo $name';
print $new_name2;             # 输出Gaoxiaofang
```

但需要注意的是，shell反引号做的命令替换，由于常用来插入到某个表达式中间，所以shell在反引号执行完毕后会自动移除换行符，除非使用双引号包围反引号。而perl则有所不同，perl总会保证所执行即所得，perl的反引号会保留每一个换行符。

一般来说，在perl中使用反引号的时候，都会使用chomp去除最后一个换行符。

```
chomp(my $date=`date +"%F %T"`);
print $date;
```

如果在列表上下文中使用反引号，则反引号中命令的每一行输出都会保存为列表的元素。
```
my @new_name=qx'who';
print "$new_name[0]";
```
同样地，可以将反引号放进foreach，因为foreach的迭代目标正是一个列表：
```
foreach (`cat /etc/passwd`){
    print $_ if m%bin/bash%;
}
```
