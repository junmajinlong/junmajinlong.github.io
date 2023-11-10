---
title: Ruby日期时间处理详解
p: ruby/ruby_datetime.md
date: 2020-05-14 13:42:24
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby日期时间处理

![](/img/ruby/1589464325239.png)
```ruby
require 'date'   # 提供Date和DateTime类
require 'time'   # 提供Time类(可直接使用，但导入后有更多方法)
```

一般来说，操作日期时间的常用操作包括：
1. 创建(构建)日期、时间对象
2. 在字符串和日期时间对象之间进行转换
3. 日期时间的比较
4. 日期时间的运算
5. 操作时区
6. 等等...

而这3个类中，都各自提供了一些方法，很多方法是重叠的。据我个人测试，DateTime这个标准库效率是最高的。

下面，针对各种常见功能将这3个类结合在一起去介绍。

## Ruby构建日期时间对象

这3个类都能直接构造日期、时间对象。其中Date只能构造日期不能构造时间对象。

### Date构造日期对象

**Date构造日期对象。如果提供了时间部分，则忽略时间部分**。因为不涉及时间部分，所以不能也没必要指定时区。

```ruby
# 1.构造当前日期对象：Date.today
>> Date.today
=> #<Date: 2019-08-05 ((2458701j,0s,0n),+0s,2299161j)>

>> puts Date.today
2019-08-05

# 2.构造指定日期：Date.new
## 可以构造超出2038年的日期
## 如果没有给定月、日，则默认为1
## 如果没有提供任何参数，默认是-4712年1月1日，这没什么意义
## 如果给定时间部分，则报错
>> Date.new(2018,3,20)
=> #<Date: 2018-03-20 ((2458198j,0s,0n),+0s,2299161j)>
>> Date.new(2078,3,20)
=> #<Date: 2078-03-20 ((2480113j,0s,0n),+0s,2299161j)>
>> Date.new(1900)
=> #<Date: 1900-01-01 ((2415021j,0s,0n),+0s,2299161j)>
>> Date.new
=> #<Date: -4712-01-01 ((0j,0s,0n),+0s,2299161j)>

# 3.构造指定日期：Date.parse
## 根据字符串解析成日期格式
## 如果给定时间部分，则忽略
## 对于年份，可以给定1、2位数的和4位数的
## 对于1、2位数，如果数值是大于等于69的，则默认加上1900
## 对于0和68之间的数值，则默认加上2000
>> Date.parse('2007/09/12')
=> #<Date: 2007-09-12 ((2454356j,0s,0n),+0s,2299161j)>
>> Date.parse('2007/9/12')
=> #<Date: 2007-09-12 ((2454356j,0s,0n),+0s,2299161j)>
>> Date.parse('2007-9-12')
=> #<Date: 2007-09-12 ((2454356j,0s,0n),+0s,2299161j)>
>> Date.parse('2007-9-12 12:30:59')
=> #<Date: 2007-09-12 ((2454356j,0s,0n),+0s,2299161j)>

>> Date.parse('08-9-12')
=> #<Date: 2008-09-12 ((2454722j,0s,0n),+0s,2299161j)>
>> Date.parse('68-9-12')
=> #<Date: 2068-09-12 ((2476637j,0s,0n),+0s,2299161j)>
>> Date.parse('69-9-12')
=> #<Date: 1969-09-12 ((2440477j,0s,0n),+0s,2299161j)>

# 4.使用strptime方法根据给定格式的字符串转换成日期时间对象
## 关于支持的格式，参见后文
>> Date.strptime('2001-02-03', '%Y-%m-%d')
=> #<Date: 2001-02-03 ((2451944j,0s,0n),+0s,2299161j)>
>> Date.strptime('02-03-2001', '%m-%d-%Y')
=> #<Date: 2001-02-03 ((2451944j,0s,0n),+0s,2299161j)>
```

### Time构造日期时间对象

常见的方法是：new(别名now)、at、local(别名mktime)、parse。

注：Time类没有strptime方法将给定格式的字符串转换成日期时间对象。

