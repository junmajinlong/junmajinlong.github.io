---
title: Bash mapfile按行读取到bash数组
p: shell/mapfile.md
date: 2023-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------


# Bash mapfile按行读取到bash数组

Bash提供了两个内置命令：readarray和mapfile，它们是同义词。它们的作用是从标准输入读取一行行的数据，然后每一行都赋值给一个数组的各元素。显然，在shell编程中更常用的是从文件、从管道读取，不过也可以从文件描述符中读取数据。

需要先说明的是，shell并不像其它专门的编程语言对数组、列表提供了大量的操作工具，反而直接操作文本文件更为常见(sed、awk等)，所以mapfile用的并不多。

## 1.语法
```
mapfile [OPTIONS] ARRAY
readarray [OPTIONS] ARRAY

其中options:
-O INDEX   ：指定从哪个索引号开始存储数据，默认存储数据的起始索引号为0
-n count   ：最多只拷贝多少行到数组中，如果count=0，则拷贝所有行
-s count   ：忽略前count行不读取
-c NUM     ：每读取NUM行就调用一次"-C callback"选项指定的callback程序
-C callback：每读取"-c NUM"选项指定的NUM行就执行一次callback回调程序
-d string  ：指定读取数据时的行分隔符，默认是换行符
-t         ：移除尾随行分隔符，默认是换行符
-u fd      ：指定从文件描述符fd而非标准输入中读取数据
```

- 如果不指定`ARRAY`参数，则默认使用数组**MAPFILE**  
- 如果不指定`-O`选项，则在存储数据之前先清空数组(如果该数组已存在)  
- 给定了`-C callback`却没有给定`-c NUM`时，则默认为每5000行调用一次回调程序  
- 回调程序是在读取给定行数之后，赋值到数组元素之前执行的。所以流程为：`读NUM行-->callback-->赋值`  
- 每次调用回调函数时，都将调用callback之前的最后一行数据及其对应的索引号作为回调程序的参数。例如`-c 3 -C callback`，则会将索引号2和第3行内容，索引号5和第6行内容作为callback程序的参数  
- `-t`去除行尾分隔符，一般来说都是换行符。用其他语言编程过的人都知道行尾换行符有多烦心，但对于shell编程来说，倒是无所谓  

## 2.几个示例和注意事项

先创建一个示例用的文件alpha.log，每行一个小写字母，共26行：
```bash
$ echo {a..z} | tr " " "\n" >alpha.log
$ cat alpha.log
a
b
c
d
e
f
g
h
i
j
k
l
m
n
o
p
q
r
s
t
u
v
w
x
y
z
```

读取该文件并将每一行存储到数组myarr中(如果不指定，则存储到默认的MAPFILE数组中)。
```bash
$ mapfile myarr <alpha.log
$ echo ${myarr[@]}
a b c d e f g h i j k l m n o p q r s t u v w x y z
$ echo ${myarr[2]}
c
```

既然是读取标准输入，常见的就有以下几种读取形式：
```bash
$ mapfile myarr <alpha.log            # 1.输入重定向
$ mapfile myarr < <(cat alpha.log)    # 2.进程替换
$ cat alpha.log | mapfile myarr       # 3.管道传递
```

第1、2种写法没什么问题，但第3种写法是有问题的。
```bash
$ cat alpha.log | mapfile myarr1
$ echo ${#myarr1[@]}
0
```
从结果中可以看到，myarr1根本就不存在。为什么？这里简单说明一下，对于管道组合的多个命令，它们都会放进同一个进程组中，会进入子shell执行相关操作。当执行完毕后，进程组结束，子shell退出。而子shell中设置的环境是不会粘滞到父shell中的(即不会影响父shell)，所以myarr1数组是子shell中的数组，回到父shell就消失了。

解决方法是在子shell中操作数组：
```bash
$ cat alpha.log | { mapfile myarr1;echo ${myarr1[@]}; }
```

mapfile可以指定每读取多少行就执行一次的回调函数，并且会将执行回调函数时读取的最后一行和对应的索引号传递给回调函数作为它额外的参数。

一个简单的示例，每读取3行就执行一次echo，注意看下面传递给给echo的参数值。
```bash
$ mapfile -c 3 -C "echo" myarr <alpha.log
2 c

5 f

8 i

11 l

14 o

17 r

20 u

23 x

```
这里的echo就是回调函数。输出结果中每执行一次就有一空行，这是因为文件中数据是分行的，而echo又自带换行功能。所以，可以使用`-t`选项，在每次读取一行后就去掉该行的换行符。
```bash
$ mapfile -t -c 3 -C "echo" myarr <alpha.log
2 c
5 f
8 i
11 l
14 o
17 r
20 u
23 x
```

可以写一个脚本，或者定义一个函数作为回调程序，实现更复杂的功能，但一定要注意，mapfile传递给callback的两个参数总是最后两个参数。例如：
```bash
$ myecho(){ echo $@; };mapfile -t -c 3 -C "myecho haha" myarr <alpha.log
haha 2 c
haha 5 f
haha 8 i
haha 11 l
haha 14 o
haha 17 r
haha 20 u
haha 23 x
```

还可以将多个操作组合起来作为一个回调程序：
```bash
$ mapfile -t -c 3 -C "echo haha;echo" myarr<alpha.log
haha
2 c
haha
5 f
haha
8 i
haha
11 l
haha
14 o
haha
17 r
haha
20 u
haha
23 x
```

