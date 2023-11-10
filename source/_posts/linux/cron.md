---
title: Linux定时任务cron
p: linux/cron.md
date: 2019-07-07 11:29:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux定时任务cron

Linux中可以通过cron和systemd来定义定时任务。本文介绍cron的配置方式，关于systemd的定时任务，参考：[systemd timer：取代cron和at的定时任务](https://www.junmajinlong.com/linux/systemd/systemd_timer)。

## 配置cron定时任务

一个不错的定时任务在线测试网站：<http://www.atool9.com/crontab.php>。

关于定时任务，首先需弄清的概念：

(1).crond是一个daemon类程序，路径为`/usr/sbin/crond`。默认会以后台方式启动，service或systemd方式启动crond默认也是后台方式的。

(2).crondtab是管理crontab file的工具，而crontab file是定义定时任务条目的文件。

(3).crontab file存在于多处，包括系统定时任务文件`/etc/crontab`和`/etc/cron.d/\*`，还有独属于各用户的任务文件`/var/spool/cron/USERNAME`。

再就是crontab命令：

```
-l：列出定时任务条目
-r：删除当前任务列表终端所有任务条目
-i：删除条目时提示是否真的要删除
-e：编辑定时任务文件，实际上编辑的是/var/spool/cron/*文件
-u：操作指定用户的定时任务
```

执行`crontab -e`命令编辑当前用户的crontab file，例如当前为root用户，则编辑的是`/var/spool/cron/root`文件。例如写入下面这一行。

```
* * * * * /bin/echo "the first cron entry"  >>/tmp/crond.txt
```

这将会每分钟执行一次echo命令，将内容追加到`/tmp/crond.txt`文件中。

任务计划中的任务条目如何定义，可以查看`/etc/crontab`文件。

```
[root@server2 ~]# cat /etc/crontab
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
```

在此文件中定义了3个变量，其中一个是PATH，该变量极其重要。在最后还给出了任务条目的定义方式：

- (1).每个任务条目分为6段，每段以空格分隔，之所以此处多了user-name段是因为`/etc/crontab`为系统定时任务文件，而一般定时任务是没有该段的。

- (2).前五段为时间的设定段，分别表示"分时日月周"，它们的定义不能超出合理值范围，第六段为所要执行的命令或脚本任务段。

- (3).在时间定义段中，使用`*`表示每单位，即每分钟，每小时，每天，每月，每周几(仍然是每天)。实际上，按man文档中解释，`*`表示的是从每个时间段的起始到结尾，也就是全部时间单位的意思。例如在小时上设置`*`，表示`0,1,2,3...22,23`的意思。

- (4).每个时间段中，都可以使用逗号`,`来表示枚举，例如定义`0,30,50 \* \* \* \*`表示每个时辰的整点、第30分钟和第50分钟都执行该任务。

- (5).每个时间段中，都可以使用`-`定义范围，可以结合逗号使用。如分钟段定义了`00,20-30,50`表示每个时辰的整点、第20到30分钟的每分钟、第50分钟都执行该任务。

- (6).每个时间段中，使用`/`表示忽略时间，如在小时段定义了`0-13/2`表示在`0/2/4/6/8/10/12`点才满足时间定义。常使用`*/N`表示每隔多久的意思。例如`00 */2 * * *`表示在每天每隔两小时的整点执行该任务(严格地说是`0-23/2`，也就是`0,2,4,...,22`，所以凌晨1点不会执行任务)。

- (7).如果定义的日和周冲突了，则会多次执行(不包括因为`*`号导致的冲突)。例如每月的15号执行该任务，同时又定义了周三执行该任务，正常无冲突情况下，将在周三和每月15号执行，但如果某月的15号同时是周三，则该任务在此日执行两次。因此，应该尽力避免同时定义周和日的任务。

- (8).命令段(即第6段)中，不能随意出现百分号`%`，因为它表示换行的特殊意义，且第一个%后的所有字符串将当作命令的标准输入。

例如下面的定义：

```
* * * * * /bin/cat >>/tmp/crond.txt %"the first %%cron entry%"
```

该任务输出的结果将是：

```
"the first

cron entry
"
```

所以，在定时任务条目中若以时间定义文件名时，应当将`%`使用反斜杠转义。如：

```
* * * * * cp /etc/fstab /tmp/`date +\%Y-\%m-\%d`.txt
```

另外一个需要注意的时间段设置是，使用\*号问题。例如`* */2 * * *`，它表示每隔两小时后的每一分钟都执行任务，也就是凌晨0点的每分钟执行任务，凌晨1点不执行任务，凌晨2点的每分钟执行任务，凌晨4点的每分钟执行任务，依此类推。同理，`*/5 */2 * * *`表示每隔2小时后的每5分钟执行一次任务。

## crondtab file 

crondtab file为任务定义文件。

- (1).在此文件中，空行会被忽略，首个非空白字符且以`#`开头的行为注释行，但`#`不能出现在行中。

- (2).可以在crontab file中设置环境变量，方式为`name=value`，等号两边的空格可随意，即`name = value`也是允许的。但value中出现的空格必须使用引号包围。

- (3). 默认crond命令启动的时候会初始化所有变量，除了某几个变量会被crond daemon自动设置好，其他所有变量都被设置为空值。自动设置的变量包括`SHELL=/bin/sh`，以及HOME和LOGNAME(在CentOS上则称为USER)，后两者将被默认设置为`/etc/passwd`中指定的值。其中SHELL和HOME可以被crontab file中自定义的变量覆盖，但LOGNAME不允许覆盖。当然，自行定义的变量也会被加载到内存。

- (4).除了LOGNAME/HOME/SHELL变量之外，如果设置了发送邮件，则crond还会寻找MAILTO变量。如果设置了MAILTO，则邮件将发送给此变量指定的地址，如果MAILTO定义的值为空(`MAILTO=""`)，将不发送邮件，其他所有情况邮件都会发送给crontab file的所有者。

- (5).在系统定时任务文件/etc/crontab中，默认已定义PATH环境变量和SHELL环境变量，其中`PATH=/sbin:/bin:/usr/sbin:/usr/bin`。

- (6).crond daemon每分钟检测一次crontab file看是否有任务计划条目需要执行。

## crond命令的调试 

很多时候写了定时任务却发现没有执行，或者执行失败，但因为crond是后台运行的，有没有任何提示，很难进行排错。但是可以让crond运行在前端并进行调试的。

先说明下任务计划程序crond的默认执行方式。

使用下面三条命令启动的crond都是在后台运行的，且都不依赖于终端。

```
[root@xuexi ~]# systemctl start crond.service
[root@xuexi ~]# service crond start
[root@xuexi ~]# crond
```

但crond是允许接受选项的。

```
crond [-n] [-P] [-x flags]
选项说明：
-n：让crond以前端方式运行，即不依赖于终端。
-P：不重设环境变量PATH，而是从父进程中继承。
-x：设置调试项，flags是调试方式，比较有用的方式是test和sch，即"-x test"和"-x sch"。
  ：其中test调试将不会真正的执行，sch调试显示调度信息，可以看到等待时间。具体的见下面的示例。
```

先看看启动脚本启动crond的方式。

```
[root@server2 ~]# cat /lib/systemd/system/crond.service
[Unit]
Description=Command Scheduler
After=auditd.service systemd-user-sessions.service time-sync.target
 
[Service]
EnvironmentFile=/etc/sysconfig/crond
ExecStart=/usr/sbin/crond -n $CRONDARGS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```

它的环境配置文件为`/etc/sysconfig/crond`，该文件中什么也没设置。

```
[root@server2 ~]# cat /etc/sysconfig/crond
# Settings for the CRON daemon.
# CRONDARGS= :  any extra command-line startup arguments for crond
CRONDARGS=
```

所有它的启动命令为：`/usr/sbin/crond -n`。但尽管此处加了`-n`选项，crond也不会前端运行，且不会依赖于终端，这是systemctl决定的。

再解释下如何进行调试。以下面的任务条目为例。

```
[root@server2 ~]# crontab -e
* * * * * echo "hello world" >>/tmp/hello.txt
```

执行crond并带上调试选项test。

```
[root@server2 ~]# crond -x test
debug flags enabled: test
[4903] cron started
log_it: (CRON 4903) INFO (RANDOM_DELAY will be scaled with factor 8% if used.)
log_it: (CRON 4903) INFO (running with inotify support)
log_it: (CRON 4903) INFO (@reboot jobs will be run at computer's startup.)
log_it: (root 4905) CMD (echo "hello world" >>/tmp/hello.txt )
```

执行crond并带上调试选项sch。

```
[root@server2 ~]# crond -x sch
debug flags enabled: sch
[4829] cron started
log_it: (CRON 4829) INFO (RANDOM_DELAY will be scaled with factor 73% if used.)
log_it: (CRON 4829) INFO (running with inotify support)
[4829] GMToff=28800
log_it: (CRON 4829) INFO (@reboot jobs will be run at computer's startup.)
# 等待crond daemon下一次的检测，所以表示38秒后crond将检测crontab file
[4829] Target time=1497950880, sec-to-wait=38      
user [root:0:0:...] cmd="echo "hello world" >>/tmp/hello.txt "
[4829] Target time=1497950940, sec-to-wait=60
Minute-ly job. Recording time 1497922081
log_it: (root 4831) CMD (echo "hello world" >>/tmp/hello.txt )
user [root:0:0:...] cmd="echo "hello world" >>/tmp/hello.txt "
[4829] Target time=1497951000, sec-to-wait=60
Minute-ly job. Recording time 1497922141
log_it: (root 4833) CMD (echo "hello world" >>/tmp/hello.txt )
```

但要注意，在sch调试结果中的等待时间是crond这个daemon的检测时间，所以它表示等待下一次检测的时间，因此除了第一次，之后每次都是60秒，因为默认crond是每分钟检测一次crontab file的。例如，下面是某次的等待结果，在这几次等待检测过程中没有执行任何任务。

```
[4937] Target time=1497951720, sec-to-wait=18
[4937] Target time=1497951780, sec-to-wait=60
[4937] Target time=1497951840, sec-to-wait=60
```

还可以同时带多个调试方式，如：

```
[root@server2 ~]# crond -x test,sch
debug flags enabled: sch test
[4914] cron started
log_it: (CRON 4914) INFO (RANDOM_DELAY will be scaled with factor 21% if used.)
log_it: (CRON 4914) INFO (running with inotify support)
[4914] GMToff=28800
log_it: (CRON 4914) INFO (@reboot jobs will be run at computer's startup.)
[4914] Target time=1497951540, sec-to-wait=9
user [root:0:0:...] cmd="echo "hello world" >>/tmp/hello.txt "
[4914] Target time=1497951600, sec-to-wait=60
Minute-ly job. Recording time 1497922741
log_it: (root 4916) CMD (echo "hello world" >>/tmp/hello.txt )
```

这样在调试定时任务时间时，也不会真正执行命令。

## 精确到秒的任务计划 

默认情况下，crond执行的任务只能精确到分钟，无法精确到秒。但通过技巧，也是能实现秒级任务的。

(1).方法一：不太精确的方法

写一个脚本，在脚本中sleep3秒钟的时间，这样能实现每3秒执行一次命令。

```java
[root@xuexi ~]# cat /tmp/a.sh
#!/bin/bash
#
PATH="$PATH:/usr/local/bin:/usr/local/sbin"
for ((i=1;i<=20;i++));do
ls /tmp
sleep 3
done
```

```java
[root@xuexi ~]# cat /var/spool/cron/lisi
* * * * * /bin/bash /tmp/a.sh
```

但是这样的方法不是最佳方法，因为执行命令也需要时间，且crond默认会有一个随机延时，随机延时由变量RANDOM\_DELAY定义。

(2).方法二：在cron配置文件中写入多条sleep命令和其他命令。

```java
[root@xuexi ~]# cat /var/spool/cron/lisi
* * * * * ls /tmp
* * * * * sleep 3 && ls /tmp
* * * * * sleep 6 && ls /tmp
* * * * * sleep 9 && ls /tmp
* * * * * sleep 12 && ls /tmp
* * * * * sleep 15 && ls /tmp
* * * * * sleep 18 && ls /tmp
* * * * * sleep 21 && ls /tmp
* * * * * sleep 24 && ls /tmp
* * * * * sleep 27 && ls /tmp
* * * * * sleep 30 && ls /tmp
…
* * * * * sleep 57 && ls /tmp
```

这种方式很繁琐，但是更精确。如果定义到每秒级别就得写60行cron记录。

由此能看出，秒级的任务本就不是crond所擅长的。实际上能用到秒级的任务也比较少。