```ruby
# 1.new()或now()构建当前日期时间对象
## new也可以根据给定参数构建日期时间对象，无参时等价于now
## 可以指定时区
>> Time.now
=> 2019-08-05 14:12:30 +0800
>> Time.new
=> 2019-08-05 14:12:31 +0800
>> Time.new(2007,11,6,17,10,0, "+08:00")  # 指定时区
=> 2007-11-06 17:10:00 +0800

# 2.at()将epoch转换成日期时间对象
## 支持时区，支持小数秒、毫秒、微秒、纳秒
>> Time.at(1553488199)
=> 2019-03-25 12:29:59 +0800
>> Time.at(1553488199, in: '+08:00') # 指定时区
=> 2019-03-25 12:29:59 +0800
>> Time.at(1553488199.3).usec  # 小数秒0.3秒，即3毫秒
=> 299999
>> Time.at(1553488199,123.345,:millisecond).usec # 毫秒参数
=> 123344
>> Time.at(1553488199,123.345).nsec   # 微秒参数
=> 123344
>> Time.at(1553488199,123.345,:usec).nsec   # 微秒参数
=> 123344
>> Time.at(1553488199,123,:nsec).nsec # 纳秒参数
=> 123

# 3.mktime或local根据参数构建本地时区的日期时间对象
## 要构建UTC时区时间，使用gm()或别名方法utc()
## mktime/local和gm/utc之间，除时区不同外，其它等价
>> Time.mktime(2009,10,23,14,3,6)
=> 2009-10-23 14:03:06 +0800
>> Time.mktime(2009,10,23)
=> 2009-10-23 00:00:00 +0800
>> Time.mktime(2009,10)
=> 2009-10-01 00:00:00 +0800

# 4.parse根据字符串转换成日期时间对象，同样可以指定时区
>> Time.parse("2009/12/25 12:29:59")
=> 2009-12-25 12:29:59 +0800
>> Time.parse("2009/12/25")
=> 2009-12-25 00:00:00 +0800
>> Time.parse("2009/12")
=> 2009-12-01 00:00:00 +0800
>> Time.parse("2009/12/25 12:29:59 +00:00") # 指定时区
=> 2009-12-25 12:29:59 +0000
>> Time.parse('2009-07-12 16:32:40.00123').nsec # 指定小数秒
=> 1230000
```

由于上面介绍的几种Time类方法构建出来的日期时间对象的时区默认是`+08:00`(东八区，中国所在时区即东八)，所以指定时区与否可随意。

但下面DateTime类的几种方法构造日期时间对象的时区默认是`+00:00`，所以通常要指定时区。

### DateTime构造日期时间对象

DateTime是Date的子集，且由于某些方法被重写，所以某些方法参数有一点点不同。最常使用的构造器是new、now、parse和strptime。

另外，DateTime对象是包含纳秒的。

```ruby
# 1.new()方法构造DateTime对象
## 可以指定时区、指定小数秒
>> DateTime.new(2009,1,2,3,4,5)
=> #<DateTime: 2009-01-02T03:04:05+00:00 ((2454834j,11045s,0n),+0s,2299161j)>
>> DateTime.new(2009,1,2,3,4,5,'+7') # 指定时区
=> #<DateTime: 2009-01-02T03:04:05+07:00 ((2454833j,72245s,0n),+25200s,2299161j)>
>> DateTime.new(2009,1,2,3,4,5,'+08:00')
=> #<DateTime: 2009-01-02T03:04:05+08:00 ((2454833j,68645s,0n),+28800s,2299161j)>
>> DateTime.new(2009,1,2,3,4,5.3,'+08:00') # 指定小数秒，即毫秒
=> #<DateTime: 2009-01-02T03:04:05+08:00 ((2454833j,68645s,300000000n),+28800s,2299161j)>
>> DateTime.new(2009,1,2,3,4,5.3,'+08:00').sec_fraction
=> (3/10)

# 2.now()获取当前时间：
>> DateTime.now
=> #<DateTime: 2019-08-05T15:00:42+08:00 ((2458701j,25242s,407854800n),+28800s,2299161j)>

# 3.parse解析字符串为日期时间对象
>> DateTime.parse('2009-12-20 12:03:30')
=> #<DateTime: 2009-12-20T12:03:30+00:00 ((2455186j,43410s,0n),+0s,2299161j)>
>> DateTime.parse('2009-12-20 12:03:30 +8')
=> #<DateTime: 2009-12-20T12:03:30+08:00 ((2455186j,14610s,0n),+28800s,2299161j)>

# 4.strptime解析给定格式的字符串为日期时间对象
## 关于支持的格式，参见后文
>> DateTime.strptime('2009-12-20 12:03:30 +8','%Y-%m-%d %H:%M:%S %z')
=> #<DateTime: 2009-12-20T12:03:30+08:00 ((2455186j,14610s,0n),+28800s,2299161j)>
```

