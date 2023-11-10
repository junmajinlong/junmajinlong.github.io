---
title: Perl读取输入<STDIN>、<>和chomp函数
p: perl/perl_stdin_chomp.md
date: 2019-07-06 17:37:43
tags: Perl
categories: Perl
---

# Perl读取输入<STDIN>、<>和chomp函数

## Perl读取标准输入\<STDIN\>

`<STDIN>`表示从标准输入中读取内容，如果没有，则等待输入。`<STDIN>`读取到的结果中，如果没有意外，都会自带换行符。

例如，test.plx文件内容：
```
#!/usr/bin/perl
#
$line=<STDIN>;
if($line eq "\n"){
        print "blank line\n";
} else {
        print "not blank: $line"
}
```
注意上面的else语句中，`$line`后面没有加换行符，因为`<STDIN>`自带换行符。

下面的命令，将等待输入和回车。如果直接回车，则if条件为真。
```
perl test.plx
```

下面是和bash shell交互。
```
echo "hello" | perl test.plx
echo -e "haha\nheihei" | perl test.plx
```

注意上面第二条语句中，heihei会被忽略，因为上面的操作是标量上下文(上下文的概念，以后会解释)，**在发现换行符的时候，结束输入的读取**，所以看到haha后面的`\n`就结束了。

因为`<STDIN>`读取的是标准输入，所以如果要通过它读取文件内容，需要使用shell的重定向功能。例如，读取a.txt文件的内容到`<STDIN>`:
```
perl test.plx <a.txt
```

另外，`<STDIN>`在标量上下文中返回的是某一行，可以使用while来遍历多行，但在遍历时要书写准确：
```
# 错误遍历
$line = <STDIN>;
while(defined($line)){
    print $line;
}

# 正确遍历
while(defined($line = <STDIN>)){
    print $line;
}
```
第一种写法会无限循环输出读取的第一行(`\n`前面的行)，因为`$line`被赋值为该第一行，且不再改变，由于`defined()`返回真，会使得while无限循环。

第二种写法每次读取一行，每读取一行做下读取的位置标记方便下次读取(也就是说`<STDIN>`是可迭代对象)，直到读取完最后一行返回undef使得defined返回false，结束循环。

由于`<STDIN>`在读取到最后一行后会返回undef，所以可以简写上面的第二种方式：
```
# (建议写法)
while(<STDIN>){
    print $_;
}

# 或(不建议写法)
foreach(<STDIN>){
    print $_;
}
```

只是需要注意上面的两种简写方式，每次读取的行并不是直接赋值给`$_`，而是在每次迭代过程中赋值的。换句话说，上面的`<STDIN>`和`$_`没有直接关系，不像`$line=<STDIN>`这种赋值，直接在读取的时候就赋值给$line，也正因为不是直接赋值给变量，才能循环读取下去。

(本段涉及到上下文和数组的概念，暂时还没介绍，如不理解，可先略过)上面的while和foreach有巨大的差别，while后面是标量上下文，foreach后面是列表上下文。这意味着while是每次从文件中读取一行，打上位置标记以便下次读取，然后进入循环体。而foreach因为操作目标是列表，它会一次性将文件中所有行读取到内存，然后当作列表进行遍历，所以性能非常差，例如400M的文件，也许需要1G的内存，因为perl会预估先分配足够多的内存以便后续不再因为分配内存的事而中断进程。上面的过程，使用下面的形式描述，就很容易理解了：
```
$line = <STDIN>;    # 一次读一行，性能好
@lines = <STDIN>;   # 一次读所有，性能差
```

`<STDIN>`会带有换行符，通常都会加上`chomp()`操作符去掉换行符，关于chomp，见下文。

## 读取文件输入\<>

perl中使用两个尖括号符号表示读取来自文件的输入，例如从命令行中传递文件作为输入源。这个符号被称为"钻石操作符"。

