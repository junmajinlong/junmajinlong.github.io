---
title: Perl的IO操作(1)：文件句柄
p: perl/perl_io_handler.md
date: 2019-07-06 17:37:54
tags: Perl
categories: Perl
---

# Perl的IO操作(1)：文件句柄

## 文件句柄

文件句柄用来对应要操作的文件系统中的文件，这么说不太严谨，但比较容易理解。首先为要打开的文件绑定文件句柄(称为打开文件句柄)，然后在后续的操作中都通过文件句柄来操作对应的文件，最后关闭文件句柄。

如不理解文件句柄的概念，可将文件句柄看作Linux中文件描述符的概念（当然，它们是不同的，Perl的文件句柄在层次上对应于Linux中的标准IO流）。例如特殊的STDIN、STDOUT、STDERR就是perl中预定义好的文件句柄，分别表示标准输入、标准输出、标准错误，要将它们对应到Linux上的话，它们是默认的文件描述符fd=0、fd=1和fd=2的字符串描述形式，而这几个文件描述符分别对应文件系统中的/dev/stdin、/dev/stdout和/dev/stderr设备文件。也就是说，Linux上的perl中的文件句柄STDIN、STDOUT和STDERR默认关联的文件是/dev/stdin、/dev/stdout和/dev/stderr。

如果还不理解文件句柄，就把它想象成通道：perl程序中IO操作和磁盘上文件的通道。例如，print语句将数据通过某个通道输出到对应的文件中。

>文件句柄和文件描述符  
>
>>实际上，文件句柄和文件描述符是有区别的。文件描述符是一个数值，代表**操作系统**所使用的**裸数据流**，文件描述符是文件句柄的核心，**文件句柄可以看作是文件描述符的更高一层次的封装，比如提供了和描述符有关数据流的输入、输出的buffer缓冲**。也就是说，文件句柄比文件描述符多一些额外的功能。

我们可以为任意要操作的文件定义一个文件句柄。通常，使用大写字母作为文件句柄的名称。

例如，下面打开一个文件句柄LOG，这个文件句柄对应的文件是/tmp/a.log，操作模式是追加写入(和shell中的追加重定向是相同的意思)。然后向LOG文件句柄中输入一段数据，最后关闭文件句柄。
```
open LOG,">>/tmp/a.log";
print LOG "haha, hello world";
close LOG;
```

一般来说，打开了文件句柄后，在操作完成后要关闭文件句柄以便节省操作系统【打开文件数量限制】的资源。但perl有时候比较智能，会在某些时候自动帮我们关掉文件句柄。而且，当打开一个文件句柄后，再次去打开这个文件句柄时，perl会先关闭这个文件句柄再打开这个文件句柄，这称为【文件句柄的reopen】，它只是隐式地关闭并重新打开，perl并不认为中间涉及了关闭操作(例如reopen时行号不会重置)。

## 打开文件句柄

要打开文件句柄，使用open函数。open函数的功能其实很丰富，如有需要，可去官方手册查看：http://perldoc.perl.org/functions/open.html

打开(open)文件是有目的的：为了读取？为了写入？为了追加写入？这是操作模式。在open文件时，需要指明操作模式。此外，还要给定要关联的文件路径，路径可以是绝对路径，也可以是相对当前perl程序的相对路径。

另外注意，文件句柄需唯一，不能出现重名。

例如：
```
open LOG1,">","/tmp/a.log";   # 以覆盖写入的方式打开文件/tmp/a.log
open LOG2,">>","/tmp/a.log";  # 以追加写入的方式打开文件/tmp/a.log
open LOG3,"<","/tmp/a.log";   # 打开/tmp/a.log文件，以提供输入源
open LOG4,"/tmp/a.log";       # 等价于上面的输入，默认的模式就是输入
```
另一种写法是将模式符号和目标文件放在一起：
```
open LOG1,">/tmp/a.log"; 
open LOG2,">>/tmp/a.log";
open LOG3,"</tmp/a.log"; 
```
中间还可以有空格：
```
open LOG1,"> /tmp/a.log"; 
open LOG2,">> /tmp/a.log";
open LOG3,"< /tmp/a.log"; 
```

