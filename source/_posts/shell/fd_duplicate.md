---
title: Shell：彻底搞懂shell的高级I/O重定向
p: shell/fd_duplicate.md
date: 2019-07-12 17:37:29
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------


基本的重定向功能想必都理解，本文就对重定向稍作深入。

## 文件描述符(file description,fd)

文件描述符是IO重定向中的重要概念。文件描述符使用数字表示，它指明了数据的流向特征。

软件设计认为，程序应该有一个数据来源、数据出口和报告错误的地方。在Linux系统中，它们分别使用描述符0、1、2来表示，这3个描述符默认的目标文件(设备)分别是/dev/stdin、/dev/stdout、/dev/stderr，它们分别是各个终端字符设备的软链接。
```shell
$ ll /dev/std*
lrwxrwxrwx 1 root root 15 Apr  2 07:57 /dev/stderr -> /proc/self/fd/2
lrwxrwxrwx 1 root root 15 Apr  2 07:57 /dev/stdin -> /proc/self/fd/0
lrwxrwxrwx 1 root root 15 Apr  2 07:57 /dev/stdout -> /proc/self/fd/1

$ ll /proc/self/fd/
total 0
lrwx------ 1 root root 64 Apr  6 03:53 0 -> /dev/pts/2
lrwx------ 1 root root 64 Apr  6 03:53 1 -> /dev/pts/2
lrwx------ 1 root root 64 Apr  6 03:53 2 -> /dev/pts/2
lr-x------ 1 root root 64 Apr  6 03:53 3 -> /proc/14038/fd
```
在Linux中，每一个进程打开时都会自动获取3个文件描述符0、1和2，分别表示标准输入、标准输出、和标准错误，如果要打开其他文件，则文件描述符必须从3开始标识。对于我们人为要打开的描述符，建议使用9以内的描述符，超过9的描述符可能已经被系统内部分配给其他进程。

文件描述符说白了就是系统为了跟踪这个打开的文件而分配给它的一个数字，这个数字和文件绑定在一起，数据流入描述符的时候也表示流入文件。

而Linux中万物皆文件，这些文件都可以分配描述符，包括套接字。

程序在打开文件描述符的时候，有三种可能的行为：从描述符中读、向描述符中写、可读也可写。从lsof的FD列可以看出程序打开这个文件是为了从中读数据，还是向其中写数据，亦或是既读又写。例如，tail命令监控文件时，就是打开文件从中读数据的(3r的r是read，w是write，u是read and write)。
```shell
$ lsof -n | grep "/a.sh" | column -t
tail  13563  root  3r  REG  8,2  182  69632966  /root/a.sh
```


![](/img/referer.jpg)

## 文件描述符的复制(duplicate)

文件描述符的复制表示复制文件描述符到另一个文件描述符中以作其副本。使用"&"进行复制。

- `[n]<&word`：**将文件描述符n复制于word 代表的文件或描述符。可以理解为文件描述符n重用word代表的文件或描述符，即word原来对应哪个文件，现在n作为它的副本也对应这个文件**。n不指定则默认为0(标准输入就是0)，表示标准输入也将输入到word所代表的文件或描述符中。  
- `[n]>&word`：**将文件描述符n复制于word 代表的文件或描述符。可以理解为文件描述符n重用word代表的文件或描述符，即word原来对应哪个文件，现在n作为它的副本也对应这个文件**。n不指定则默认为1(标准输出就是1)，表示标准输出也将输出到word所代表的文件或描述符中。  

例如，`3>&1`表示fd=3复制于fd=1，而fd=1目前的重定向目标文件是/dev/stdout(fd=1指向与输出设备是默认的)，因此fd=3也重定向到/dev/stdout，以后进程将数据写入fd=3的时候，将直接输出到屏幕。这里的`3>&1`等价于`3>&/dev/stdout`。如果用"复制"来理解，就是fd=3是当前fd=1的一个副本，即指向/dev/stdout设备。如果后面改变了fd=1的输出目标(如file1)，由于fd=3的目标仍然是/dev/stdout，所以可以拿fd=3来还原fd=1使其目标变回/dev/stdout。

```
(fd=1) --> /dev/stdout
  |
 3>&1
 \|/
(fd=3) --> /dev/stdout
```

>**关于文件描述符的duplicate**  
>>在操作系统(或C)中，对于实体文件的文件描述符来说，**文件描述符是用来描述它所指向的实体文件**的。例如fd=5指向文件a.txt。复制(duplicate)实际上是执行dup()函数，表示创建另一个文件描述符(例如fd=6)，指向同一个底层对象，例如指向同一个实体文件。这时fd=5和fd=6都将指向a.txt。  
>>在shell中，我们将文件描述符和实体文件的关联关系(或者称为指向的关系)称为重定向，其实用更底层的指向关系更容易理解。例如，"3>&1"表示复制fd=1，使得fd=3和fd=1都指向同一个对象，也就是stdout。  

