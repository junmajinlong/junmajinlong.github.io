---
title: 效率对比：构造100W个时间对象
p: coding/lang_performance.md
date: 2019-07-06 18:20:41
tags: Coding
categories: Coding
---

原本是用perl写了一个通过给定的时间范围来筛选一个比较大的日志文件。但是测试发现筛选130W行日志需要2分多钟，对其中几个低效率函数单独进行了效率测试，发现构造100W个时间对象所花时间也是个大户。

于是，特地比较了几种语言构造100W个时间对象(或时间结构)的性能。以下是结论：
- Perl(Time::Piece模块)：13秒  
- Python(time模块)：8.8秒(arrow包：50秒)  
- Ruby(内置Time类)：2.8秒，直接指定时区则只需0.6秒  
- **Ruby(标准库DateTime)：strptime格式化需0.9-1秒，直接DateTime.new则只需0.4秒**  
- **Golang(Time包)：编译好的0.17秒，编译+执行0.26秒**  
- C(time.h)：0.042秒  

下面是各语言测试代码：

```shell
# Perl(Time::Piece)
$ time perl -MTime::Piece -e 'for(1..1000000){$t = Time::Piece->strptime("2017-03-23 16:30:15", "%Y-%m-%d %H:%M:%S")}'
real    0m13.045s
user    0m11.969s
sys     0m0.011s
```


```shell
# Ruby(Time)
$ time ruby -e '1000000.times {|x| t=Time.new(2017,3,23,16,30,15)}'
real    0m2.755s
user    0m1.781s
sys     0m0.767s

$ time ruby -e '1000000.times {|x| t=Time.new(2017,3,23,16,30,15,"+08:00")}'

real    0m0.685s
user    0m0.500s
sys     0m0.203s
```

```shell
# Ruby(Datetime)
$ time ruby -r'date' -e '1000000.times {|x| t = DateTime.strptime("2017-03-23 16:30:15", "%Y-%m-%d %H:%M:%S")}'
real    0m0.994s
user    0m0.885s
sys     0m0.036s

$ time ruby -r'date' -e '1000000.times {|x| t = DateTime.new(2017,3,23,16,30,15)}'
real    0m0.450s
user    0m0.234s
sys     0m0.219s
```

![](/img/referer.jpg)

```shell
# Python
import time

fmt = "%Y-%m-%d %H:%M:%S"
for i in range(0,1000000):
        time.strptime("2017-10-23 12:30:23", fmt)

$ time python3 a.py
real    0m8.411s
user    0m7.805s
sys     0m0.094s
```

```shell
# Golang
package main

import (
        "time"
)

func main() {
        const format = "2006-01-02 15:04:05"
        for i := 1; i < 1000000;i++ {
                time.Parse(format, "2018-03-23 16:30:15")
        }
}

$ time go run a.go       # 编译加执行的go程序

real    0m0.267s
user    0m0.213s
sys     0m0.070s

$ time ./a         # 编译过后的go程序

real    0m0.176s
user    0m0.162s
sys     0m0.001s
```

```shell
// C
#define _XOPEN_SOURCE
#include <time.h>

int main(void) {
        struct tm tm;
        int i;
        for(i = 1;i<=1000000;++i){
                strptime("2017-03-23 16:30:15", "%Y-%m-%d %H:%M:%S", &tm);
        }
        return 0;
}

$ time ./time.out 

real    0m0.042s
user    0m0.039s
sys     0m0.001s
```

![](/img/referer.jpg)


那么，对于写脚本的话，采用哪门语言呢？我想Ruby是很不错的选择，Golang也是不错的选择。

后来我又对比了一下perl/python/ruby在单纯的循环上的效率，算是解释型语言在纯计算上的优劣，结果python差的很远，而ruby又再次领先。
```shell
# perl循环一亿次
$ time perl -e '$a=0;$i=0;for(1..100000000){$a+=1;}'
real    0m2.688s
user    0m2.672s
sys     0m0.016s

# python循环一亿次
[root]$ time python <<eof
a=0
i=0
for i in range(100000000):
  a+=1

eof

real    0m6.682s
user    0m4.594s
sys     0m2.094s

# ruby以range的方式循环一亿次
[root]$ time ruby -e 'a=0;for i in (1..100000000) do a+=1;end'
real    0m3.778s
user    0m3.531s
sys     0m0.266s

# ruby以while条件判断的方式循环一亿次
[root]$ time ruby -e 'a=0;i=0;while(i<100000000) do a+=1;i+=1 end'
real    0m1.728s
user    0m1.516s
sys     0m0.234s
```