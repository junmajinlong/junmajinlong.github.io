---
title: Shell脚本深入教程：Bash高级重定向
p: shell/script_course/shell_redirection.md
date: 2020-05-19 14:49:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------


# Bash高级重定向

在Shell中，具备重定向基础之后就能满足绝大多数需求了，但是还可以学习更多关于文件描述符的操作。

Linux中读写文件都会通过文件描述符来完成，而不是通过文件名来完成的，所以只要打开文件，Linux内核就会为程序分配一个文件描述符。例如`cat a.log`会打开a.log文件并分配一个文件描述符，cat命令退出后，打开的文件就会被关闭，所分配的文件描述符也会释放。

## 获取当前终端

当前终端是标准输入fd=0、标准输出fd=1、标准错误fd=2的默认数据流目标。

如何知道当前是在哪个终端下呢？Linux一切皆文件，终端也是文件，是终端设备文件，在/dev目录下。

tty命令可以获取当前Shell所在的终端，这是最快获取终端的方式。

```shell
$ tty
/dev/pts/0
```

此外，还能通过其它间接方式来肉眼判断，比较麻烦罢了。

```shell
 $ readlink /proc/self/fd/1
/dev/pts/0

$ ps -o tty | tail -n 1
pts/0
```

## Bash初始分配的文件描述符

bash自身也是一个进程，它会打开0、1、2三个文件描述符，它们默认都对应终端文件。只要在bash中运行的外部命令，都是bash的子进程，所以会继承这三个文件描述符。

此外，如果bash是交互式的bash，那么它还需要连接到一个终端上，终端也是文件，所以交互式bash要为连接的终端文件分配一个文件描述符，默认分配的是fd=255。

```shell
$ echo $$
799
$ ls -l /proc/799/fd/
total 0
lrwx------ 1 root root 64 Jan 26 21:48 0 -> /dev/pts/1
lrwx------ 1 root root 64 Jan 26 21:48 1 -> /dev/pts/1
lrwx------ 1 root root 64 Jan 26 21:48 2 -> /dev/pts/1
lrwx------ 1 root root 64 Jan 26 21:49 255 -> /dev/pts/1
```

此外，如果使用了路径自动提示功能(比如两次tab键)，在提示目录中的文件时，需要打开那个目录，所以要为这个目录分配一个文件描述符，因为提示或补全也是bash的内置功能，所以是bash负责打开那个目录的，所分配的文件描述是当前最小的可用的整数。但是，如果没有使用路径提示或补全，则不会分配该文件描述符。

```shell
$ ls -l /proc/799/fd/   # tab两次
0    1    2    255  3    
$ ls -l /proc/799/fd/
```

## 再看基础重定向操作

对于基本的重定向操作应该都很熟悉，比如：

```shell
$ echo hello world >/tmp/a.log
$ cat </tmp/a.log
```

但是在命令行中指定的重定向目标都只在该命令中有效，命令退出后，重定向行为就消失了。

如果想让Shell下的所有命令都向某文件写数据呢？那么就在bash下修改fd=1的目标即可：

```shell
exec >/tmp/a.log
```

exec是shell内置命令，那么第一时间我们就要想一想这是不是一个Shell环境设置类的操作。

答案是确定的，这个操作会设置当前bash进程的标准输出fd=1的数据流目标为/tmp/a.log。

由于bash下运行的命令都是bash的子进程，会继承bash进程的设置，所以经过上面的设置后，其它命令的标准输出默认也会输出到/tmp/a.log文件。

```shell
echo helloworld
cat /etc/fstab
```

同理，对标准输入也是一样的设置方式：

```shell
exec </tmp/a.log
```

那如何恢复bash的标准输出呢？只需要将fd=1重新重定向到终端设备文件上即可：

```shell
exec >$(tty)
cat /etc/fstab
```

## open file

其实，每个重定向操作也是打开文件的操作，同时会分配文件描述符。

比如`echo ha >a.log`命令，Shell会打开a.log文件并将fd=1关联到该文件，当echo进程运行时会继承Shell的fd=1以及它关联的a.log属性，于是echo的标准输出就会输出到a.log中。