## 日期、时间有效性检测

构建日期时间对象时，有些日期、时间是无效的，但因为取值范围的不同、特殊日期特殊时间点的取值不同，导致处理比较麻烦。

比如，每月可能有29、30、31天，但11月31号是无效的。再比如，秒数的范围是在0-60，但第61秒仅作为闰秒时才是有效值，所以几乎所有时间的第61秒都是错误的秒数。

Time可以检测范围外的无效时间，但是不能检测范围内的无效时间。比如Time知道11月32号是错误的，但它不知道11月31号是错误的，实际上Time会将有效范围内超出的部分进位，比如11月31号进位到12月1号。
```ruby
# 有效日期
>> Time.new(2007,11,30,17,10,30)
=> 2007-11-30 17:10:30 +0800

# 范围内的无效日期，进位到12月1号
>> Time.new(2007,11,31,17,10,30)
=> 2007-12-01 17:10:30 +0800

# 有效秒
>> Time.new(2007,11,31,17,10,59)
=> 2007-12-01 17:10:59 +0800

# 第61秒进位到下一分钟
>> Time.new(2007,11,31,17,10,60)
=> 2007-12-01 17:11:00 +0800

# 错误的秒，报错
>> Time.new(2007,11,31,17,10,61)
ArgumentError: sec out of range
from (pry):25:in `initialize'

# 错误的日期，报错
>> Time.new(2007,11,32,17,10,59)
ArgumentError: argument out of range
from (pry):26:in `initialize'
```

Time会进位的特性有时候是有益的，但不利于检测。好在，Date、DateTime可以很好的检测无效的日期、时间，只要是无效的日期时间，它们都会报错，其中Date只能检测无效日期，DateTime可检测无效日期，也能检测无效时间。
```ruby
# Date检测无效日期
>> Date.new(2007,11,30)
=> #<Date: 2007-11-30 ((2454435j,0s,0n),+0s,2299161j)>
>> Date.new(2007,11,31)
ArgumentError: invalid date
from (pry):28:in `initialize'

# DateTime检测无效日期、时间
>> DateTime.new(2007,11,31,11,12,13)
ArgumentError: invalid date
from (pry):31:in `new'
>> DateTime.new(2007,11,30,11,12,60)
ArgumentError: invalid date
from (pry):32:in `new'
```

既然它们会报错，那么只要加上异常捕获的代码即可完成日期时间的有效性检测。比如在Time类中使用DateTime类来检测有效的日期和时间：
```ruby
class Time
  def self.valid?(y,m=1,d=1,H=0,M=0,S=0,us=0)
    require 'date'
    begin
      dt = DateTime(y,m,d,H,M,S,us)
    rescue
      returin nil
    end
    dt
  end
end
```

## Date、Time、DateTime对象之间的类型转换

这三个类的渊源：  
- Time类是对底层C库的时间函数的封装，它们通常基于UNIX epoch，因此不能表示1970年以前的时间  
- Date类是对Time类的补充，用于处理1970年之前的时间，但是它只能处理日期不能处理时间  
- DateTime继承自Date，可以处理任意时间点的日期时间，但有些Time的功能DateTime没有  

所有有些时候有必要对它们进行类型的转换。

这3个类都提供了下面3个方法，在各自之间进行转换：

```ruby
to_date
to_time
to_datetime
```

例如：
```ruby
# Date对象转换
>> Date.today.to_date
=> #<Date: 2019-08-05 ((2458701j,0s,0n),+0s,2299161j)>
>> Date.today.to_time
=> 2019-08-05 00:00:00 +0800
>> Date.today.to_datetime
=> #<DateTime: 2019-08-05T00:00:00+00:00 ((2458701j,0s,0n),+0s,2299161j)>

# Time对象转换
>> Time.now.to_date
>> Time.now.to_time
>> Time.now.to_datetime

