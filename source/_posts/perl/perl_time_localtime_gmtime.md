---
title: Perl的time、localtime和gmtime函数
p: perl/perl_time_localtime_gmtime.md
date: 2019-07-06 17:37:59
tags: Perl
categories: Perl
---

# Perl的time、localtime和gmtime函数

- time用于返回当前时间点，返回格式是以从1970年1月1日(纪元由操作系统决定，但unix系统一般都是1970年1月1日)距离现在的秒数表示的epoch  
- localtime用于返回给定时间的秒、分、时、日、月、年、周等9个部分的时间属性，参数为epoch时间格式，不给参数则返回当前时间点对应的秒、分、时、日、月、年等属性  
- gmtime和localtime类似，但是返回的UTC时间  

```
print time,"\n";
```

localtime在列表上下文返回的是各个时间部分，在标量上下文返回的是一个本地格式的时间值。

```
[root@xuexi perlapp]# perl -e '$a=localtime;print $a,"\n";' 
Sat Sep  8 09:03:56 2018
```

以下是localtime在列表上下文返回的各个时间部分：
```
#  0    1    2     3     4    5     6     7     8
($sec,$min,$hour,$mday,$mon,$year,$wday,$yday,$isdst) = localtime(time);
```

其中:
- sec：秒
- min：分
- hour：时
- mday：日，即当月的第几天
- mon：月份，值为0-11，0表示1月，11表示12月，如此表示的好处后面解释
- year：年，返回从1900年开始的整数值，如果要返回4位数的年份，将其加上1900即可
- wday：周几，值为0-6，0是周日，1是周一，6是周六
- yday：一年的第几天，值为0-364或0-365
- isdst：是否是夏令时

例如：
```
use 5.010;

@time=localtime;

say qq(second  : $time[0]);
say qq(minute  : $time[1]);
say qq(hour    : $time[2]);
say qq(mon_day : $time[3]);
say qq(month   : $time[4]);
say qq(year    : $time[5]);
say qq(week_day: $time[6]);
say qq(year_day: $time[7]);
say qq(isdst   : $time[8]);
```

输出结果为：
```
second  : 42
minute  : 10
hour    : 9
mon_day : 8
month   : 8
year    : 118
week_day: 6
year_day: 250
isdst   : 0
```

之所以用0表示1月份，11表示12月份，是为了让月份数值和偏移对应。例如，偏移0位表示1月。
```
my @month = qw(Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec);
print "$month[$mon]"
```

如果想要格式化输出日期时间，则可以采用printf：

```perl
($S,$M,$H,$d,$m,$y) = localtime;
printf "%04d-%02d-%02d %02d:%02d:%02d\n",1900+$y,1+$m,$d,$H,$M,$S;
```

但是不推荐这种方式。可以使用POSIX中提供的strftime来格式化时间：

```perl
use POSIX qw(strftime);

strftime("%F %T", localtime);
```