但是在命令中打开的文件(包括重定向)都是临时的，命令退出完后文件就会关闭，所分配的文件描述符也会释放。

在Shell中，可以手动打开文件：

```shell
exec N> FILENAME   # 覆盖式只写模式打开
exec N>> FILENAME  # 追加式只写模式打开
exec N< FILENAME   # 只读模式打开
exec N<> FILENAME  # 可读可写(覆盖式)模式打开，不能设置可读且追加模式打开
```

整数N建议在[3,9]范围内，超出9的文件描述符有可能已经被bash内部使用了。

这表示在当前Shell进程内打开文件FILENAME并分配文件描述符N，只要不手动关闭文件，只要当前Shell进程不退出，那么打开的FILENAME就一直处于打开状态。

例如：

```shell
$ exec 3<> /tmp/a.log
$ lsof -n | grep /tmp/a.log | grep -v grep
bash  342   root  3u  REG   8,2   0  101410762 /tmp/a.log
```

## 重定向方式读写文件描述符

绝大多数时候，读写文件都是直接重定向文件名的，例如：

```shell
echo hello world >/tmp/a.log
echo hello world >>/tmp/a.log
cat </tmp/a.log
```

但是也可以直接重定向读写文件描述符。

```shell
>&N  # 写向文件描述符N(对应的目标文件)
<&N  # 从文件描述符N中读取数据
```

实际上，`>&N`等价于`1>&N`，即将标准输出的内容写向fd=N，至于原因，参考下文文件描述符的复制(fd duplicate)。

例如：

```shell
exec 3> /tmp/a.log
echo hello world >&3
```

此外，bash的内置命令`read`命令也可以直接从文件描述符中读取数据，参考后文。

## close file

在编程的时候，不再使用的文件就要关闭，以便释放文件描述符，防止程序一直占用文件，导致无法释放文件占用的磁盘空间。如果大量文件描述符不关闭，还可能达到打开文件的上限，使得程序报错。

虽然在Shell中可以无视这种问题，但有时候为了测试或为了某些逻辑，还是需要关闭文件描述符。

关闭文件描述符的方式：

```shell
[exec] N>&-
[exec] N<&-
```

这表示关闭文件描述符N。如果是用`exec N<>FILENAME`打开的文件，则上面两种方式都能关闭N。

```shell
$ exec 3<> /tmp/a.log
$ lsof -n | grep /tmp/a.log | grep -v grep
bash  342   root  3u  REG   8,2   0  101410762 /tmp/a.log

$ exec 3<&-  # 或exec 3>&-
$ lsof -n | grep /tmp/a.log | grep -v grep

```

## fd duplicate

通常被翻译为文件描述符的复制。

当内核为某个打开的文件FILE分配文件描述符N后，N和FILE就有了关联关系。可以简单而不严谨地认为，fd=N指向了磁盘文件FILE。

比如`exec 3<> FILE`表示当前Shell环境下，fd=3已经关联到了FILE文件。而对于`cat a.log >>b.log`命令，可认为fd=1关联到了b.log文件。

文件描述符的复制语法：

```shell
[exec] [M]>&N   # 省略M时，默认M=1，即标准输出
[exec] [M]<&N   # 省略M时，默认M=0，即标准输入
```

fd dup表示新分配一个文件描述符M，使得M也关联到N关联的文件FILE上。换句话说，M是N的一个副本，就像是复制了N文件描述符一样，于是现在M和N都关联到了FILE文件。这就像是复制了软链接，两个软链接都指向同一个目标文件。

例如：

```shell
$ exec 3<> a.log
$ exec 4>&3
$ lsof -n | grep /tmp/a.log | grep -v grep
bash  342  root  3u  REG  8,2  0  69331702 /root/a.log
bash  342  root  4u  REG  8,2  0  69331702 /root/a.log
```

整个过程如图：

![](/img/shell/1581478384933.png)

因为fd=3和fd=4都关联在a.log上，所以无论是使用fd=3还是fd=4来写入数据，都是向a.log写入数据。

那如何向fd=3或fd=4写入数据呢？只能借助fd=1，因为Shell下的命令(除非是你自己写的脚本或程序)默认都使用标准输出fd=1来输出正确数据，fd=2输出错误数据，并且默认使用标准输入fd=0来读取数据。

