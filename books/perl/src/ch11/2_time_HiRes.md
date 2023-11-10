## Time::HiRes模块

Perl内置的time、sleep函数都只能精确到秒，`Time::HiRes`模块提供了更高精度的相关函数，此外还提供了stat、utime、lstat等函数来提供更高精度的文件属性。

这个模块无需安装，已经内置成为标准模块，但如果要使用里面提供的函数，则需要手动导入。例如：

```perl
use Time::HiRes qw(utime usleep sleep gettimeofday);
```

下面介绍该模块提供的几个常用函数。

1. `gettimeofday`：获取当前时间点，在标量上下文返回epoch秒和微秒组成的浮点数，在列表上下文返回epoch秒和微秒两个元素。

   ```perl
   use Time::HiRes qw(gettimeofday);
   # 标量上下文
   my $t0 = gettimeofday;
   say $t0;    # 1625732664.42617
   
   # 列表上下文
   say "@{[gettimeofday]}"; # 1625732664 426361
   
   # 可用来衡量某些操作耗时多久，可精确到微秒级
   my $start = gettimeofday;
   ...do something...
   say getimeofday - $start;
   ```

2. `time`：获取当前时间点，返回精确到微秒的浮点数。即等价于标量上下文的`gettimeofday`。  

   ```perl
   use Time::HiRes qw(time);  # 覆盖默认的内置time函数
   say time;        # 1625732928.18472
   ```

3. `sleep`和`usleep`和`nanosleep `：提供可精确到毫秒、微秒、纳秒的睡眠行为。

   ```perl
   use Time::HiRes qw(sleep usleep nanosleep);  # 覆盖默认的内置sleep函数
   sleep 3.2;     # 睡眠3.2秒
   usleep 20000;  # 睡眠0.2秒(20000微秒)
   nanosleep 1000;#睡眠1000纳秒，即1微秒
   nanosleep 0;  # 效果类似于Thread yield，即直接放弃cpu等待下次调度
   ```

4. `tv_interval`：计算两个时间点的时间差，主要用来计算gettimeofday返回的值的时间差，接收的两个参数均是数组的引用。

   ```perl
   use Time::HiRes qw(tv_interval gettimeofday);
   my $t0 = [gettimeofday];
   # do bunch of stuff here
   my $t1 = [gettimeofday];
   # do more stuff here
   say tv_interval $t0, $t1;
   ```





