## time

perl内置的`time`函数获取从`1970-01-01 00:00:00`开始到现在流逝的秒数，即epoch时间。

```shell
$ perl -E 'say time'
1625729583    
# date -d@1625729583 +"%F %T" => 2021-07-08 15:33:03
```

time只能精确到秒级别，如果想要获取更低精度的时间，可使用`Time::HiRes`模块。稍后再介绍这个模块的用法。

## localtime

perl内置的`localtime`函数，根据给定的epoch时间参数计算对应的时间，如未给定参数，则以当前时间点的epoch作为参数：  

- 在列表上下文，返回对应时间的各个日期时间部分  
- 在标量上下文直接返回对应的时间点，返回的时间格式为操作系统所默认的格式(即和date命令返回的结果一样)  

例如，在标量上下文：

```shell
$ perl -E '$a=localtime;say $a'
Thu Jul  8 15:49:01 2021

$ date
Thu Jul  8 15:49:01 CST 2021
```

在列表上下文：

```perl
# 2021-07-08 15:52:11
$ perl -E 'say "@{[localtime]}"'
11 52 15 8 6 121 4 188 0

# 2015-10-20 12:30:22
$ perl -E 'say "@{[localtime(1445315422)]}"' 
22 30 12 20 9 115 2 292 0
```

在列表上下文，localtime返回共9项信息，它们的顺序分别是：

```perl
#     0    1    2     3     4    5     6     7     8
my ($sec,$min,$hour,$mday,$mon,$year,$wday,$yday,$isdst) = localtime(time);
```

即：秒、分、时、日(月中日)、月、年、周中日、年中日、是否夏令时。主要使用的是前6项。

需注意几个事项：

- mon月份的范围是0-11，0表示1月，11表示12月  
- year年份的值是从1900年开始的整数值，因此它是2位数或3位数，如果要四位数表示，加上1900  
- wday表示的周几，0表示周日，6表示周六  

通过localtime得到时间的各部分后，如果要将它格式化为某种格式的日期时间字符串，可考虑使用`printf`。

```perl
my ($S, $M, $H, $d, $m, $y) = localtime;
printf "%04d-%02d-%02d %02d:%02d:%02d\n",1900+$y,1+$m,$d,$H,$M,$S;
```

但是不推荐这么啰嗦的格式化方式，可引入POSIX中提供的`strftime`函数来格式化时间：

```perl
use POSIX qw(strftime);
strftime("%F %T", localtime);  # 2021-07-08 16:00:39
```

更多日期时间的操作，可使用`Time::Piece`模块或`DateTime`模块。

## gmtime

gmtime和localtime的用法完全一致，唯一的区别是`gmtime`将给定的时间参数当作UTC时间，而不是本地时间。