# DateTime对象转换
>> DateTime.now.to_date
>> DateTime.now.to_time
>> DateTime.now.to_datetime
```

需要注意，Date类对象由于不包含时间部分，它也没有时区。所以：
- Time/DateTime转成Date后会丢失时间和时区部分的数据
- Date对象to_time转成Time对象，设置的时区是其默认时区『+08:00』，而to_datetime转成DateTime对象，设置的时区是其默认时区『+00:00』。但是Time和DateTime之间的转换不会变换时区

```ruby
d = Date.new(2019,5,23)
d.to_time  #=> 2019-05-23 00:00:00 +0800
d.to_datetime.to_s #=> "2019-05-23T00:00:00+00:00"
```

## 日期时间和字符串、数值之间的转换

这是常用的需求。

- 日期时间到字符串：
    - to_s转成特定的字符串格式
    - strftime转成自定义的字符串格式，参见下文
- 日期时间到数值：只有包含时间部分(即Date不在讨论范围)才能转成数值
    - 通常转成的是epoch时间，支持整数、浮点数、分数
- 字符串转成日期时间：前文构造日期时间对象部分已介绍
- 数值转日期时间：前文构造日期时间对象部分已介绍

日期时间对象还支持转换成数组。

先看Time类型对象，它支持to_s、to_f、to_i、to_r(转分数)、to_a(转数组)。

```ruby
# 转字符串
>> Time.now.to_s
=> "2019-08-05 15:29:49 +0800"

# 转数值，即整数epoch，和Time.at是相反的功能
>> Time.now.to_i
=> 1564990192

# 转浮点数，即小数epoch
## 默认保留6位小数，即微秒级别
## 可以格式化保留成纳秒级别的字符串
## 但无法转成纳秒级别的浮点数，因为超出了float精度范围
## 可以使用BigDecimal保存成科学记数法的浮点数
>> Time.now.to_f
=> 1564990197.671165  # 微秒
>> "%.9f" % Time.now.to_f
=> "1564990502.569714785"  # 纳秒字符串
>> ("%.9f" % Time.now.to_f).to_f
=> 1564991500.1034229  # 超出，被剪掉

## 使用BigDecimal保存十进制科学记数法浮点数
>> require 'bigdecimal'
>> BigDecimal("%.9f" % Time.now.to_f) # 纳秒浮点数
=> 0.1564991123879058599e10

# 转分数
>> Time.now.to_r
=> (3912475500217301/2500000)

# 转数组
## 10个数组元素
## [sec,min,hour,day,month,year,wday,yday,isdst,zone]
## [秒/分/时/日/月/年/周中天/年中天/是否夏令时/时区]
>> Time.now.to_a
=> [4, 30, 15, 5, 8, 2019, 1, 217, false, "DST"]
```

注意，Time所查看的时区以DST、CST、UTC、GMT这样的方式显示，而DateTime查看的时区则是以类似于『+08:00』这种方式显示。

再看Date和DateTime转这些类型。其实Date/DateTime支持的转换方法很少，它们只支持`to_s`，其它方法都不支持。所以，要想转成整数，可以先to_time，再to_i。
```ruby
>> Date.today.to_s
=> "2019-08-05"

>> Date.today.to_time
=> 2019-08-05 00:00:00 +0800
>> Date.today.to_time.to_i
=> 1564934400

>> DateTime.now.to_s
=> "2019-08-05T15:55:05+08:00"
>> DateTime.now.to_time.to_f
=> 1564991712.1981702
```

## strftime方法

strptime是将给定格式的字符串转成(解析成)日期时间对象，而strftime则是将日期时间转成给定格式的字符串。

Date、Time和DateTime这3个类都具有strftime。

```ruby
>> d = Date.today
>> d.strftime("%Y-%m-%d %H:%M:%S")
=> "2019-08-05 00:00:00"

>> dt = DateTime.now
>> dt.strftime("%Y-%m-%d %H:%M:%S")
=> "2019-08-05 16:00:03"

>> t = Time.now
>> t.strftime("%Y-%m-%d %H:%M:%S")
=> "2019-08-05 16:01:07"
```

其中百分号部分就是日期时间中的格式化字符串占位符。

另外，strftime和strptime所使用的格式是统一的。以下摘自官方手册：[strftime](https://ruby-doc.org/stdlib-2.6.2/libdoc/date/rdoc/Date.html#method-i-strftime)。

```
%<flags><width><modifier><conversion>
Flags:

-  don't pad a numerical output.
_  use spaces for padding.
0  use zeros for padding.
^  upcase the result string.
#  change case.
The minimum field width specifies the minimum width.

The modifiers are “E”, “O”, “:”, “::” and “:::”. “E” and “O” are ignored. No effect to result currently.

Format directives:

