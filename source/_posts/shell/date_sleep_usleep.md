---
title: date、sleep、usleep命令
p: shell/date_sleep_usleep.md
date: 2023-07-06 18:20:20
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# date、sleep、usleep命令

## date命令

date用于获取和设置操作系统的时间，还有hwclock是获取硬件时间。

date有个选项`-d`，可以用来描述获取什么时候的时间，描述的方式非常开放，但不能使用`now`关键字，其他的如3天前`3 days ago`，3天后`3 days`，昨天`yesterday`，下周一`next Monday`，epoch时间`@EPOCH`等等。

Linux中设置date命令的显示格式：`date [+format]`，其中`+`表示从前面的时间中获取其中的格式部分，如`date -d "yesterday" +"%Y"`获取的是昨天的年份部分。

format格式如下所示：标红色的较常用。

```
%y: 年(后两位)，00-99
%Y: 年，四位

%m: 月，01..12
%j: 日，年中某天，(001..366)
%d: 日，月中某天，如01 
%w: 日，周中某天，0..6，0是周日
%u: 日，周中某天，1..7，1是周一
%U: 周，年中某周，00-53，周日为星期第一天
%W: 周，年中某周，00-53，周一是星期第一天

%H: 时，24小时制，00..23
%I: 时，12小时制，01..12
%M: 分，00..59
%S: 秒，00..60
%N: 纳秒
%s: Epoch，从1970-01-01到目前时间的总秒数

%T: 时间部分，等价于%H:%M:%S，如 12:23:30
%F: 日期部分，等价于%Y-%m-%d，如 2015-03-12
%D: 日期部分，等价于%m/%d/%y，如 03/12/15

%n: 换行
%t: 制表符(tab)
```

下面是一些示例：

```bash
[root@xuexi ~]# date +%F
2016-09-25

# 有空格需要使用双引号或引号来分隔
[root@xuexi ~]# date +"%F %T"
2016-09-25 10:48:34

[root@xuexi ~]# date +"%Y-%m-%d %H:%M:%S"
2016-09-25 10:47:49
```

使用date命令可以计算时间差。例如：

```bash
# 以下3个命令等价
date -d "3 days ago" +%F
date -d "-3 days" +%F
date -d "now  - 3 days" +%F
```

再例如，给定一个时间，计算它的前几天，后几天。

```bash
# 以下两条命令等价
date -d "2018-02-19 3 days ago" +%F
date -d "2018-02-19 - 3 days" +%F
```

给定一个日期，计算该日期所在星期的星期一是几月几号。例如，`2018-05-12`是星期六，那么星期一是`2018-05-07`。

```bash
#!/bin/bash
src_date="2018-05-12"
src_weekday=`date -d $src_date +%w`
Mon_date=`date -d "$src_date - $(( src_weekday - 1 )) days" +%F`
echo $Mon_date
```

date命令还可以计算延迟时间(两个时间点的时间差)。如果要计算精确度为秒级的延迟，可直接使用`%s`计算，但如果要计算毫秒级、微秒级甚至是纳秒级的时间差，则需要对date的结果进行一番计算和转换。

```bash
#!/bin/bash
start_time=$(date +"%s")
find / -type f -name "*.db" &>/dev/null
end_time=$(date +"%s")
time_diff=$(( start_time - end_time ))
echo $time_diff
```

## sleep和usleep

在shell中常使用`sleep`命令指定休眠时间，休眠的意思表示让当前进程进入睡眠状态。例如：

```bash
sleep 5
```

`sleep`默认的休眠单位为秒，因此上面表示休眠5秒钟。如果要休眠毫秒级、微秒级，则可以使用小数。例如下面命令表示休眠半秒钟：

```bash
sleep 0.5
```

此外，还有专门的微秒级的休眠命令`usleep`。例如：

```bash
usleep 1000
```

表示休眠1000微秒，即1毫秒。