再例如，`cat <&1`表示fd=0复制于fd=1上，而此时fd=1的重定向文件是/dev/stdout，所以fd=0也指向这个/dev/stdout文件，而cat从fd=0中读取标准输入，于是/dev/stdout既是标准输入设备，也是标准输出设备，也就是说进程从/dev/stdout(屏幕)接受输入，输入后再直接输出到/dev/stdout。以下是结果：
```shell
$ cat <&1
q   # 进入交互式，输入数据
q   # 直接输出
```

## 重定向顺序很重要：`>file 2>&1`和`2>&1 >file` 

想必很多人都知道`>file 2>&1`的作用，它等价于`&>file`，表示标准输出和标准错误都重定向到file中。那它和`2>&1 >file`有什么区别呢？

首先解释`>file 2>&1`。这里分两个过程：先打开file，再将fd=1重定向到file文件上，这样file文件就成了标准输出的输出目标；之后再将fd=2复制于fd=1，而fd=1此时已经重定向到file文件上，因此fd=2也重定向到file上。所以，最终的结果是标准输出重定向到file上，标准错误也重定向到file上。

再解释`2>&1 >file`。这里也分两个过程：先将fd=2复制于fd=1，而此时fd=1重定向的文件是默认的/dev/stdout，所以fd=2也重定向到/dev/stdout；之后再将fd=1重定向到file文件上。也就是说，这里的标准错误和标准输出仍然是分开输出的，只不过是使用/dev/stdout替代了/dev/stderr，使用file替代了/dev/stdout。所以，最终的结果是标准错误输出到/dev/stdout，即屏幕上，而标准输出将输出到file文件中。

可以使用下面的命令来测试`2>&1 >file`。第一个ls命令是正确的，结果输出到/tmp/a.log中，第二个ls命令是错误的，结果将直接输出到屏幕上。
```shell
$ ls /boot 2>&1 >/tmp/a.log
$ ls sjdfk 2>&1 >/tmp/a.log
ls: cannot access sjdfk: No such file or directory
```

最后需要说明的是一种特殊情况，如果是`>&word`，且word不是一个数值，比如`echo haha >&/tmp/a.log`，那么`>&word`和`&>word`是等价的，都表示`>word 2>&1`，即标准错误和标准输出都重定向同一个目标。参考man bash的"Redirecting Standard Output and Standard Error"段落。

![](/img/referer.jpg)

## 改变当前shell环境的重定向目标

如果在命令中直接改变重定向的位置，那么命令执行结束的时候描述符会自动还原。正如上面的`ls /boot 2>&1 >/tmp/a.log`命令，在ls执行结束后，fd=2还原回默认的/dev/stderr，fd=1还原回默认的/dev/stdout。

但是我们可以通过exec程序直接在当前的shell环境下改变重定向目标，只有在当前shell退出的时候才会释放描述符的绑定。

例如：下面的命令将标准错误fd=2指向fd=3对应的文件上。

```
exec 2>&3
```
因此，我们可能在一段程序执行结束后，需要将描述符还原到原来的位置，并关闭不再需要的描述符。毕竟描述符也是资源，是有限的(ulimit -n)。


## 关闭文件描述符


- `[n]>&-`  
- `[n]<&-`  

关闭文件描述符的方式是将`[n]>&word`和`[n]<&word`中的word使用符号"-"，这表示释放fd=n描述符，且关闭其指向的文件。

## 打开文件

- `[n]<> filename`：打开filename，并指定其文件描述符为n，该描述符是可读、可写的描述符。若不指定n则默认为0，若filename文件不存在，则先创建filename文件。

例如：

```shell
$ exec 3<> /tmp/a.log
$ lsof -n | grep "/a.log" | column -t  
bash  13637  root  3u  REG  8,2  292018  69632965  /tmp/a.log
```
如果再`exec 1>&3`将fd=1复制于fd=3，那么/tmp/a.log就成了标准输出的目标。

## 文件描述符的移动

文件描述符的移动表示将文件描述符1移动到描述符2上，同时关闭文件描述符1。

- `[n]>&digit-`：将文件描述符digit代表的输出文件移动到n上，并关闭digit值的描述符  
- `[n]<&digit-`：将文件描述符digit代表的输入文件移动到n上，并关闭digit值的描述符  

