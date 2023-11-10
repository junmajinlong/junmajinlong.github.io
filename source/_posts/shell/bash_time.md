---
title: 获取bash中的时间
p: shell/bash_time.md
date: 2023-07-25 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------


# 获取bash中的时间

下面是几个bash中有关于时间的几个内容(为了区分，下面的变量我都加了前缀`$`符号)：

- 特殊变量：`$SECONDS $EPOCHSECONDS $EPOCHREALTIME`  
- 内置关键字`time`和它的格式描述变量`$TIMEFORMAT`  

## bash内置特殊变量SECONDS

bash的内置变量`SECONDS`记录了bash进程从启动到现在已经过去了多少秒，它会读取硬件时钟来递增(当然，硬件时钟的时间减了，它也会减)。

`SECONDS`变量可以被重新赋值为数值(可以是0 负数 小数)，赋值为a后，SECONDS变量的值讲从a开始逐秒增加(如果是小数，则小数部分会被截断)。但不能赋值为空，赋值为空后它就从内置变量变成了一个没有特殊含义的普通变量，即使后来再给它赋值为整数值也不行。

```bash
# 该bash进程从启动到现在已经过去了459秒
$ echo $SECONDS
459

# 重新赋值为0
$ SECONDS=0

# 重新赋值后，过去了4秒
$ echo $SECONDS
4
```

通过`$SECONDS`可以用来计算一个命令或一个脚本运行了多长时间(只能精确到秒)。例如：

```bash
#!/bin/bash

# 脚本第一行讲该变量赋值为0
SECONDS=0

sleep 3
# do anything...

# 脚本末尾加上下行，输出该变量的值，就是脚本的运行时长
echo $SECONDS
```

另外，子bash进程会继承父bash进程的`SECONDS`变量。

```bash
$ echo $SECONDS;(echo $SECONDS;sleep 2;echo $SECONDS)
1224
1224
1226
```

## bash内置特殊变量EPOCHSECONDS

为了让你看清楚点，看看小写的`epoch seconds`。

和`SECONDS`变量类似，只不过`EPOCHSECONDS`记录的是从`1970-01-01 00:00:00.000`开始到现在已经过去了多少秒。

```bash
$ echo $EPOCHSECONDS
1690335995
```

由于`EPOCHSECONDS`是获取系统时间得到的，因此无所谓父子bash进程的继承关系。

当然，也可以用`EPOCHSECONDS`计算命令或脚本的运行时长。

```bash
#!/bin/bash

start=$EPOCHSECONDS

sleep 2
# do anything...

echo $EPOCHSECONDS - $start | bc
```

## bash内置特殊变量EPOCHREALTIME

为了让你看清楚点，看看小写的`epoch realtime`。

和`EPOCHSECONDS`一样，唯一的区别是它保存的是精确到微妙的小数值(浮点数)。

```bash
$ echo $EPOCHREALTIME
1690336492.178162
```

用来计算命令或脚本的执行时长：

```bash
#!/bin/bash

start=$EPOCHREALTIME

sleep 2
# do anything...

echo $EPOCHREALTIME - $start | bc
```

## time和bash内置变量TIMEFORMAT

`time`大家用的应该很多，用来统计命令的执行时长。

但注意，使用bash时，有两个time可用，一个是bash内置关键字，一个是外部命令。

```bash
# bash内置关键字time
$ time sleep 1
real    0m1.004s
user    0m0.000s
sys     0m0.000s

# 外部命令time
$ /usr/bin/time sleep 1
0.00user 0.00system 0:01.00elapsed 0%CPU (0avgtext+0avgdata 864maxresident)k
0inputs+0outputs (0major+241minor)pagefaults 0swaps
```

对于内置关键字time来说，它的默认输出中：

- `real`表示命令从执行到结束，在现实中逝去的时长，也就是我们人认为的命令真正的执行时长。  
- `user`表示命令从执行到结束，在用户空间中花费的时长。  
- `sys`表示命令从执行到结束，在内核空间中花费的时长。发起的系统调用次数越多，涉及的系统内核对内存的读写越多，sys时间就会越长  

内置关键字还可以直接统计bash语法的组合命令的时长。下面是等价的：

```bash
$ time { sleep 1;sleep 1; }
real    0m2.008s
user    0m0.000s
sys     0m0.000s

$ /usr/bin/time bash -c '{ sleep 1;sleep 1; }'
0.00user 0.00system 0:02.01elapsed 0%CPU (0avgtext+0avgdata 2500maxresident)k
0inputs+0outputs (0major+1139minor)pagefaults 0swaps
```

内置关键字time默认会输出`real user sys`三种时长，但可以通过bash内置变量`TIMEFORMAT`来指定输出格式。TIMEFORMAT的设置格式如下：

```
%%
A literal ‘%’.

%[p][l]R
The elapsed time in seconds.

%[p][l]U
The number of CPU seconds spent in user mode.

%[p][l]S
The number of CPU seconds spent in system mode.

%P
The CPU percentage, computed as (%U + %S) / %R.
```

其中：

- `p`是时长小数位，最大可指定为3，指定大于3的值则等价于3，且不指定p时，默认为3。指定为0则不包含小数，即只精确到秒。
- `l`是长格式的时长

```bash
# 精确到毫秒级别
$ TIMEFORMAT="消耗 %3R 秒"
$ time sleep 2
消耗 2.004 秒

# 指定为长格式
$ TIMEFORMAT="消耗 %3lR"
$ time sleep 2
消耗 0m2.004s
```