例如，test.plx程序内容如下：
```
#!/usr/bin/perl

while(defined($line = <>)){
    print $_;
}
```
或者简写的：
```
#!/usr/bin/perl

while(<>){
    print $_;
}
```
然后在perl程序的命令行中指定输入文件：
```
$ ./test.plx test.log
```
或者使用`@ARGV`数组指定输入文件：
```
#!/usr/bin/perl

@ARGV=qw(test.log);
```
如果想要读取标准输入的数据，则在命令行中使用短横线`-`。
```
$ echo -e "haha\nheihei" | ./test.plx -
```

一般来说，while循环中使用`<STDIN>`或`<>`读取输入后(也包括open关键字打开文件再读取行的情况)，第一行就是去除行尾的换行符，所以大多数都采用如下通用格式：
```
while(<>){
    chomp;
    COMMANDS;
}

while(<STDIN>){
    chomp;
    COMMANDS;
}
```

上面的chomp没有参数，所以采用默认变量`$_`，也就是迭代中的每一行。

关于`<>`的机制，请继续阅读下文的`<<>>`。

## chomp()函数

这个函数用于去掉**字符串**或者读取到的标准输入的**一个** **换行符**。注意关键字：字符串、一个、换行符。

注意，**这个函数是在原处修改字符串**。但这个函数有自己的返回值：  
- 如果能去掉换行符，则返回移除的字符数，也就是数值1。这是个没什么用的返回值，因为我们都已经知道了；  
- 如果没有换行符，则返回数值0；  
- 如果结尾有两个换行符，则只去掉一个。  

比较下面两个程序，它们会从标准输入中读取：
```
$line=<STDIN>;
print $line;

$line=<STDIN>;
chomp($line);
print $line;
```

或者下面的例子，它直接操作一个已有的字符串：
```
$foo="hello world!\n"
chomp($foo);
print $foo;
```

字符串赋值操作和chomp()函数可以结合在一起。
```
chomp($line=<STDIN>);
chomp($line="hello world!\n");
print $foo;
```

但注意，chomp()是有返回值的。下面返回的是数值1。
```
print chomp($line="hello world!\n");
```


## \<\<\>\>

实际上，除了`<>`还有`<<>>`。它们之间有区别，目前为止还没有介绍文件句柄和ARGV变量，所以能看懂则看，看不懂则过。

`<>`会隐式打开来自`@ARGV`数组中的参数文件，打开方式是两参数格式的open，类似于`open "FH","$ARGV[N]"`。正常情况下这没什么问题，但可能会成致命的危险。例如Perl脚本文件a.pl内容如下:

```
#!/usr/bin/perl
while(<>){print}
```

如果执行该脚本的方式为：
```
$ perl a.pl "rm -rfv * |"
```

其中`rm -rfv * |`被当作一个参数收集到`@ARGV`数组中，因为`<>`以两参数模式的方式隐式打开文件，这等价于：
```
open FH,"rm -rfv *|";
```

这表示先打开一个管道，再执行`rm -rfv *`命令，并将该命令的输出通过管道传递供`<>`读取。

再看下面的示例，它们是等价的。
```
$ perl a.pl "ls /tmp |"
$ ls /tmp | perl a.pl
```

这在写一行式perl程序的时候比较方便：
```
perl -pe '' "ls /tmp |"
ls /tmp | perl -pe ''
```

所以，以`<>`的方式打开`@ARGV`中的文件时，因为无法保证这个数组中的参数一定是文件，所以可能会出现危险操作。

而`<<>>`则能避免这个问题，它隐式地以三参数的open打开`@ARGV`中的文件。类似于：
```
open FH,"<","$ARGV[N]";
```

它限定了第二个参数是输入重定向操作，所以它保证了`@ARGV`中的参数必须是文件，否则就会报错。
```
perl -e 'while(<<>>){print}' /etc/passwd
perl -e 'while(<<>>){print}' "ls /tmp |"   # 报错
```

这时想要从管道读取数据，只能将管道放在perl命令的前面。
```
ls /tmp | perl -e 'while(<<>>){print}'
```
