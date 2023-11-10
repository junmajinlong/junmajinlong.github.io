---
title: 精通awk系列(28)：awk预定义函数(3)：日期时间类预定义函数
p: shell/awk/awk_builtin_datetime_function.md
date: 2020-04-12 15:51:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# awk时间类内置函数

awk常用于处理日志，它支持简单的时间类操作。有下面3个内置的时间函数：  
- `mktime("YYYY MM DD HH mm SS [DST]")`：构建一个时间，返回这个时间点的秒级epoch，构建失败则返回-1  
- `systime()`：返回当前系统时间点，返回的是秒级epoch值  
- `strftime([format [, timestamp [, utc-flag] ] ])`：将时间按指定格式转换为字符串并返回转的结果字符串  

注意，awk构建时间时都是返回秒级的epoch值，表示从`1970-01-01 00:00:00`开始到指定时间已经过的秒数。

```
awk 'BEGIN{print systime();print mktime("2019 2 29 12 32 59")}'
1572364974
1551414779
```

## awk mktime()

mktime在构建时间时，如果传递的DD给定的值超出了月份MM允许的天数，则自动延申到下个月。例如，指定"2019 2 29 12 30 59"中2月只有28号，所以构建出来的时间是`2019-03-01 12:30:59`。

此外，其它部分也不限定必须在范围内。例如，`2019 2 23 12 32 65`的秒超出了59，那么多出来的秒数将进位到分钟。
```
awk 'BEGIN{
    print mktime("2019 2 23 12 32 65") | "xargs -i date -d@{} +\"%F %T\""
}'
2019-02-23 12:33:05
```
如果某部位的数值为负数，则表示在此时间点基础上减几。例如：
```
# 2019-02-23 12:00:59基础上减1分钟
$ awk 'BEGIN{print mktime("2019 2 23 12 -1 59") | "xargs -i date -d@{} +\"%F %T\""}'  
2019-02-23 11:59:59

# 2019-02-23 00:32:59基础上减1小时
$ awk 'BEGIN{print mktime("2019 2 23 -1 32 59") | "xargs -i date -d@{} +\"%F %T\""}'  
2019-02-22 23:32:59
```

## awk strftime()

```
strftime([format [, timestamp [, utc-flag] ] ])
```

将指定的时间戳tiemstamp按照给定格式format转换为字符串并返回这个字符串。

如果省略timestamp，则对当前系统时间进行格式化。

如果省略format，则采用PROCINFO["strftime"]的格式，其默认格式为`%a %b %e %H:%M%:S %Z %Y`。该格式对应于Shell命令`date`的默认输出结果。
```
$ awk 'BEGIN{print strftime()}'
Wed Oct 30 00:20:01 CST 2014

$ date
Wed Oct 30 00:20:04 CST 2014

$ awk 'BEGIN{print strftime(PROCINFO["strftime"], systime())}'
Wed Oct 30 00:24:00 CST 2014
```

支持的格式包括：
```
%a 星期几的缩写，如Mon、Sun Wed Fri
%A 星期几的英文全名，如Monday
%b 月份的英文缩写，如Oct、Sep
%B 月份的英文全名，如February、October
%C 2位数的世纪，例如1970对应的世纪是19
%y 2位数的年份(00–99)，通过年份模以100取得，例如2019/100的余数位19
%Y 四位数年份(如2015)
%m 月份(01–12)
%j 年中天(001–366)
%d 月中天(01–31)
%e 空格填充的月中天
%H 24小时制的小时(00–23)
%I 12小时制的小时(01–12)
%p 12小时制时的AM/PM
%M 分钟数(00–59)
%S 秒数(00–60)
%u 数值的星期几(1–7)，1表示星期一
%w 数值的星期几(0–6)，0表示星期日
%W 年中第几周(00–53)
%z 时区偏移，格式为"+HHMM"，如"+0800"
%Z 时区偏移的英文缩写，如CST

%k 24小时制的时间(0-23)，1位数的小时使用空格填充
%l 12小时制的时间(1-12)，1位数的小时使用空格填充
%s 秒级epoch

##### 特殊符号
%n 换行符
%t 制表符
%% 百分号%

##### 等价写法：
%x 等价于"%A %B %d %Y"
%F 等价于"%Y-%m-%d"，用于表示ISO 8601日期格式
%T 等价于"%H:%M:%S"
%X 等价于"%T"
%r 12小时制的时间部分格式，等价于"%I:%M:%S %p"
%R 等价于"%H:%M"
%c 等价于"%A %B %d %T %Y"，如Wed 30 Oct 2015 12:34:48 AM CST
%D 等价于"%m/%d/%y"
%h 等价于"%b"
```

例如：
```
$ awk 'BEGIN{print strftime("%s", mktime("2077 11 12 10 23 32"))}'
3403909412

$ awk 'BEGIN{print strftime("%F %T %Z", mktime("2077 11 12 10 23 32"))}' 
2077-11-12 10:23:32 CST

$ awk 'BEGIN{print strftime("%F %T %z", mktime("2077 11 12 10 23 32"))}' 
2077-11-12 10:23:32 +0800
```

## awk将日期时间字符串转换为时间：strptime1()

例如：
```
2019-11-11T03:42:42+08:00
```

1.将日期时间字符串中的年月日时分秒全都单独保存起来  
2.将年月日时分秒构建成mktime()的字符串格式"YYYY MM DD HH mm SS"  
3.使用mktime()可以构建出时间点  

```
function strptime(str,    time_str,arr,Y,M,D,H,m,S){
    time_str = gensub(/[-T:+]/," ","g",str)
    split(time_str, arr, " ")
    Y = arr[1]
    M = arr[2]
    D = arr[3]
    H = arr[4]
    m = arr[5]
    S = arr[6]
    # mktime失败返回-1
    return mktime(sprintf("%d %d %d %d %d %d", Y,M,D,H,m,S))
}

BEGIN{
  str = "2019-11-11T03:42:42+08:00"
  print strptime(str)
}
```

## awk将日期时间字符串转换为时间：strptime2()

下面是更难一点的，月份使用的是英文或英文缩写，日期时间分隔符也比较特殊。

```
Sat 26. Jan 15:36:24 CET 2013
```

```
function strptime(str,     time_str,arr,Y,M,D,H,m,S){
    time_str = gensub(/[.:]+/, " ", "g", str)
    split(time_str, arr, " ")
    Y = arr[8]
    M = month_map(arr[3])
    D = arr[2]
    H = arr[4]
    m = arr[5]
    S = arr[6]
    return mktime(sprintf("%d %d %d %d %d %d", Y,M,D,H,m,S))
}

function month_map(str,   mon){
    # mon = substr(str,1,3)
    # return (((index("JanFebMarAprMayJunJelAugSepOctNovDec", mon)-1)/3)+1)
    mon["Jan"] = 1
    mon["Feb"] = 2
    mon["Mar"] = 3
    mon["Apr"] = 4
    mon["May"] = 5
    mon["Jun"] = 6
    mon["Jul"] = 7
    mon["Aug"] = 8
    mon["Sep"] = 9
    mon["Oct"] = 10
    mon["Nov"] = 11
    mon["Dec"] = 12
    return mon[str]
}

BEGIN{
    str = "Sat 26. Jan 15:36:24 CET 2013"
    print strptime(str)
}
```