例如：
```shell
$ exec 3<> /tmp/a.log
$ lsof -n | grep "/a.log" | column -t  
bash  13637  root  3u  REG  8,2  292018  69632965  /tmp/a.log

$ exec 1>&3-  # 将3移动到1上，关闭3
$ lsof -n | grep "/a.log" | column -t   # 在另一个bash窗口查看
bash  13637  root  1u  REG  8,2  292018  69632965  /tmp/a.log
```
可见，fd=3移动到fd=1后，原本与fd=3关联的/tmp/a.log已经关联到fd=1上。

![](/img/referer.jpg)

## 经典示例

**(1).示例一**

以下是《Advanced Bash-Scripting Guide》中的示例：
```
echo 1234567890 > File # (1).写字符串到"File".
exec 3<> File          # (2).打开"File"并且给它分配fd 3.
read -n 4 <&3          # (3).只读4 个字符.
echo -n . >&3          # (4).写一个小数点.
exec 3>&-              # (5).关闭fd 3.
cat File               # (6).1234.67890
```
(1)向文件File中写入几个字符。  
(2)打开文件File以备read/write，并分配fd=3给该文件。  
(3)将fd=0复制于fd=3上，而fd=3的重定向目标为File，所以fd=0的目标也是File，即从File中读取数据。这里读取4个字符，由于read命令中没有指定变量，因此分配给默认变量REPLY。注意，这个命令执行结束后，fd=0的重定向目标会变回/dev/stdin。  
(4)将fd=1复制于fd=3上，而fd=3的重定向目标文件为File，所以fd=1的目标也是File，即数据写入到File中。这里写入一个小数点。注意，这个命令结束后，fd=1的重定向目标回变回/dev/stdout。  
(5)关闭fd=3，这也会关闭其指向的文件File。  
(6)File文件中已经写入了一个小数点。如果此时执行echo $REPLY，将输出"1234"。  

**(2).示例二：关于描述符恢复、关闭**

```
exec 6>&1                   # (1)
exec > /tmp/file.txt        # (2)
echo "---------------"      # (3)
exec 1>&6 6>&-              # (4)
echo "==============="      # (5)
```
(1)首先将fd=6复制于fd=1，此时fd=1的重定向目标为/dev/stdout，因此fd=6的重定向目标为/dev/stdout。  
(2)将fd=1重定向到/tmp/file.txt文件。此后所有标准输出都将写入到/tmp/file.txt中。  
(3)写入数据。该数据将写入到/tmp/file.txt中。  
(4)将fd=1重新复制回fd=6，此时fd=6的重定向目标为/dev/stdout，因此fd=1将恢复到/dev/stdout上。最后将fd=6关闭。  
(5)写入数据，这段数据将输出在屏幕上。  

可能你会疑惑，为什么要先将fd=1复制于fd=6，再用fd=6来恢复fd=1，恢复的时候直接将fd=1重定向回/dev/stdout不就可以了吗？

实际上，这里借用fd=6这个中转描述符是为了方便操作。可以不用它，但是在恢复fd=1的重定向目标的时候，应该重定向到`/dev/{伪终端字符设备}`上，而不是/dev/stdout，因为/dev/stdout是软链接，其目标指向/proc/self/fd/1，但该文件还是软链接，它指向/dev/{伪终端字符设备}。同理/dev/stdin和/dev/stderr都一样。

因此，如果你当前所在的终端如果是pts/2，那么可以使用下面的命令来实现上面同样的功能：
```shell
exec > /tmp/file.txt
echo "---------------" 
exec >/dev/pts/2
echo "==============="

# exec >/dev/tty  # 更方便
```
如果不借用fd=6这个中转描述符，你要先去获取并记住当前shell所在的终端，很不方便。但可以使用`/dev/tty`这个文件来表示当前所在终端，这会方便的多。

但如果要恢复的不是终端相关的文件，那么可能就只能通过文件描述符的备份、还原来恢复了。

最后给张描述符复制、恢复的过程实例图：

![](/img/shell/733013-20180406121920101-752401821.jpg)

## 使用变量作为文件描述符

有时候一些特殊的需求下，可能想要使用变量来保存所分配的文件描述符，从而让多个手动打开的文件夹描述符不至于前后混乱。

要用变量保存文件描述符，可以采用如下方式：

```shell
fd=3
eval "exec ${fd}<> /tmp/a.log"
lsof -n | grep a.log
```

在bash 4.1之后，bash自身提供了变量文件描述符的功能，只要在需要分配文件描述符的时候将原来的fd指定为`{fdvar}`即可创建这个变量，在分配文件描述符后会自动将其保存到变量fdvar中。使用这种模式时，文件描述符是从10开始分配的，所以fdvar是大于等于10的值。

```shell
exec {fd1}<> /tmp/a.log
echo $fd1   # 输出：10

exec {fd2}<> /tmp/a.log
echo $fd2   # 输出：11
```


