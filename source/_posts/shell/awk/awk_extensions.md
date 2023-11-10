---
title: 精通awk系列(29)：awk几个实用的扩展插件
p: shell/awk/awk_extensions.md
date: 2020-04-12 15:54:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# 几个常见的gawk扩展

使用扩展的方式：
```
awk -l ext_name 'BEGIN{}{}END{}'
awk '@load "ext_name";BEGIN{}{}END{}'
```

## 1.文件相关的扩展

awk和文件相关的扩展是"filefuncs"。

它支持chdir()、stat()函数。

## 2.awk文件名匹配扩展

"fnmatch"扩展提供文件名通配。

```
@load "fnmatch"
result = fnmatch(pattern, string, flags)
```

## 3.awk原处修改文件

awk通过加载inplace.awk，也可以实现`sed -i`类似的功能，即内容直接修改源文件(其本质是先写入临时文件，写完后将临时文件重命名为源文件进行覆盖)。

例如：

```
# 从源文件中直接去掉包含strict的行
awk -i inplace '!/strict/{print}' x.txt
awk '@include "inplace";!/strict/{print}' x.txt

# 禁用原处替换，inplace默认值为1。
# inplace应当在BEGIN中设置，而非main中设置，因为这时已开始读文件
awk -i inplace 'BEGIN{inplce=0}!/strict/{print}' x.txt
awk '@include "inplace";BEGIN{inplce=0}!/strict/{print}' x.txt

# 指定后缀名进行备份，下面将生成x.txt和x.txt.bak文件，x.txt.bak是原文内容
awk -i inplace 'BEGIN{INPLACE_SUFFIX=".bak"}!/strict/{print}' x.txt
awk '@include "inplace";BEGIN{INPLACE_SUFFIX=".bak"}!/strict/{print}' x.txt
```

## 4.awk多进程扩展

"fork"扩展提供多进程相关功能。

```
@load "fork"

pid = fork()
创建一个子进程，对子进程返回值为0，对父进程返回值为子进程的PID，返回-1表示错误。
在子进程中，PROCINFO["pid"]和PROCINFO["ppid"]会随之更新。

ret = waitpid(pid)
等待某个子进程退出。awk的waitpid是非阻塞的，如果等待的进程还未退出，则返回值为0，等待的进程已经退出，则返回该进程pid。

ret = wait()
等待任意一个子进程退出。wait()是阻塞的，必须等待到一个子进程退出，同时返回该子进程PID。
```

例如：
```
awk '
    @load "fork"
    BEGIN{
        if( (pid=fork()) == 0 ){
            print "Child Process"
            print "CHILD PID: "PROCINFO["pid"]
            print "CHILD PPID: "PROCINFO["ppid"]
            system("sleep 1")
        } else {
            while(waitpid(pid) == 0){
                system("sleep 1")
            }
            print "Parent PID: "PROCINFO["pid"]
            print "Parent PPID: "PROCINFO["ppid"]
            print "Parent Process"
        }
    }
'
```

## 5.awk日期时间扩展

"time"扩展提供了两个函数。

```
@load "time"

the_time = gettimeofday()
    获取当前系统时间，以浮点数方式返回，精确的浮点小数位由操作系统决定  

res = sleep(sec)
    睡眠指定时间，可以是小数秒
```

```
$ awk '@load "time";BEGIN{printf "%.9f\n",gettimeofday()}'
1572422333.740148067

$ awk '@load "time";BEGIN{printf "%.19f\n",gettimeofday()}'
1572422391.5475890636444091797
```

睡眠是很好用的功能：
```
awk '@load "time";BEGIN{sleep(1.2);print "hello world"}'
```