Date (Year, Month, Day):
  %Y - 4位整数年份，可以是负数
  %C - year / 100 (round down.  20 in 2009)
  %y - year % 100 (00..99)

  %m - 两位数月份，不足时填充0(01..12)
          %_m  blank-padded ( 1..12)
          %-m  no-padded (1..12)
  %B - 英文名称的月份全称(January)
          %^B  大写的月份全称(JANUARY)
  %b - 简写月份名称(Jan)
          %^b  大写的月份简写(JAN)
  %h - 等价于%b

  %d - 两位数月中天，不足时填充0(01..31)
          %-d  no-padded (1..31)
  %e - 月中天，空格填充不足位数( 1..31)

  %j - 年中天，0填充(001..366)

Time (Hour, Minute, Second, Subsecond):
  %H - 24-hour clock, zero-padded (00..23)
  %k - 24-hour clock, blank-padded ( 0..23)
  %I - 12-hour clock, zero-padded (01..12)
  %l - 12-hour clock, blank-padded ( 1..12)
  %P - lowercase(am or pm)
  %p - uppercase(AM or PM)

  %M - Minute of the hour (00..59)

  %S - Second of the minute (00..60)

  %L - 毫秒(000..999)
  %N - 小数秒，默认是9位小数，即纳秒
      %3N  毫秒millisecond (3 digits)
      %6N  微秒microsecond (6 digits)
      %9N  纳秒nanosecond  (9 digits)
      %12N 皮秒picosecond  (12 digits)
      %15N femtosecond (15 digits)
      %18N attosecond  (18 digits)
      %21N zeptosecond (21 digits)
      %24N yoctosecond (24 digits)

Time zone:
  %z - 时区，从UTC开始偏移(例如：+0900)
      %:z - 时区，从UTC开始偏移，但带一个冒号(+09:00)
      %::z - 时区，从UTC开始偏移，但带两个冒号(+09:00:00)
      %:::z - 时区，从UTC开始偏移，冒号随意(+09,+09:30,+09:30:30)
  %Z - 等价于%:z (+09:00)

Weekday:
  %A - 周几的全称(Sunday)
          %^A  大写的周几全称(SUNDAY)
  %a - 周几的简称(Sun)
          %^a  大写的周几简称(SUN)
  %u - 数值周几(1..7)，1表示周一(Monday is 1, 1..7)
  %w - 数值周几(0..6)，0表示周日(Sunday is 0, 0..6)

ISO 8601 week-based year and week number:
The week 1 of YYYY starts with a Monday and includes YYYY-01-04.
The days in the year before the first week are in the last week of
the previous year.
  %G - The week-based year
  %g - The last 2 digits of the week-based year (00..99)
  %V - Week number of the week-based year (01..53)

Week number:
The week 1 of YYYY starts with a Sunday or Monday (according to %U
or %W).  The days in the year before the first week are in week 0.
  %U - Week number of the year.  The week starts with Sunday.  (00..53)
  %W - Week number of the year.  The week starts with Monday.  (00..53)

Seconds since the Unix Epoch:
  %s - epoch秒，即从1970-01-01 00:00:00 UTC距离现在已经过去的秒数
  %Q - 毫秒epoch

Literal string:
  %n - Newline character (\n)
  %t - Tab character (\t)
  %% - Literal ``%'' character

Combination: 其中%F和%T常用
  %c - date and time (%a %b %e %T %Y)
  %D - Date (%m/%d/%y)
  %F - The ISO 8601 date format (%Y-%m-%d)
  %v - VMS date (%e-%b-%Y)
  %x - Same as %D
  %X - Same as %T
  %r - 12-hour time (%I:%M:%S %p)
  %R - 24-hour time (%H:%M)
  %T - 24-hour time (%H:%M:%S)
  %+ - date(1) (%a %b %e %H:%M:%S %Z %Y)
```

## 查看日期时间各部分信息

比如查看一个日期中的年份、月份、分钟、秒数、时区等信息。

对于Date和DateTime对象，提供了如下几个查看各部分信息的方法：
- year
- month或mon
- day
- hour
- minute或min
- second或sec
- sec_fraction或second_fraction：查看小数秒(结果以分数显示)
- zone：查看时区
- cweek：查看当前星期是一年的第几个星期(1-53)
- yday：查看年中天(当前日期在一年中的第几天，1-366)
- mday：查看月中天(1-31)
- wday：查看周中天(一周中的第几天，0-6，0代表周日，6代表周六)
- day_fraction：查看一天以过去多少，以分数表示，例如中午12点表示过去1/2，看下面示例
- monday?：是周一吗？
- tuesday?：是周二吗？
- wednesday?:是周三吗？
- thursday?：是周四吗？
- friday?：是周五吗？
- saturday?：是周六吗？
- sunday?：是周日吗？
- leap?：是闰年吗？

对于Time对象来说，除了上面几个方法外(但不支持sec_fraction/second_fraction)，它还支持直接查看毫秒、微妙、纳秒，即将小数秒转换成对应的单位数值：
- subsec：等价于sec_fraction/second_fraction，即以分数的方式返回小数秒
- usec：毫秒数
- nsec：纳秒数


```ruby
# DateTime
>> dt = DateTime.new(2009,7,12,16,32,40.00123)
>> dt.year           #=> 2009
>> dt.mon            #=> 7
>> dt.day            #=> 12
>> dt.mday           #=> 12
>> dt.cweek          #=> 28
>> dt.hour           #=> 16
>> dt.min            #=> 32
>> dt.sec            #=> 40
>> dt.sec_fraction   #=> (123/100000)
>> dt.yday           #=> 193
>> dt.wday           #=> 0  周日
>> dt.sunday?        #=> true

