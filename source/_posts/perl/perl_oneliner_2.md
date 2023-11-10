---
title: Perl一行式：处理行号和单词数
p: perl/perl_oneliner_2.md
date: 2019-07-08 17:39:50
tags: Perl
categories: Perl
---

# Perl一行式：处理行号和单词数

**perl一行式程序系列文章**：[Perl一行式](/perl/index#blogperloneline)

-----------------------------

## 所有行的行号

```
$ perl -pe '$_ = "$. $_"' file.log
$ perl -ne 'print "$. $n"' file.log
```

这里涉及了一个特殊变量`$.`。

这个特殊变量代表的是当前处理行的行号。对于Perl的一行式来说，通过`<>`隐式打开的文件句柄默认不会关闭，所以如果参数中有多个文件，进入下一个文件时行号不会重置。

例如：
```
$ cat a.txt
aaa
bbb
$ cat b.txt
ccc
ddd

# 行号不重置
$ perl -pe '$_ = "$. $_"' a.txt b.txt
1 aaa
2 bbb
3 ccc
4 ddd
```

如果想要每个文件的行号都独立计算。可以使用下面这种方式进行判断：遇到文件尾部，显式关闭文件。
```
$ perl -e '
    while(<>){
        print "$. $_"
    }continue{
        close ARGV if eof
    }' a.txt b.txt
1 aaa
2 bbb
1 ccc
2 ddd
```

## 非空行的行号

`$.`是Perl自带的文件句柄上的行号计数器，读取的每一行都会计数。所以如果想要统计文件中的某些行的行号，使用自带的`$.`是不可行的，只能自己实现行号计数器。

如下：
```
$ perl -pe '$_ = ++$x." $_" if /\S/' file.log
```

这里的逻辑是，只要行中有非空白字符，就自增一个变量的值。自增后的值和字符串进行串联，并赋值给`$_`被-p输出。


## 非空行行号并删除空白行

因为要删除某些行不输出，所以不能使用-p选项，它会将所有行都输出，除非使用`s///`来删除整行。可以考虑使用-n选项。

```
$ perl -ne 'print ++$x." $_" if /\S/' file.log
```

## 输出匹配行的行号

例如输出文件中匹配"nologin"单词的行号，其它行照常输出。

```
$ perl -pe '$_ = "$. $_" if /nologin/' file.log
root   x 0     0 root   /root     /bin/bash
2 daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
3 bin    x 2     2 bin    /bin      /usr/sbin/nologin
4 sys    x 3     3 sys    /dev      /usr/sbin/nologin
sync   x 4 65534 sync   /bin      /bin/sync
```

如果想要单独计数被匹配行的行号，可以自己写计数器。
```
$ perl -pe '$_ = ++$num." $_" if /nologin/' file.log
root   x 0     0 root   /root     /bin/bash
1 daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
2 bin    x 2     2 bin    /bin      /usr/sbin/nologin
3 sys    x 3     3 sys    /dev      /usr/sbin/nologin
sync   x 4 65534 sync   /bin      /bin/sync
```

如果需要格式化输出，使得没有匹配的行也和带有行号的行对齐。可以进行多分支的赋值：
```
$ perl -pe '
    $_ = do {
        if(/nologin/){
            ++$num." $_"
        }else{
            "  $_"
        }}' file.log

  root   x 0     0 root   /root     /bin/bash
1 daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
2 bin    x 2     2 bin    /bin      /usr/sbin/nologin
3 sys    x 3     3 sys    /dev      /usr/sbin/nologin
  sync   x 4 65534 sync   /bin      /bin/sync
```

或者3目逻辑运算：
```
$ perl -pe '$_ = /nologin/ ? ++$num." $_" : "  $_"' file.log
```

更规范的格式化可以使用printf来对齐，因为这里我使用-p选项，使用使用sprintf格式化字符串保存到`$_`变量上。
```
$ perl -pe '
    $_ = do {
        if(/nologin/){
            sprintf("%-3s %s", ++$num, $_);
        }else{
            sprintf("%-3s %s","", $_);
        }}' file.log

    root   x 0     0 root   /root     /bin/bash
1   daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
2   bin    x 2     2 bin    /bin      /usr/sbin/nologin
3   sys    x 3     3 sys    /dev      /usr/sbin/nologin
    sync   x 4 65534 sync   /bin      /bin/sync
```

## 输出匹配行及行号

例如输出能匹配`nologin`的行以及它们的行号。

因为只输出某些匹配行，而不是所有行，所以不使用-p选项。
```
$ perl -ne 'print "$. $_" if /nologin/' file.log
2 daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
3 bin    x 2     2 bin    /bin      /usr/sbin/nologin
4 sys    x 3     3 sys    /dev      /usr/sbin/nologin
```

如果匹配行的行号要独立计数，则不使用`$.`，自己写个自增的计数器即可：
```
$ perl -ne 'print ++$num." $_" if /nologin/' file.log
1 daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
2 bin    x 2     2 bin    /bin      /usr/sbin/nologin
3 sys    x 3     3 sys    /dev      /usr/sbin/nologin
```


## 统计行数

```
$ perl -lne 'END{print $.}' file.log
5
```

这里使用END语句块，表示执行完主逻辑代码后程序退出前执行的，因为这个示例中没有主逻辑代码，所以读取完所有行后就会执行END语句块。另外，这里的-l选项主要用来为print追加换行符。

上面的语句仅会输出行号，不会输出文件名，而且多个文件的时候只会输出总行数，而不是每个文件单独统计。

还有其它实现方式，介绍两个：
```
# (1)
perl -le 'print scalar(@tmp = <>)' file.log
perl -le 'print ~~@tmp = <>' file.log

# (2)
perl -ne '}{print $.' file.log
```

上面的方式(1)没有使用-p和-n，所以自己在-e表达式中写`<>`。而`@tmp = <>`是让`<>`以列表的方式一次性读取所有行，然后scalar强制转换其为标量上下文，于是得到行数量。它等价于`scalar( () = <> )`，还等价于`$num = () =<>`。

scalar()可以替换成`~~`符号，它是两个比特位取反操作，等价于什么都不做，但它工作在标量上下文，所以可以用来转换上下文。

上面的方式(2)使用的是超乎想象的`}{`，这不是Perl中的什么特殊符号，仅仅只是结合-n选项时的一个技巧。-n选项的代码逻辑如下：
```
while(<>){
    ... -e expression here
}
```

所以，-e中指定`}{print $.`表示破坏原始的-n逻辑，使之变成下面的逻辑：
```
while(<>){
    }{print $.
}
```

这个格式化一下就是：
```
while(<>){}
{
    print $.
}
```

也就是说，while循环体内不做任何操作，直到`<>`读取完成后，while结束，然后运行一次性语句块`{print $.}`。所以，`-ne }{xxx`等价于在END语句块中执行`xxx`操作。


### 模仿 wc -l

`wc -l`会单独输出每个文件的行数，并总计所有文件的行数。例如：
```
$ wc -l file.log
5 file.log

$ wc -l file.log paragraph.log
  5 file.log
 18 paragraph.log
 23 total
```

所以，这也可以使用perl一行式程序来实现。因为这个逻辑中需要单独统计每个文件，所以必须显式区分每个文件。使用eof判断每个文件的尾部即可。

```
$ perl -M'List::Util qw(sum)' -lne '
        # 将@ARGV保存起来，以便后续能够按先后顺序获取所有文件名
        BEGIN{
            @files = @ARGV;
        }
        # 每个文件处理完时，保存行和文件信息，并关闭文件以便重置行号计数器
        if(eof){
            # 将每个文件的行数注册到一个hash结构中
            # hash的key为当前处理的文件名
            $line_filename{$ARGV} = $.;
            close ARGV;
        }
        END{
            # 获取总行数以及总行数的字符长度，以便格式化对齐
            $total_lines = sum values %line_filename;
            $longest = length $total_lines;

            # 输出每个文件对应的行数及文件名，且按照@ARGV的顺序输出
            foreach (@files){
                printf "%${longest}d %s\n",$line_filename{$_},$_;
            }
            # 输出总行数
            print "$total_lines total";
        }
    ' file.log paragraph.log
```

这是遇到的第一个比较大的程序，**这样的逻辑应该写成Perl脚本而不是一行式程序**。不过这里的几个知识点很适合引入Perl一行式。

先分析下这段程序的逻辑：要统计总行数，且要输出每个文件对应的行数，输出时还要进行格式化对齐，所以先将每个文件对应的行数保存到一个hash结构中，最后在END语句块中计算总行数，并计算总行数有多少个字符以便确定格式化对齐时的字符数量。

上面使用了`-M`选项，它表示导入一个模块。此处所导入的模块是`List::Util`模块，它是额外的列表(数组)工具模块，该模块中有不少处理列表的工具。例如这里使用`qw(sum)`表示导入这个模块中的sum函数，用于对列表元素进行加总。如果不写`qw(sum)`，那么在使用sum函数的时候，需要写完整的名称`List::Util::sum @arr`。

"-l"选项的目的是给print函数追加换行符。

另外这里使用了BEGIN语句块来保存`@ARGV`数组，虽然`%line_filename`中也能取得所有的文件名，但hash结构中的元素是无序的，要保证文件名的顺序，只能使用数组(列表)来保存。再者，因为`@ARGV`中的参数文件会随着`<>`的读取而被剔除出`@ARGV`，所以应该在BEGIN中对`@ARGV`进行保存。


## 统计非空白行的行数

用计数器实现非常简单：
```
$ perl -lne '++$num if /\S/;END{print $num+0;}' paragraph.log
```

这里的逻辑非常简单。唯一需要注意的是`$num+0`，因为文件可能是空的，使得END语句块中的`$num`变量仍处于未定义状态。加上一个`+0`，可以保证它会输出数值格式，未定义时则输出0。

为了保证得到数值，可以使用int函数进行转换：
```
$ perl -lne '++$num if /\S/;END{print int $num;}' paragraph.log
```

我准备在这里引入grep函数的简单用法。

Shell中有个grep命令可以用来匹配内容，在Perl中也有一个grep函数，它的简单工作方式可以类同于shell的grep命令，用于筛选列表中符合条件的元素，并将这些元素构成一个新的列表。比如能正则匹配的元素、操作后布尔真的元素。

所以，可以使用grep函数来匹配非空白行：
```
$ perl -e '@lines = grep /\S/,<>;print "@lines"' paragraph.log
```

grep期待的是列表上下文，使得`<>`一次性读完所有行形成一个列表，然后grep对这个列表的每个元素进行筛选，只要是非空白行都放入一个新的列表。

那么要统计非空白行数就非常简单了，直接将grep的结果转换成标量上下文就可以。
```
$ perl -le 'print ~~grep /\S/,<>' paragraph.log
```

前面说过，`~~`可以用来转换标量上下文。

其实Perl grep函数要强大的多，它支持完整的流程控制逻辑。如有需要，参考[Perl grep函数](https://www.cnblogs.com/f-ck-need-u/p/9678875.html#bloggrep)。

## 计数每个单词

为文件中每行中的单词进行计数。

```
$ perl -pe 's/(\w+)/"<".++$num.">.$1"/ge' file.log

<1>.first <2>.paragraph:
        <3>.first <4>.line <5>.in <6>.1st <7>.paragraph
        <8>.second <9>.line <10>.in <11>.1st <12>.paragraph
        <13>.third <14>.line <15>.in <16>.1st <17>.paragraph
```

这里使用`s///`命令的e修饰符。该修饰符可以评估`s/reg/replacement/`的replacement部分，将其作为Perl的代码被perl执行，然后进行s替换操作。

正如上面的示例，每一行匹配后评估replacement部分是`"<".++$num.">.$1"`，被perl执行的话，这里的`++$num`就会在perl环境下执行自增操作，`$1`也会被替换成已匹配的分组，最后完成s的替换。

下面的示例可能会更容易理解一些：
```
$ perl -pe 's/(\w+)/++$num/ge' file.log

1 2:
        3 4 5 6 7
        8 9 10 11 12
        13 14 15 16 17
```

如果想让它们顶格输出，可以继续删除行首空白：
```
$ perl -pe 's/(\w+)/++$num/ge;s/^\s+//' file.log
1 2:
3 4 5 6 7
8 9 10 11 12
13 14 15 16 17
```

再比如，每行的单词单独计数，只需在每次读入行的开头进行计数器重置即可：
```
$ perl -pe '$num=0;s/(\w+)/"<".++$num.">.$1"/ge' file.log

<1>.first <2>.paragraph:
        <1>.first <2>.line <3>.in <4>.1st <5>.paragraph
        <1>.second <2>.line <3>.in <4>.1st <5>.paragraph
        <1>.third <2>.line <3>.in <4>.1st <5>.paragraph
```
