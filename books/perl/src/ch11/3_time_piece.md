## Time::Piece模块

`Time::Piece`提供了一些操作日期时间的方法，用法非常简单，也已经内置到标准库中。如果有Time::Piece不方便解决或无法解决的更复杂的需求，可使用`DateTime`模块。

> 注：某些32位计算机中，`Time::Piece`最大只能支持2038年，如果需要处理2038年之后的时间，使用DateTime模块。另外，64位则无此限制。

导入`Time::Piece`模块之后，内置的`localtime`和`gmtime`函数将被覆盖，它们将返回一个`Time::Piece`对象。此外，`Time::Piece`自身也提供了一个new方法创建`Time::Piece`对象。

```perl
use Time::Piece;
my $t = localtime;
my $t1 = Time::Piece->new;
my $t2 = Time::Piece->new(1625732928);
```

得到`Time::Piece`对象后，可以使用一大堆的和日期时间相关的方法：

```perl
$t->sec                 # 获取秒，别名：$t->second
$t->min                 # 获取分钟，别名：$t->minute
$t->hour                # 获取小时(24小时制)
$t->mday                # 获取天，即几月几号的几号，别名：$t->day_of_month
$t->mon                 # 获取月份，1代表一月
$t->_mon                # 获取月份，0代表一月
$t->monname             # 获取月份简称，如二月是Feb
$t->month               # 同上，也是获取月份简称
$t->fullmonth           # 获取月份全称，如二月是February
$t->year                # 获取4位数的年份
$t->_year               # 获取从1900年开始算的年份，如2000年时的值是100
$t->yy                  # 获取2位数的年份
$t->wday                # 获取周几，1代表周日
$t->_wday               # 获取周几，0代表周日
$t->day_of_week         # 同上，获取周几，0代表周日
$t->wdayname            # 获取周几简称，如Tue代表周二
$t->day                 # 同上
$t->fullday             # 获取周几全称，如Tuesday代表周二
$t->yday                # 获取一年中的第几天，0代表1月1号，别名：$t->day_of_year
$t->isdst               # 是否是夏令时，别名：$t->daylight_savings

$t->hms                 # 获取小时分钟秒，格式`12:34:56`
$t->hms(".")            # 获取小时分钟秒，格式`12.34.56`
$t->time                # 同$t->hms

$t->ymd                 # 2000-02-29
$t->date                # 同$t->ymd
$t->mdy                 # 02-29-2000
$t->mdy("/")            # 02/29/2000
$t->dmy                 # 29-02-2000
$t->dmy(".")            # 29.02.2000
$t->datetime            # 2000-02-29T12:34:56 (ISO 8601)
$t->cdate               # Tue Feb 29 12:34:56 2000
"$t"                    # same as $t->cdate

$t->epoch               # epoch
$t->tzoffset            # 时区偏移值，timezone offset in a Time::Seconds object

$t->week                # 一年中的第几周(ISO 8601)

$t->is_leap_year        # 是否是闰年
$t->month_last_day      # 本月的最后一天是几号，值范围：28-31
 
$t->time_separator($s)  # 设置时间分隔符，默认":"
$t->date_separator($s)  # 设置日期分隔符，默认"-"
$t->day_list(@days)     # 设置周几的默认显示值，周日应作为列表的第一个元素
$t->mon_list(@days)     # 设置月份的默认显示值，一月应作为列表的第一个元素
 
$t->strftime(FORMAT)    # 将日期时间格式化为字符串
$t->strftime()          # "Tue, 29 Feb 2000 12:34:56 GMT"
 
$t->truncate   # 将日期时间截断到指定单位处。修改的是拷贝的对象。
               # 例如$t->truncate(to => 'day')将使时间部分全部为0
               # 支持的截断目标：year, quarter, month, day, hour, minute和second 

# 根据给定格式，将字符串解析为日期时间对象
Time::Piece->strptime(STRING, FORMAT)
```

例如：

```perl
use Time::Piece;

# 2021-07-08 16:28:48
my $t = Time::Piece->new(1625732928);

say $t->year;     # 2021
say $t->hms;      # 16:28:48
say $t->ymd;      # 2021-07-08
say $t->epoch;    # 1625732928

# 设置周和月的默认显示名称
Time::Piece::day_list(qw(周日 周一 周二 周三 周四 周五 周六));
Time::Piece::mon_list(qw(一月 二月 三月 四月 五月 六月 七月 八月 九月 十月 十一月 十二月));
say $t->wdayname();    # 周四
say $t->monname();     # 七月

# 将日期时间格式为字符串
say $t->strftime("%F %T");  # 2021-07-08 16:28:48

# 将字符串根据指定格式解析为日期时间对象
my $t1 = Time::Piece->strptime(1625732928, "%s");
my $t2 = Time::Piece->strptime("2021-07-08 16:28:48", "%F %T");
say $t1->datetime;   # 2021-07-08T08:28:48
say $t2->datetime;   # 2021-07-08T16:28:48

my $t3 = $t2->truncate(to => 'day');
say $t3->datetime;   # 2021-07-08T00:00:00
```

`Time::Piece`对象还能进行一些算术运算(仅支持加减秒、月和年以及计算时间差)以及进行大小比较(`< > <= >= <=> == !=`)。

例如：

```perl
# 加减秒、月、年
$t1 - 42;  # returns Time::Piece object
$t1 + 533; # returns Time::Piece object
$t = $t->add_months(6);
$t = $t->add_years(5);

# 计算时间差
$t1 - $t2; # returns Time::Seconds object
```

注意，计算时间差时返回的是`Time::Seconds`对象，`Time::Seconds`模块提供了一些秒转换的时间常量(如常量`ONE_DAY`代表一天的秒数)，以及几个简单的时间转换方法(如60秒转换为1分钟)。

```perl
# Time::Seconds提供的常量
ONE_DAY
ONE_WEEK
ONE_HOUR
ONE_MINUTE
ONE_MONTH
ONE_YEAR
ONE_FINANCIAL_MONTH
LEAP_YEAR
NON_LEAP_YEAR

# Time::Seconds对象的方法
my $val = Time::Seconds->new(360);  # 360秒
$val->seconds;  # 360
$val->minutes;  # 6
$val->hours;    # 0.1
$val->days;     # 0.00416666666666667
$val->weeks;    # 0.000595238095238095
$val->months;   # 0.00013689544435102
$val->financial_months; # 固定30天的月，0.000138888888888889
$val->years;
$val->pretty;   # 6 minutes, 0 seconds
```

通过`Time::Seconds`提供的常量，可以轻松地对`Time::Piece`对象加减任意单位的时间。例如：

```perl
use Time::Piece;
use Time::Seconds;

# 三天后的时间
my $t = localtime;
my $t1 = $t + 3 * ONE_DAY;
say $t1->datetime;
```