# DateTime: day_fraction
>> DateTime.new(2009,7,12).day_fraction
=> (0/1)
>> DateTime.new(2009,7,12,12).day_fraction
=> (1/2)
>> DateTime.new(2009,7,12,16,32,40).day_fraction
=> (1489/2160)
>> DateTime.new(2009,7,12,23,59,59).day_fraction
=> (86399/86400)
>> DateTime.new(2009,7,12,16,32,40.00123).day_fraction
=> (5956000123/8640000000)

# Time
>> t = Time.parse('2009-07-12 16:32:40.00123')
>> t.year      #=> 2009
>> t.subsec    #=> (123/100000)
>> t.usec      #=> 1230
>> t.nsec      #=> 1230000
```

## 日期时间的运算

这是比较常见的需求，比如加7天之后的日期，10天前的日期等等。不过，对于Ruby来说，这些都很简单，因为它已经实现好了相关的加减法运算符以及一些相关的方法，非常方便。

### 日期时间加减法运算

Date/DateTime/Time都实现了`+ -`操作，它们都返回新的日期时间对象：
- 对于Time来说分别用于增加、减少秒数(可以是小数)
- 对于Date/DateTime来说分别用于增加、减少天数(可以是小数)

下面是Date/DateTime类对象使用加减法进行日期运算的示例：
```ruby
>> d = Date.parse('2019-02-26')
>> dt = DateTime.parse('2019-02-26 12:30:30')
>> d + 1
=> #<Date: 2019-02-27 ((2458542j,0s,0n),+0s,2299161j)>
>> d + 3
=> #<Date: 2019-03-01 ((2458544j,0s,0n),+0s,2299161j)>
>> d + 3 - 3
=> #<Date: 2019-02-26 ((2458541j,0s,0n),+0s,2299161j)>

>> (dt + 3).to_s
=> "2019-03-01T12:30:30+00:00"
>> (dt + 2.5).to_s
=> "2019-03-01T00:30:30+00:00"
```

下面是Time类对象使用加减法进行时间运算的示例，注意秒运算可以是小数：
```ruby
>> t = Time.new(2019,2,26,12,30,30)
=> 2019-02-26 12:30:30 +0800
>> t + 20         #=> 2019-02-26 12:30:50 +0800
>> t + 86400      #=> 2019-02-27 12:30:30 +0800
>> t + 86400 * 3  #=> 2019-03-01 12:30:30 +0800
>> (t + 10.32).nsec  #=> 320000000
```

### 月份运算

对于Date/DateTime类来说，还提供了月份运算的功能`<< >>`：
- `<<`表示前几个月，可以给负数来表示后几个月
- `>>`表示后几个月，可以给负数来表示前几个月

```ruby
>> dt.to_s       # 2月26
=> "2019-02-26T12:30:30+00:00"

>> (dt << 1).to_s  # 1月26
=> "2019-01-26T12:30:30+00:00"
>> (dt >> -1).to_s
=> "2019-01-26T12:30:30+00:00"

>> (dt << -1).to_s  # 3月26
=> "2019-03-26T12:30:30+00:00"
>> (dt >> 1).to_s
=> "2019-03-26T12:30:30+00:00"
```

但是，月份操作需要注意，有些月份的最后一天值是不一样的，比如3月份最后一天是31日，向前移1个月是2月，2月最后一天可能是28日，也可能是29日。而对于月份操作来说，当运算后的月份的天数超出了该月范围时，将自动取该月最后一天。

```ruby
>> d = Date.new(2019,3,31)

