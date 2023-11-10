---
title: shell脚本动画小工具
p: shell/shell_perl_gif.md
date: 2019-07-06 18:20:41
tags: Shell
categories: Shell
---

# shell脚本动画小工具

看gif图：

![](/img/shell/shell.gif)

## shell脚本版

脚本内容如下：
```
#!/usr/bin/env bash

## ------------------------------------------------------------
## Author：博客园——骏马金龙
## shell scripts：https://www.junmajinlong.com/others/shell_perl_gif
## ------------------------------------------------------------

## Usage：$0 "COMMAND"
## you must enclosing the COMMAND by double-quotes
## example1: $0 "sleep 3;echo haha"
## example2: $0 "service mysql start"

killmyself="pkill -13 -f `basename $0`"

trap "$killmyself" sigint

while true;do
    for i in '-' "\\" '|' '/';do
        printf "\r%s" $i
        sleep 0.2
    done
done &

bgpid=$!

tmp="`bash -c \"$@\"`"
kill $bgpid
printf "\r%s\n" "$tmp"
$killmyself
```

必须将待运行的命令放进引号中包围，并作为脚本的参数。
```
## example1: $0 "sleep 3;echo haha"
## example2: $0 "service mysql start"
## example3: $0 "service mysql stop"
```

![](/img/referer.jpg)

## perl版

下面是用perl写的，作用完全一样。将内容保存到一个文件中，赋予可执行权限即可。同样，待执行的命令需要使用双引号包围。
```
#!/usr/bin/env perl
use strict;
use warnings;
#use Time::HiRes qw(sleep);

defined(my $pid = fork) or die "can't fork child: $!";

unless($pid){
    # 子进程
    select STDOUT; $| = 1;
    while(1){
        foreach my $i ('-','\\','|','/'){
            printf("\r%s",$i);
            #sleep(0.1);
            # 有些低版本的perl没有Time::HiRes标准模块，所以使用select
            select(undef,undef,undef,0.1)
        }
    }
}

# 捕获信号SIGINT,放在unless()后面，可以保证不会监控到子进程
$SIG{'INT'}='trap';
sub trap {
        kill INT => $pid or die "Cannot signal to $pid with SIGINT: $!";
        die "";
}

my $var = `/bin/sh -c "@ARGV"`;
kill INT => $pid or die "Cannot signal to $pid with SIGINT: $!";
printf "\r%s",$var;
```

假如该perl文件名为mygif.pl，用法：
```
./mygif "sleep 3;echo haha"
./mygif "service mysql start"
./mygif "service mysql stop"
```