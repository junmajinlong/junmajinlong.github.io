---
title: Perl一行式：选择输出、删除的行
p: perl/perl_oneliner_5.md
date: 2019-07-08 17:39:54
tags: Perl
categories: Perl
---

# Perl一行式：选择输出、删除的行

**perl一行式程序系列文章**：[Perl一行式](/perl/index#blogperloneline)

-----------------------------

对于Perl的一行式perl程序来说，选择要输出的、要删除的、要插入/追加的行是非常容易的事情，因为print/say决定行是否输出/插入/追加/删除。虽然简单，但对于广泛应用在sed的示例还是可以拿到这里来讨论一番。

因为输出/删除/插入/追加行都是通过print/say在不同条件下的操作，所以本文只会介绍输出操作，删除/插入/追加其实都是同样的原理。

## 输出第一行

```
$ perl -lne 'print;exit' file.log
```

## 输出第13行

```
$ perl -ne 'print if $. == 13' file.log
```

## 输出前10行

```
$ perl -ne 'print if $.<=10' file.log
$ perl -ne 'print if 1..10' file.log
$ perl -ne '$. <= 10 && print' file.log
$ perl -ne 'print; exit if $. == 10' file.log
```

## 输出最后一行

```
$ perl -ne '$last=$_;END{print $last}' file.log
```

或者通过文件结尾eof来判断：
```
$ perl -ne 'print if eof' file.log
```

这里的eof函数的作用是：如果下一行读取到了文件尾部eof，就返回1。否则


## 输出倒数10行

这个实现起来可能稍显复杂，但逻辑很简单：向一个数组中添加10行元素，如果数组元素个数超过了10，则剔除数组的第一个元素。

```
$ perl -ne '
    push @lines,$_;
    if(@lines>10){
        shift @lines;
    }
    END{
        print @lines
    }
    ' /etc/passwd
```

这里是shift一个元素来保证【窗口】的稳定性：最多只有10个元素。另一种稳妥的方式是直接切片，从数组中取最后10个元素：
```
$ perl -ne '
    push @lines,$_;
    @lines = @lines[@lines-10..$#lines] if @lines>10;
    END{print @lines}
    ' /etc/passwd
```



## 输出倒数第11行到倒数第2行

有了前一个示例作为基础，这个需求很容易实现。

保留一个11行元素的数组，最后输出前10个元素即可。

```
$ perl -ne '
    push @a,$_;
    shift @a if @a>11;
    END{print @a[0..$#a-1]}
    ' /etc/passwd
```

## 输出文件的第偶数行

这个很简单，只需判断行号的奇偶性即可。

```
$ perl -ne 'print if $. % 2 == 0' file.log
$ perl -ne 'print unless $. % 2' file.log
```

## 输出能匹配的行

```
$ perl -ne 'print if /regexp/' file.log
```

## 输出两个匹配之间的行

```
$ perl -ne 'print if /regexp1/../regexp2/' file.log
```

## 输出匹配行的前一行

只需将每行保留到变量中，如果当前行匹配了，则输出上一行保存的值。
```
$ perl -ne '/regexp/ && $last && print $last;$last = $_' file.log
```

如果想要输出匹配的前M行，只需把这些数量的行保存到数组中，并不断地shift剔除就可以。

## 输出匹配行的后一行

```
$ perl -ne '$p && print; $p = /regexp/' file.log
```

Perl中正则表达式的匹配操作返回的是成功与否的布尔真假，所以`$p = /regexp/`表示如果匹配了，则`$p`的值为真，否则为假。

如果`$p`为真，则下一行将被输出，且继续对输出行进行匹配，如果输出行仍然能匹配，则继续输出下一行。

上面的过程可以改写成逻辑更为清晰的一行式：
```
$ perl -ne 'if($p){print;$p=0}++$p if /regexp/' file.log
```

上面的`$p`是一个状态标记变量，如果匹配成功，就标记为真值，并在输出的时候重置状态变量。

还可以采用另一种处理逻辑：自己编写从`<>`读取行的while循环，如果匹配了就继续读入下一行。因为读入的下一行可能继续匹配，所以在while循环中使用redo逻辑回到while循环的开头。
```
$ perl -se '
    while(<>){
        if(/$reg/){
            if(eof){ exit; }
            print $_ = <>;
        }
        redo if /$reg/;
    }
    ' -- -reg="REGEXP" file.log
```


## 输出匹配行及其后5行

上面采用状态标记变量`$p`，这个状态标记变量可以更深入地使用。

如果匹配了，则`$p`设置为5，然后输出后面的行时对`$p`自减。

```
$ perl -ne '
    if($p){print;$p--}
    if(/regexp/){$p = 5;print};
    ' file.log
```


## 连续行去重

```
$ perl -ne '
    next if "$line" eq "$_";
    print $line = $_;
    ' file.log
```



