---
title: 如何让Shell脚本自杀
p: shell/kill_script_self.md
date: 2020-03-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# 如何让Shell脚本自杀

<a name="blog1"></a>

## 脚本自杀正文

有些时候我们写的shell脚本中有一些后台任务，当脚本的流程已经执行到结尾处或kill掉脚本时，这些后台任务会直接挂靠在init/systemd进程下，而不会随着脚本退出而停止。

例如：
```
[root@mariadb ~]# cat test1.sh 
#!/bin/bash
echo $BASHPID
sleep 50 &

[root@mariadb ~]# ps -elf | grep slee[p]
0 S root      10806      1  0  80   0 - 26973 hrtime 19:26 pts/1    00:00:00 sleep 50
```
从结果中可以看到，脚本退出后，sleep进程的父进程变为了1，也就是挂在了init/systemd进程下。

这时我们可以在脚本中直接使用kill命令杀掉sleep进程。
```
[root@mariadb ~]# cat test1.sh 
#!/bin/bash
echo $BASHPID
sleep 50 &
kill $!
```
但是，如果这个sleep进程是在循环中，那就麻烦了。

例如下面的例子，直接将循环放入后台，杀掉sleep、或者exit、或者杀掉脚本自身进程、或者让脚本自动退出、甚至exec退出当前脚本shell都是无效的。
```
[root@mariadb ~]# cat test1.sh 
#!/bin/bash
echo $BASHPID

while true;do
    sleep 50
    echo 1
done &

killall sleep
kill $BASHPID
```

为了分析，新建一个脚本test2.sh：
```
#!/bin/bash
echo $BASHPID

while true;do
    sleep 50
    echo 1
done &

sleep 60
```

然后在脚本执行的60秒内查看test2.sh进程的信息：
```
[root@mariadb ~]# pstree -p | grep "test2.sh"
            |        `-bash(2687)---test2.sh(2923)-+-sleep(2925)
            |                                      `-test2.sh(2924)---sleep(2926)
```
其中pid=2923的test2.sh进程是脚本自身进程，pid=2924的test2.sh进程是**while开始运行后为while提供执行环境的子shell进程**(为什么会生成这个进程，后文会给个简单版的补充知识，详细的原理见我的另一篇文章)。

所以，对于前面的test1.sh进程，杀掉了`$BASHPID` 对应的test1.sh进程后，其实还有一个为while提供运行环境的test1.sh进程，且这个进程在`​$BASHPID`结束后，会挂在init/systemd下。

```
[root@mariadb ~]# ./test1.sh 
10859
./test1.sh: line 7: 10862 Terminated              sleep 50
Terminated
1
[root@mariadb ~]# pstree -p | grep sleep
           |-test1.sh(10860)---sleep(10863)
```

这就是**shell脚本中的一个"疑难杂症"，CTRL+C中止了脚本进程，这个脚本却还在后台不断运行，且时不时地输出点信息到终端(我这里是循环中的echo命令输出的)**。

除非我们手动杀掉新生成的test1.sh，否则这个脚本将无限循环下去。但是，这不是很麻烦吗？

那么如何实现"脚本自杀"？其实很简单，只要在脚本退出前，使用killall命令杀掉脚本进程即可。
```
[root@mariadb ~]# cat test1.sh 
#!/bin/bash
echo $BASHPID

while true;do
    sleep 50
    echo 1
done &

killall `basename $0`
```

这样，在脚本退出前，两个test1.sh进程都会被杀掉。

再考虑一个问题，如果脚本已经执行到了while中的后台任务，但在执行到killall命令之前按下了CTRL+C，这时由于没有执行killall，后台任务也将挂在新的脚本进程下。我们的目的是保证脚本终止，其内进程一定终止。所以我们需要对这种情况做出合理的处理。

可以使用trap捕捉ctrl+c信号，捕捉到的时候执行killall命令即可。例如：
```
[root@mariadb ~]# cat test1.sh 
#!/bin/bash

trap "killall `basename $0`" SIGINT
echo $BASHPID

while true;do
    sleep 50
    echo 1
done &

killall `basename $0`
```
这样就能保证脚本终止时，其内一切任务都将终止的目的。

上面的脚本并不健壮，因为`./test1.sh`和`bash test1.sh`两种执行方式的进程名称不一样，前者的进程名称为test1.sh，后者的进程名称为bash，所以killall没法同时解决这两种情况。为了健壮性，可以加上杀后台进程"$!"的代码，并将killall换成pkill，且通过筛选全路径的方式杀掉进程：
```
[root@mariadb ~]# cat test1.sh 
#!/bin/bash

trap "pkill -f `basename $0`" SIGINT
echo $BASHPID

while true;do
    sleep 50
    echo 1
done &

pid=$!
kill $pid
pkill -f `basename $0`
```

为了让脚本自杀更健壮、更通用化，并省去上面结尾处的一大堆额外命令。可以在trap中一次性完成这些任务：
```
#!/bin/bash

trap "pkill -f $(basename $0);exit 1" SIGINT SIGTERM EXIT ERR

while true;do
    sleep 1
    echo "hello world!"
done &

# do something
sleep 60
```

可能写100个shell脚本也遇不到需要一个脚本需要将while/for/until这样的语句放入后台的。但有时候也是有用的。例如，有个需求：每秒去a.txt文件中同步数据到b.txt中，然后每分钟对b.txt文件做处理。
```
#!/bin/bash

while true;do
    (a.txt--->b.txt)
    sleep 1
done &

while true;do
    (b.txt)
    sleep 60
done
```

