---
title: tee的花式用法和pee：多重定向
p: shell/tee_pee.md
date: 2023-07-11 10:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------


# tee的花式用法和pee：多重定向

## tee多重定向

```bash
tee [options] FILE1 FILE2 FILE3...
```

tee的作用是将一份标准输入多重定向，一份重定向到标准输出/dev/stdout，然后还将标准输入重定向到每个文件FILE中。

例如：
```bash
$ cat alpha.log | tee file1 file2 file3 | cat
$ cat alpha.log | tee file1 file2 file3 >/dev/null
```

上面第一个命令将alpha.log的文件内容重定向给`file{1..3}`和标准输出通过管道传递给cat。

上面第二个命令将alpha.log的文件内容重定向给`file{1..3}`和/dev/null。  

## tee重定向给多个命令

写多了脚本的人可能遇到过这样一种需求：将一份标准输入，重定向到多个命令中去。大概是这样的：
```
                      | CMD1
                    ↗
        INPUT | tee 
                    ↘
                      | CMD2
```

其实bash自身的特性就能实现这样的需求，通过重定向到子shell中，就能模拟一个文件重定向行为：
```bash
cat alpha.txt | tee >(grep -E "a|b") >(grep -E "d|b|c")
```

(实际上这里的两个`>(cmd_list)`不是重定向，而是进程替换。命令行解析开始时，将首先进行进程替换，这两个grep将等待标准输入。然后启动cat和tee，然后tee将标准输出交给两个进程的标准输入)

上面的命令将alpha.txt文件内容重定向为3份：一份给第一个grep命令，一份给第二个grep命令，一份给标准输出。假如alpha.txt的内容是`a b c d e`5个字母分别占用5行(每行一个字母)，上面的输出结果如下：
```bash
a
b
c
d
e  # 前5行是重定向到/dev/stdout的
a
b  # 这2行是重定向给第一个grep后的执行结果
b
c
d  # 这3行是重定向给第二个grep后的执行结果
```

如果不想要给标准输出的那份重定向，加上`>/dev/null`：
```bash
cat alpha.txt | tee >(grep -E "a|b") >(grep -E "d|b|c") >/dev/null
```

## tee重定向给多个命令时的问题

但是必须注意，**tee将数据重定向给不同命令时，这些命令是独立执行的，它们都会各自打开一个属于自己的STDOUT**，如果它们都重定向到标准输出，由于涉及到多个不同的/dev/stdout，它们的结果将出现两个问题：  
1. **不保证有序性**  
2. 因为跨了命令，交互式模式下(默认标准输出为屏幕)可能会出现命令行隔断的问题(非交互式下不会有问题)  

例如：
```bash
$ cat alpha.txt | tee >(grep -E "a|b") >(grep -E "d|b|c") >/dev/null
$ a     # 结果直接出现在提示符所在行
b
b
c
d

$ cat alpha.txt | tee >(grep -E "a|b") >(grep -E "d|b|c") >/dev/null
b
c      # 这次的结果和上次的顺序不一样
d
a
b
```

这两个问题，在写脚本过程中必须解决。

对于第二个问题：不同/dev/stdout同时输出时在屏幕上交叉输出的问题，只需将它们再次重定向走即可，这样两份不同的/dev/stdout都再次同时作为一份标准输入：
```bash
$ cat alpha.txt | tee >(grep -E "a|b") >(grep -E "d|b|c") >/dev/null | cat
```

对于第一个问题：不同/dev/stdout同时输出时，输出顺序的随机性，这个没有好方法，只能在各命令行中将各自的结果保存到文件中：
```bash
$ cat alpha.txt | tee >(grep -E "a|b" >file1) >(grep -E "d|b|c" >file2) >/dev/null
```

所以，tee在重定向到多个命令中是有缺陷的，或者说用起来非常不方便，只要将各命令的结果各自保存时，才能一切按照自己的预期进行。那么，pee登场了，多重定向非常好用的一个命令。

## pee代替tee

pee是moreutils包中的一个小工具，先安装它(epel源中有)：
```bash
yum -y install moreutils
```

在man pee中，pee的作用是将标准输入tee给管道。语法：
```bash
pee ["cmds"]
```

不是很好理解，可以通过几个示例直接感受它的用法。
```bash
$ cat alpha.txt | pee 'grep -E "a|b"' 'grep -E "d|b|c"'
a
b
b
c
d
```

所以，它的基本用法是`pee "CMD1" "CMD2"`。

如果想将结果保存到文件，只需加一个命令即可，例如下面的`cat >myfile`。
```bash
$ cat alpha.txt | pee 'grep -E "a|b"' 'grep -E "d|b|c"' 'cat >myfile'
```

和tee有同样的问题，如果各命令都没有指定自己的标准输出重定向，它们将各自打开一个属于自己的/dev/stdout，**同样会有多个/dev/stdout同时输出时结果数据顺序随机性的问题**，但是不会有多个/dev/stdout同时输出时交互式的隔断性问题，因为**pee会收集各个命令的标准输出，然后将收集的结果作为自己的标准输出**。

pee和tee最大的不同，在于pee将来自多个不同命令的结果作为pee自己的标准输出，所以下面的命令是可以像普通命令一样进行重定向的。
```bash
INPUT | pee CMD1 CMD2 >/FILE
```
而tee则不同，是将cmd1和cmd2的结果放进标准输出(假设各命令自身没有使用重定向)，保存到FILE中的是tee读取的标准输入。
```bash
INPUT | tee >(cmd1) >(cmd2) >/FILE
```
所以，想要重定向tee中cmd1和cmd2的总结果，必须使用额外的管道，或者将整个tee放进子shell。
```bash
INPUT | tee >(cmd1) >(cmd2) >/dev/null | cat >FILE1
INPUT | ( tee >(cmd1) >(cmd2) >/dev/null ) >/FILE1
```