所以，只需要将标准输出fd=1关联到fd=3或fd=4上即可，这样命令写入fd=1的数据，也会写入fd=3和fd=4对应的磁盘文件中，因为它们三都关联在同一个磁盘文件上。

```shell
$ echo helloworld >&3
$ cat a.log
helloworld

$ echo HELLOWORLD >&4
$ cat a.log
helloworld
HELLOWORLD
```

注意上面echo命令中的fd dup是临时的，echo退出后因回到了Shell进程，使得fd=1又重新关联到/dev/stdout，也即当前终端。

可以使用exec直接改变当前Shell环境的标准输入、标准输出、标准错误的默认目标：

```shell
exec >&3   # 当前Shell所有标准输出都将输出到fd=3所关联的的文件
exec 2>&3  # 当前Shell所有标准错误都将输出到fd=3所关联的的文件
exec <&4   # 当前Shell将默认从fd=4所关联的的文件中读取数据
```

这样修改后，即使之后的命令中不指定重定向，它们的输出、输入也会是a.log。

```shell
$ echo hellohellohello
```

一般修改fd=0、1、2的目标都是临时行为，之后都会将其重新恢复其目标，即关联到当前终端。两种方式可恢复：

```shell
# 方式1.改变目标之前先备份当前目标，以便恢复时使用
exec 9>&1  # fd=1和fd=9都关联到终端
exec >&3   # 改变fd=1的目标，此时fd=9仍然关联到终端
......     # 一些操作
exec >&9 ;exec 9>&-  # 恢复fd=1使其关联到终端，然后关闭fd=9

# 方式2.直接通过tty获取当前终端，并重新重定向
exec >&3  # 改变fd=1的目标
......    # 一些操作
exec >$(tty)  # tty获取当前终端，然后重定向改变目标
```

## fd move

还可执行文件描述的移动操作。文件描述符的移动，表示先复制源文件描述符，再删除源文件描述符。

```shell
[exec] [M]>&N-    # 省略M时，默认M=1
[exec] [M]<&N-    # 省略M时，默认M=0
```

这等价于：

```shell
[exec] [M]>&N;N>&-    # 省略M时，默认M=1
[exec] [M]<&N;N<&-    # 省略M时，默认M=0
```

即，先fd dup得到fd=N的副本fd=M，同时关闭fd=N。这就实现了从fd=N关联文件FILE变成fd=M关联文件FILE。

例如：

```shell
$ exec 4<> /tmp/a.log
$ exec 5>&4-
$ lsof -n | grep /tmp/a.log | grep -v grep
bash  1118  root  5u  REG  8,2  39  101410762 /tmp/a.log
```

## 自动分配文件描述符号

使用重定向方式打开文件时，可以将手动指定的文件描述符数值指定为一个`{var_name}`格式，这样bash会自动分配文件描述符号，并将文件描述符数值保存到bash变量var_name中。

```bash
exec {tmp_alog}<>/tmp/a.log
echo $tmp_alog       # 11
exec {tmp_alog}> /tmp/a.log
echo $tmp_alog       # 12
```

关闭文件描述符时也可以使用这个变量：

```bash
exec {tmp_alog}>&-
```

## read从文件描述符读取数据

read是bash内置命令，它默认从标准输入中读取数据，但也可以使用`-u FD`选项指定从文件描述符中读取数据。

例如，两个文件a.log和b.log：

```
# a.log的内容
a
b
c

# b.log的内容
aa
bb
cc
```

现在想要将两个文件内容一一对应起来，得到：

```
a aa
b bb
c cc
```

实现的方法有很多种，但如果使用read呢？

```shell
exec 3<> a.log
exec 4<> b.log
while read -u 3 linea;do
  read -u 4 lineb
  echo $linea $lineb
done
exec 3>&-
exec 4>&-
```

如果想省略关闭文件描述符的操作，只需将打开文件和while循环放在子Shell这中执行即可，子Shell的环境设置不会影响父Shell：

```shell
(exec 3<> a.log;exec 4<> b.log;while read -u 3 linea;do read -u 4 lineb;echo $linea $lineb;done)
```