>> (d << 1).to_s
=> "2019-02-28"
```

这样可能会导致一些意料之外的运算结果。例如，3月31号前移两个月本是1月31号，但是通过两次前移1个月，得到的将是1月28或1月29。
```ruby
>> d = Date.new(2019,3,31)
>> (d << 2).to_s       #=> "2019-01-31"
>> (d << 1 << 1).to_s  #=> "2019-01-28"

>> (d << 1 << -1).to_s #=> "2019-03-28"
```

所以，使用`<< >>`来做月份运算是不安全的，如果要保证安全，还是尽量使用日期时间的加减法进行运算。

### 日期时间的prev和next

除了`+ - << >>`这几个运算符，对于Date/DateTime来说还支持next/prev等一些操作：
- next或succ
- next_day
- next_month
- next_year
- prev_day
- prev_month
- prev_year

当然，这些都能通过前面介绍的`+ - << >>`来实现等价的。而且，对于month和year的操作，同样有不安全的问题，参见上面对`<< >>`的介绍。


## 日期时间的比较

Date/Time/DateTime都实现了`<=>`运算符，而且它们都mix-in了Comparable，所以可以直接进行大小比较，还可以使用between?这样的方法来判断某个时间点是否在时间范围内。这是非常实用方便的功能。

此外，Date实现了`===`运算符，它等价于`==`，所以只要日期相同，就返回true，而DateTime是Date的子类，所以，也适用于DateTime对象，尽管它们的时间部分可能不一样。所以，Date和DateTime对象之间可以互相比较，但它们都不能直接于Time对象进行比较。

```ruby
>> d1 = Date.new(2019,5,23)
>> d2 = Date.new(2019,5,24)
>> d1 < d2  #=> true

>> dt1 = DateTime.new(2019,5,23,12,30,30.123)
>> dt2 = DateTime.new(2019,5,23,12,30,30.234)
>> dt1 < dt2    #=> true
>> dt1 === dt2  #=> true，尽管时间不一样，但结果true
>> d1 === dt1   #=> true

>> t1 = Time.new(2019,5,23,12,30,30.123)
>> t2 = Time.new(2019,5,23,12,30,30.234)
>> t1 === t2    #=> false
>> t1 == t2     #=> false
>> t1 < t2      #=> true
```

## 日期时间的迭代

Date/DateTime还支持`downto`和`upto`两种方式的日期迭代(Time不支持)，默认每次迭代一天。

此外，还支持step迭代，它可以指定迭代时的步长。

```ruby
>> d.to_s      #=> "2019-03-31"
>> (d+7).to_s  #=> "2019-04-07"
>> d.upto(d + 7) {|date| puts date}
2019-03-31
2019-04-01
2019-04-02
2019-04-03
2019-04-04
2019-04-05
2019-04-06
2019-04-07

>> (d + 7).downto(d) {|date| puts date}
2019-04-07
2019-04-06
2019-04-05
2019-04-04
2019-04-03
2019-04-02
2019-04-01
2019-03-31
```

对于step来说：
```ruby
step(limit[, step=1]) → enumerator
step(limit[, step=1]){|date| ...} → self
```

唯一需要注意的是，step语句块返回的是原始对象，所以在语句块中应当做出一些有意义的操作，并且不依赖于语句块来构建返回值。

简单的用法如下：
```ruby
d = Date.new(2019, 5, 23)
d1 = Date.new(2019, 5, 28)
d.step(d1) { |date| puts date }
puts "-" * 20
d.step(d1, 2) { |date| puts date }
```

输出结果：
```
2019-05-23
2019-05-24
2019-05-25
2019-05-26
2019-05-27
2019-05-28
--------------------
2019-05-23
2019-05-25
2019-05-27
```

## 几种构建日期时间对象的效率对比

Date/DateTime/Time这几个类都有几种构建日期时间对象的方式，但不同的构建方式，其性能肯定是有差别的。

下面测试这几种方式构建100W个日期时间对象，来对比下它们的性能。

此处先说明结论：
- 根据测试结果，使用new方法构建日期时间对象几乎总是最佳选择
- 手动指定时区的效率要比自动设置时区差，但Time.new除外，Time.new指定时区后效率提高几倍
- parse方法的效率最差，而且差很多很多

### Date类的几种构建方式

Date只能构建日期对象，不包含时间，所以在某些场景下不太适合。

下面是构建100W个日期对象几种方式的效率对比，从结果中可以看出，Date.new效率是最高的，Date.parse是最差的。

```bash
## Date.new
$ time ruby -r'date' -e '1000000.times {|x| Date.new(2017,3,23)}'
real    0m0.410s
user    0m0.203s
sys     0m0.203s