可以将目标文件赋值给一个变量，然后在open函数中使用变量名替换。
```
my $tmp_file = "/tmp/a.log";
open LOG1,">","$tmp_file";
```

如果要指明输入、输出的文件编码，则使用上面将【模式和路径分开】的方式。例如：
```
open LOG1,">:encoding(UTF-8)","/tmp/a.log";   # 以UTF-8编码方式写入数据
open LOG1,">>:encoding(UTF-8)","/tmp/a.log";
open LOG1,"<:encoding(UTF-8)","/tmp/a.log";   # 以UTF-8编码方式读入数据
```

需要注意，perl自身是无法打开外部文件的，它需要请求操作系统内核，让操作系统来打开文件。所以，打开文件正确、错误时，操作系统都会有相应的回馈信息。对于perl来说，open函数的返回值就表示正确、错误打开文件。所以，通过以下方式可以判断是否正确打开：
```
my $success = open LOG,">","/tmp/a.log";
if(!success){
    exit 1;
}
```

更好、更常用的方式是使用die：
```
open LOG,">","/tmp/a.log"
    or die "open file wrong: $!";
```
或者使用autodie功能，当捕获到某些错误时，会自动调用die结束程序：
```
use autodie;
open LOG,">","/tmp/a.log";
```
无法打开文件的可能原因有很多，比如读取时文件不存在，比如上级目录不存在，比如无权限等等。操作系统会向perl报告这些错误，使用`$!`可以引用操作系统向perl报告的错误，例如`die "can't open file: $!";`被触发时的消息如下：
```
can't open file: No such file or directory at myperl.plx line 5.
```

上面的【No such file or directory at myperl.plx line 5.】就是`$!`收集和整理后的错误信息。

同理，关闭文件句柄错误也可以捕捉：
```
close DATA
    or die "Couldn't close file properly";
```

## 使用文件句柄：读取文件数据


例如，要从test.log文件中读取所有数据行，一般的流程如下：
```
#!/usr/bin/perl
use 5.010;

open LOG,"<","test.log"
    or die "open file wrong: $!"
while(<LOG>){
    chomp;
    say $_;
}
```

另外，从特殊文件句柄`<STDIN>`、`<>`、`<ARGV>`中读取数据时，由于它们是预定义好的，所以不需要先open。

## 使用文件句柄：写入数据到文件

要向文件中写入数据，可以使用输出语句，如print/say和printf，要写多行的时候，可以使用heredoc的方式。

在使用print/say/printf的时候，在这几个关键字后面接上文件句柄即表示本输出语句写入到此文件句柄中。其实，当它们不指定文件句柄的时候，所采用的就是默认的文件句柄STDOUT。

例如，以追加模式写入一行数据到test.log中。

```
#!/usr/bin/perl

use 5.010;

open LOG,">>","test.log"
    or die "Can't open file: $!";

say LOG "NEW LINE!";
```

再例如，向标准输出、标准错误中输出信息：
```
say STDOUT "NEW LINE!";
say STDERR "NEW LINE!";
```

## 选一个默认的输出文件句柄

注意是选择默认的输出文件句柄，不适用于输入的文件句柄。

默认情况下的默认输出文件句柄就是STDOUT，但是可以使用select关键字自己选一个默认的输出文件句柄。只是需要注意的是，再将内容输出到自选的默认输出文件句柄结束后，应该重新选回STDOUT。

例如，读取某个文件的内容，追加重定向输出到另一个文件中：
```
#!/usr/bin/perl

open LOG,">>","test1.log" or die "Can't open file: $!";

select LOG;
while(<>){
    print "Line $. from $ARGV: $_";
}
select STDOUT;
print "restored default filehandler: STDOUT\n";
```