此外，对一些比较复杂的需求(我个人遇到过多次)，可能也会使用到后台的循环。

本文只是提供一种杀脚本的解决方案。很多情形并非如我这里所描述的，例如不是while循环放后台，而是循环内的sleep放后台，这时(脚本终止时)sleep会挂在init/systemd下，不过这很简单。相信读懂了本文，各位已经了解了一些trap的功能以及处理这类问题的逻辑，也知道其他各种情形如何处理。

最后，有一种更方便更精确的自杀手段：man kill。在该man手册中解释了，如果kill的pid值为0，表示发送信号给当前进程组中所有进程，对shell脚本来说这意味着杀掉脚本中产生的所有进程。方案如下：
```
#!/bin/bash

trap "echo 'signal_handled:';kill 0" SIGINT SIGTERM

while true;do
    sleep 5
    echo "hello world! hello world!"
done &
sleep 60
```


<a name="blog2"></a>
## 补充：bash内置命令的特殊性

为什么上文运行脚本进程，脚本中的后台while会新生成一个脚本进程？在这里补充说明下。


究其原因，是**因为while/for/until等是bash内置命令**，它们的特殊性在于它们有一个很替它们着想的爹：bash进程。bash进程对他们的孩子非常负责，所有能直接执行的内置命令都不会创建新进程，它们直接在当前bash进程内部调用执行，所以我们用ps/top等工具是捕捉不到cd、let、expr等等内置命令的。但正因为爹太负责，把孩子们宠坏了，这些bash内置命令的执行必须依赖于bash进程才能执行。

内置命令中还有几个比较特殊的关键字：while、for、until、if、case等，它们无法直接执行，需要结合其他关键字(如do/done/then等)才能执行。非后台情况下，它们的爹会直接带它们执行，**但当它们放进后台后，它们必须先找个bash爹：**

- **如果是在当前shell中放进后台，则这个爹是新生成的bash进程。**这个新的bash进程只负责一件事，就是负责这个后台，为它的孩子们提供它们依赖的bash环境。  
- **如果是在脚本中放进后台，则这个爹就是脚本进程。**由于脚本不是内置命令，它能直接负责这个后台(因为脚本进程也算是bash进程的特殊变体，也相当于一个新的bash进程)。

验证下就知道咯。

目前bash进程信息为：
```
[root@xuexi ~]# pstree -p | grep bash
           |-sshd(1142)-+-sshd(5396)---bash(5398)---mysql(5659)
           |            `-sshd(7006)-+-bash(7008)
           |                         `-bash(12280)-+-grep(13294)
```
将for、unitl、while、case、if等语句放进后台。例如：
```
[root@xuexi ~]# if true;then sleep 10;fi &  
```
然后再查bash进程信息：
```
[root@xuexi ~]# pstree -p | grep bash
           |-sshd(1142)-+-sshd(5396)---bash(5398)---mysql(5659)
           |            `-sshd(7006)-+-bash(7008)---bash(13295)---sleep(13296)
           |                         `-bash(12280)-+-grep(13298)
```
不难看出，sleep进程之前先生成了一个pid=13295的bash进程。(注：如果这几个特殊关键字不进入后台执行，则是当前在bash进程下执行的)

**无论它们的爹是脚本进程还是新的bash进程，它们都是当前shell下的子shell。如果某个子shell中有后台进程，当杀掉子shell，意味着杀掉了它们的爹。非内置bash命令不依赖于bash，所以直接挂在init/systemd下，而bash内置命令严重依赖于bash爹，没有爹就没法执行，所以在杀掉bash进程(上面pid=7008)的时候，bash爹(pid=13295)会立即带着它下面的进程(sleep)挂在init/systemd下。**

再来验证下咯。还是刚才的后台命令。
```
[root@xuexi ~]# while true;do sleep 2;done &
```
另一个窗口，查看bash进程信息：
```
[root@xuexi ~]# pstree -p | grep bash 
           |-sshd(1142)-+-sshd(5396)---bash(5398)---mysql(5659)
           |            `-sshd(7006)-+-bash(7008)---bash(13468)---sleep(13526)
           |                         `-bash(12280)-+-grep(13528)
```
杀掉pid=7008的bash进程(为什么不杀pid=13468的bash进程？它是为while提供环境的bash进程，杀了这个相当于杀了while循环结构)。注意，这个bash进程是交互式登陆shell，默认情况下会忽略SIGTERM信号，所以只能使用SIGKILL信号来杀。
```
[root@xuexi ~]# kill -9 7008

[root@xuexi ~]# pstree -p | grep bash
           |-bash(13468)---sleep(13562)
           |-sshd(1142)-+-sshd(5396)---bash(5398)---mysql(5659)
           |            `-sshd(7006)---bash(12280)-+-grep(13564)
```
可以看到，这个bash爹带着sleep挂在init/systemd下，这意味着该bash和终端无关。看下面的状态为"?"。
```
[root@xuexi ~]# ps aux | grep bas[h]
root       5398  0.0  0.1 116548  3300 pts/0    Ss   09:04   0:00 -bash
root      12280  0.0  0.1 116568  3340 pts/2    Ss   14:43   0:00 -bash
root      13468  0.0  0.1 116556  1924 ?        S    15:49   0:00 -bash
```

更多内容参考：[bash内置命令的特殊性、后台任务的本质](/shell/jobs_special)