## Date.parse
$ time ruby -r'date' -e '1000000.times {|x| Date.parse("2017-3-23")}'
real    0m2.604s
user    0m2.406s
sys     0m0.219s

## Date.strptime
$ time ruby -r'date' -e '1000000.times {|x| Date.strptime("2017-3-23","%Y-%m-%d")}'
real    0m0.790s
user    0m0.609s
sys     0m0.203s
```

### Time类的几种构建方式

Time类构建100W个日期时间对象的几种方式效率对比。从结果中可看出：
- Time.at构建效率是最好的，但是它只能转换epoch时间戳
- 其它通用的几种构建方式，Time.new效率最佳，特别是手动指定时区时，几乎能比肩Time.at
- Time.parse效率是最差的，而且差很多

```bash
# Time类
## Time.new，不指定时区
$ time ruby -r'time' -e '1000000.times {|x| Time.new(2017,3,23,16,30,15)}'
real    0m2.546s
user    0m0.969s
sys     0m1.578s

## Time.new，指定时区
$ time ruby -r'time' -e '1000000.times {|x| Time.new(2017,3,23,16,30,15,"+08:00")}'
real    0m0.663s
user    0m0.453s
sys     0m0.219s

## Time.at，不指定时区
$ time ruby -e '1000000.times {|x| Time.at(1490257815)}'
real    0m0.358s
user    0m0.141s
sys     0m0.219s

## Time.at，指定时区
$ time ruby -e '1000000.times {|x| Time.at(1490257815,in: "+08:00")}'
real    0m0.941s
user    0m0.734s
sys     0m0.219s

## Time.parse，指定时区
$ time ruby -r'time' -e '1000000.times {|x| Time.parse("2017-03-23 16:30:15 +0800")}'
real    0m10.949s
user    0m10.422s
sys     0m0.531s

## Time.parse，不指定时区
$ time ruby -r'time' -e '1000000.times {|x| Time.parse("2017-03-23 16:30:15")}'
real    0m10.972s
user    0m8.953s
sys     0m1.984s

## Time.mktime，指定时区
$ time ruby -r'time' -e '1000000.times {|x| Time.mktime(2017,3,23,16,30,15,"+08:00")}'
real    0m2.575s
user    0m0.984s
sys     0m1.578s

## Time.mktime，不指定时区
$ time ruby -r'time' -e '1000000.times {|x| Time.mktime(2017,3,23,16,30,15)}'
real    0m2.490s
user    0m0.984s
sys     0m1.531s
```

### DateTime类的几种构建方式

DateTime类构建100W个日期时间对象的几种方式效率对比。从结果中可看出：
- DateTime.new构建效率是最好的，指定时区与否几乎无差别
- DateTime.parse效率是最差的，而且差很多

```bash
## DateTime.new：不指定时区
$ time ruby -r'date' -e '1000000.times {|x| DateTime.new(2017,3,23,16,30,15)}'
real    0m0.409s
user    0m0.219s
sys     0m0.203s

## DateTime.new：指定时区
$ time ruby -r'date' -e '1000000.times {|x| DateTime.new(2017,3,23,16,30,15,"+08:00")}'
real    0m0.502s
user    0m0.328s
sys     0m0.188s

## DateTime.strptime：不指定时区
$ time ruby -r'date' -e '1000000.times {|x| DateTime.strptime("2017-3-23 16:30:15","%Y-%m-%d %H:%M:%S")}'
real    0m0.994s
user    0m0.797s
sys     0m0.219s

## DateTime.strptime：指定时区
$ time ruby -r'date' -e '1000000.times {|x| DateTime.strptime("2017-3-23 16:30:15 +08:00","%Y-%m-%d %H:%M:%S %z")}
'
real    0m1.838s
user    0m1.656s
sys     0m0.203s

## DateTime.parse：不指定时区
$ time ruby -r'date' -e '1000000.times {|x| DateTime.parse("2017-3-23 16:30:15")}'
real    0m6.206s
user    0m5.984s
sys     0m0.203s

## DateTime.parse：指定时区
$ time ruby -r'date' -e '1000000.times {|x| DateTime.parse("2017-3-23 16:30:15 +08:00")}'
real    0m6.944s
user    0m6.734s
sys     0m0.203s
```