然后执行该perl程序(程序名：15.plx)，并传递a.log作为命令行参数。作为运行结果，会将a.log中的数据追加到test1.log文件中，并输出一行内容到终端屏幕上。
```
$ perl 15.plx a.log
restored default filehandler: STDOUT
```

选择默认的文件句柄后，上面while循环中的print，等价于`print LOG ...`。

通常，选择默认的文件句柄更常用于设置文件句柄是否要缓冲。例如，输出到下面三个文件句柄(LOG/STDERR/STDOUT)的数据不会缓冲，而是直接输出到文件句柄。
```
select    LOG; $| = 1;  # make unbuffered
select STDERR; $| = 1;  # make unbuffered
select STDOUT; $| = 1;  # make unbuffered
```

其中，控制输出的缓冲变量为`$|`，通常在使用管道、套接字的时候，可能不需要甚至不应该对数据进行缓冲，而是直接暴露给其它进程。

一般来说，在超过一个文件句柄需要关闭缓冲时，不会使用这种`select XXX; $|=1`的方式，而是导入`IO::Handle`模块，然后使用它的`autoflush(1)`函数来实现关闭IO缓冲。
```
use IO::Handle;
FH1->autoflush(1);
FH2->autoflush(2);
FH3->autoflush(3);
```

`autoflush(1)`的功能等价于：
```
select( (select(NEWOUT), $| = 1 )[0] );
```

因为`select()`的返回值是当前标准输出的文件句柄，然后内层的select的结果和`$| = 1`的结果构成一个匿名列表，选择列表的第一个元素即之前标准输出的文件句柄，将其作为外层select的参数，即表示恢复了之前的文件句柄。(如果目前看不懂，请忽略)

>关于IO Buffer
>>分为两种IO缓冲模式：**block buffer、line buffer**。  
>>block buffer是表示先积累一定数据量(比如通常是几K大小)之后再输出。line buffer是表示只有读取到了换行符的时候才输出。`$|=1`或者autoflush(1)都表示从block buffer(默认)切换成无缓冲模式，也即禁用缓冲。  

## 文件句柄变量

除了使用大写字母(一般情况下文件句柄都如此命名，称为裸句柄)的文件句柄，还可以使用变量来命名文件句柄。

例如，使用变量代表的文件句柄：
```
my $rock_fh;
open $rock_fh,"<","/tmp/a.log"
    or die "Can't open file: $!";
```
或者：
```
open my $rock_fh,"<","/tmp/a.log"
    or die "Can't open file: $!";
```

使用变量文件句柄时：
```
while(<$rock_fh>){
    chomp;
    ...
}
```

不过有时候，使用文件句柄变量会产生歧义。例如下面的语句：
```
print $rock_fh;
```

perl并不知道`$rock_fh`是要输出的列表数据还是输出的目标文件句柄。如果是目标文件句柄，这意味着print将`$_`写入到文件句柄`$rock_fh`中。如果是要输出的列表数据，由于`$rock_fh`是一个已定义好的文件句柄，print将输出它的引用(类似于：GLOB(oxABCDEF12))。


这时可以使用大括号包围文件句柄变量。
```
print {$rock_fh};
print {$rock_fh} "hello world";
```

最后需要注意的是，**裸句柄是包变量，在整个文件内都是有效的，而变量方式的文件句柄只在代码块范围内有效，出了自己的作用域范围就失效(自动关闭)**。

## 目录句柄

目录句柄和文件句柄类似，可以打开它，并读取其中的**文件列表**。只不过需要使用：
- opendir替换open函数
- 使用readdir来替换readline，readdir返回一个列表
    - readdir不会递归到子目录中
- 使用closedir来替代close
- 注意每个Unix的目录下都包含两个特殊目录`.`和`..`

```
opendir JAVAHOME,"/usr/local/java"
    or die "Can't open dir handler: $!";
foreach $file (readdir JAVAHOME){
    print "Filename: $file \n";
}
closedir JAVAHOME;
```