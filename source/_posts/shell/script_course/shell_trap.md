---
title: Shell脚本深入教程：trap信号捕捉用法详解
p: shell/script_course/shell_trap.md
date: 2020-05-19 15:19:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# trap信号捕捉用法详解

在Bash中，可以通过内置的`trap`命令设置信号捕获。trap中文就翻译为陷阱、圈套，即用于布置所谓的陷阱，这个陷阱当然不是捕老鼠的，而是捕捉信号。

比如，按下`Ctrl+C`默认是终止shell脚本(即bash程序)，按下`Ctrl+C`就是向程序(脚本程序)发送了一个SIGINT信号，通过trap设置信号捕获，可以设置在捕获到了某些信号时的处理方式，比如常见的操作是：  
- 忽略信号不管，发送信号等于没发送  
- 删除某些脚本创建的临时文件  
- 杀掉某些脚本中的后台进程  
- 等等  

需注意，trap是bash的内置命令，它设置的信号处理方式只针对bash进程(或脚本进程)本身收到的信号，但是bash进程中调用的命令子进程可能会继承bash进程中设置的信号处理逻辑。

## 常见信号说明

常见的信号以及它们的数值代号、说明如下：
```
Signal     Value   Comment

─────────────────────────────

SIGHUP      1      终止进程，特别是终端退出时，此终端内的进程都将被终止
SIGINT      2      中断进程，几乎等同于sigterm，会尽可能的释放执行clean-up，
                   释放资源，保存状态等(CTRL+C)
SIGQUIT     3      从键盘发出杀死(终止)进程的信号
SIGKILL     9      强制杀死进程，该信号不可被捕捉和忽略，进程收到该信号后不会
                   执行任何clean-up行为，所以资源不会释放，状态不会保存
SIGTERM    15      杀死(终止)进程，几乎等同于sigint信号，会尽可能的释放执行
                   clean-up，释放资源，保存状态等
SIGSTOP    19      该信号是不可被捕捉和忽略的进程停止信息，收到信号后会进入stopped状态
SIGTSTP    20      该信号是可被忽略的进程停止信号(CTRL+Z) 
```

每个信号其真实名称并非是SIGXXX，而是去除SIG后的单词，每个信号还有其对应的数值代号，在使用信号时，可以使用这3种方式中的任一一种。例如SIGHUP，它的信号名称为HUP，数值代号为1，发送HUP信号时，以下3种方式均可。

```bash
kill -1 PID
kill -HUP PID
kill -SIGHUP PID
```

在上面所列的信号列表中，KILL和STOP这两个信号无法被捕捉。一般来说，在设置信号陷阱时，只会考虑HUP、INT、QUIT、TERM这4个会终止、中断进程的信号。

## trap布置陷阱基本用法

trap的语法格式为：

```bash
(1) trap [-lp]
(2) trap cmd-body signal_list
(3) trap '' signal_list
(4) trap    signal_list
(5) trap -  signale_list
```

语法说明：  
- 语法1：-l选项用于列出当前系统支持的信号列表，和`kill -l`一样的作用  
  - -p选项用于列出当前shell环境下已经布置好的陷阱  
- 语法2：当捕捉到给定的信号列表中的某个信号时，就执行此处给定cmd-body中的命令  
- 语法3：命令参数为空字符串，这时**shell进程和shell进程内的子进程**都会忽略信号列表中的信号  
- 语法4：省略命令参数，重置陷阱为启动shell时的陷阱。不建议此语法，因为给定多个信号时结果将出人意料  
- 语法5：等价于语法4  

trap不接任何参数和选项时，默认为`-p`。

(1).查看当前shell已布置的陷阱。

```bash
[root@xuexi ~]# trap
trap -- '' SIGTSTP
trap -- '' SIGTTIN
trap -- '' SIGTTOU
```

这3个陷阱都是信号忽略陷阱，当捕获到TSTP、TTIN或TTOU信号时，将不做任何处理。

(2).设置一个可以忽略`CTRL+C`和15信号的陷阱。

```bash
[root@xuexi ~]# trap '' SIGINT SIGTERM

[root@xuexi ~]# trap
trap -- '' SIGINT
trap -- '' SIGTERM
trap -- '' SIGTSTP
trap -- '' SIGTTIN
trap -- '' SIGTTOU
```

这样一来，当前的shell就无法被`kill -15`杀死。

```bash
[root@xuexi ~]# kill $BASHPID;echo kill current bash failed
kill current bash failed
```

(3).设置一个陷阱，当这个陷阱捕捉到15信号时，就打印一条消息。

```bash
[root@xuexi ~]# trap 'echo caught the TERM signal' TERM   

[root@xuexi ~]# kill $BASHPID
caught the TERM signal
```

再查看已设置的陷阱，之前设置为忽略TERM信号的陷阱已经被覆盖。

```bash
[root@xuexi ~]# trap
trap -- '' SIGINT
trap -- 'echo caught the TERM signal' SIGTERM
trap -- '' SIGTSTP
trap -- '' SIGTTIN
trap -- '' SIGTTOU
```

(4).重置针对INT和TERM这两个信号的陷阱为初始状态。

```bash
[root@xuexi ~]# trap - SIGINT SIGTERM

[root@xuexi ~]# trap
trap -- '' SIGTSTP
trap -- '' SIGTTIN
trap -- '' SIGTTOU
```

(5).在脚本中设置一个能忽略CTRL+C和SIGTERM信号的陷阱。

```bash
[root@xuexi ~]# cat trap1.sh
#!/bin/bash

trap '' SIGINT SIGTERM
sleep 10
echo sleep success
```

当执行该脚本后，将首先陷入睡眠状态，按下CTRL+C将无效。仍会执行完所有的命令。

```bash
[root@xuexi ~]# ./trap1.sh
^C^C^C^Csleep success
```

(6).布置一个当脚本中断时能清理垃圾并立即退出脚本的陷阱。

```bash
[root@xuexi ~]# cat trap1.sh
#!/bin/bash

trap 'echo trap handling...;rm -rf /tmp/$BASHPID$BASHPID;echo TEMP file cleaned;exit' SIGINT SIGTERM SIGQUIT SIGHUP
mkdir -p /tmp/$BASHPID$BASHPID/
touch /tmp/$BASHPID$BASHPID/{a.txt,a.log}
sleep 10
echo first sleep success
sleep 10
echo second sleep success
```

这样，无论是什么情况中断(除非是SIGKILL)，脚本总能清理掉临时垃圾。

## 搞懂trap的信号处理机制，使用trap布置完美陷阱

**(1).trap设置的信号处理陷阱，其守护对象是shell进程本身，不会将收到的信号转发给shell环境内的子进程。这意味着shell进程收到信号时，可能会等待其子进程执行完成才执行信号处理逻辑**

以下面这个脚本为例，设置的陷阱会捕捉到SIGINT和SIGTERM两个信号，捕捉到信号时将输出陷阱做出处理的时间点。

```bash
[root@xuexi ~]# cat trap2.sh
#!/bin/bash

trap 'echo trap_handle_time: $(date +"%F %T")' SIGINT SIGTERM
echo time_start: $(date +"%F %T")
sleep 10
echo time_end1: $(date +"%F %T")
sleep 10
echo time_end2: $(date +"%F %T")
```

执行该脚本，并另开一个会话窗口，杀死trap2.sh脚本。

```bash
[root@xuexi ~]# ./trap2.sh 
[root@xuexi ~]# killall -s SIGTERM trap2.sh  # 另一个窗口执行
```

执行结果如下。

```bash
time_start: 2017-08-14 **12:59:23**
trap_handle_time: 2017-08-14 **12:59:33**   
time_end1: 2017-08-14 12:59:33
time_end2: 2017-08-14 12:59:43 
```

结果中的trap_handle_time证明，脚本所在shell进程收到SIGTERM信号后，trap成功进行了处理。如果细心的话，会发现trap处理的时间正好是10秒之后，这并不是因为正好10秒之后才发送SIGTERM信号，而是因为trap就是这么工作的，**它不会把shell脚本进程(当前的bash进程)收到的信号也发送给在该脚本中执行的子进程**，并且在执行到第一个sleep命令时发送SIGTERM信号给脚本进程，此时shell脚本进程正处于等待sleep执行完毕的阻塞状态，所以在sleep执行完成后，shell进程才开始执行信号处理逻辑。

**(2).shell环境内的子进程会继承其父进程(shell进程)中设置的信号处理逻辑。**

再次执行上面的trap2.sh脚本，但是这次不是发送信号给脚本进程，而是发送给其内的sleep子进程。

```bash
[root@xuexi ~]# ./trap2.sh

# 另一个会话终端下执行此命令，发送信号给sleep子进程
[root@xuexi ~]# killall -s SIGTERM sleep ;sleep 3; killall -s SIGINT trap2.sh  
```

最终将返回如下结果：

```bash
time_start: 2017-08-14 12:23:06
Terminated                        # 接收到对sleep发送的SIGTERM信号
time_end1: 2017-08-14 12:23:09    # 没有trap_handle_time，陷阱没有守护sleep进程
trap_handle_time: 2017-08-14 12:23:19   # shell进程本身收到了SIGINT信号，并被陷阱处理了
time_end2: 2017-08-14 12:23:19
```

之所以会有上面的结果，是因为sleep是shell进程的子进程，它继承了气父进程(shell进程)中的信号处理逻辑，也就是说，该脚本中的sleep进程接收到SIGTERM或SIGINT时，将会被终止。

另一方面，发送SIGINT信号给脚本时，脚本并不会向其子进程转发信号，因此后面发送给脚本进程的SIGINT信号不会终止sleep子进程。

再修改脚本中的陷阱为信号忽略陷阱。

```bash
[root@xuexi ~]# cat ./trap3.sh
#!/bin/bash

trap '' SIGINT SIGTERM
echo time_start: $(date +"%F %T")
sleep 10
echo time_end1: $(date +"%F %T")
sleep 10
echo time_end2: $(date +"%F %T") 
```

执行trap3.sh，并在另一个会话终端下杀死sleep进程。

```bash
[root@xuexi ~]# ./trap3.sh

# 另一个会话终端下执行此命令
[root@xuexi ~]# killall -s SIGTERM sleep;sleep 3;killall -s SIGINT sleep   
```

结果如下：

```bash
time_start: 2017-08-14 12:31:54
time_end1: 2017-08-14 12:32:04
time_end2: 2017-08-14 12:32:14 
```

从时间差可以看出，无论是SIGTERM还是SIGINT信号，sleep进程都被忽略型trap守护了。理由前面解释过了，shell脚本内部的sleep子进程继承了shell脚本进程的信号处理逻辑，即忽略SIGTERM和SIGINT信号。

**(3).CTRL+C和SIGINT不是等价的。当某一时刻按下CTRL+C，它是在向整个当前运行的进程组发送SIGINT信号。对shell脚本来说，CTRL+C的SIGINT信号不仅发送给shell脚本进程，还发送给脚本中当前正在运行的进程。**

所以，如果shell中设置SIGINT陷阱，`CTRL+C`不仅会终止脚本中当前正在运行的进程，trap还会立即进行对应的处理。

以下面的脚本trap4.sh为例。

```bash
[root@xuexi ~]# cat trap4.sh
#!/bin/bash

trap 'echo trap_handle_time: $(date +"%F %T")' SIGINT
echo time_start: $(date +"%F %T")
sleep 10
echo time_end1: $(date +"%F %T")
sleep 10
echo time_end2: $(date +"%F %T")
```

如果使用kill命令向trap4.sh发送信号，正常情况下trap会在当前运行的sleep进程完成后才进行相关处理。但如果是按下CTRL+C，先看结果。

```bash
[root@xuexi ~]# ./trap4.sh 
time_start: 2017-08-14 13:41:30
^Ctrap_handle_time: 2017-08-14 13:41:31
time_end1: 2017-08-14 13:41:31
^Ctrap_handle_time: 2017-08-14 13:41:32
time_end2: 2017-08-14 13:41:32
```

结果中显示，两次按下CTRL+C后，不仅sleep立刻结束了，trap也立即进行处理了。这说明CTRL+C不仅让脚本进程收到了SIGINT信号，也让当前正在运行的进程收到了SIGINT信号。

需要特别说明的是，如果当前正在运行的进程处在循环内，当该进程收到了终止进程后，仅仅只是立即终止当次进程，而不会终止整个循环，也就是说，它还会继续向下执行后续命令并进入下一个循环。如果此时是使用CTRL+C发送SIGINT，则每次CTRL+C时，trap也会一次次进行处理。

注意这些信号处理的机制很重要，因为搞清楚了它们，才能明白脚本中当前正在运行的进程是先完成还是立即结束，这在写复杂脚本或任务型脚本极其重要。

**(4).每个陷阱都有守护范围。每一个陷阱只将守护它后面的所有进程，直到遇到下一个相同信号的陷阱。**

以shell脚本为例，如下图所示。

![](/img/shell/1589876167413.png)

**(5).信号忽略陷阱是子shell唯一继承的陷阱类型，且子进程的信号忽略陷阱不可再改变或重置。也就是说，当shell环境下设置了信号忽略陷阱时，子shell在启动时将继承该陷阱。**

先在当前shell环境下设置一个忽略SIGINT的陷阱，和一个不忽略SIGTERM的陷阱。

```bash
[root@xuexi ~]# trap '' SIGINT
[root@xuexi ~]# trap 'echo haha' SIGTERM
```

以下是测试脚本。脚本中首先输出脚本刚启动时的最初陷阱列表，随后修改陷阱并输出新的陷阱列表，最后重置陷阱并输出重置后的陷阱列表。

```bash
[root@xuexi ~]# cat trap6.sh 
#!/bin/bash

echo old_trap:--------
trap -p
trap 'echo haha' SIGINT SIGTERM
echo new_trap:--------
trap -p
echo "reset trap:------"
trap - SIGINT SIGTERM
trap -p
```

执行结果如下。

```bash
[root@xuexi ~]# ./trap6.sh     
old_trap:--------
trap -- '' SIGINT
new_trap:--------
trap -- '' SIGINT
trap -- 'echo haha' SIGTERM
reset trap:------
trap -- '' SIGINT
```

从结果中可以看出，启动脚本时，父shell中忽略SIGINT的陷阱被继承了，但不忽略信号的陷阱未被继承。而且脚本继承的信号忽略陷阱无法被修改和重置。

**(6).交互式的shell下，如果没有定义任何SIGTERM信号的陷阱，则会忽略该信号。**

所以，在默认(未定义SIGTERM陷阱)时，无法直接通过15信号杀死当前bash进程。

```bash
[root@xuexi ~]# kill $BASHPID;echo passed;kill -9 $BASHPID
passed
          # 此处当前bash已被kill -9强制杀死
```

交互式Shell默认忽略SIGTERM信号的机制在某些时候非常好用。在我们对bash的理解还不深入时，我们经常会错误地执行循环后台命令，这些不断出现的后台命令的输出会扰乱当前Shell的终端界面，除使用`killall bash`外，没有更好更便捷的办法清理这些不断出现的循环后台进程。`killall bash`只会杀掉非交互式的bash进程，而不会中断任何当前正在交互的bash进程。

例如，执行下面这条命令，它会每隔1秒在当前终端输出时间信息，你没法快捷地终止它们。最好的办法是输入`killall bash`，在当前终端窗口或其它终端窗口输入都可以
```bash
[root@xuexi ~]# ( while true;do date +"%T";sleep 1;done & )
```

**(7).除了`kill -`l或`trap -l`列出的信号列表，trap还有4种特殊的信号：EXIT(或信号代码0)、ERR、DEBUG和RETURN。DEBUG和RETURN这两种信号陷阱无需关注**。

EXIT信号也是0信号，当设置了EXIT陷阱时，每当shell进程终止时(无论何种方式，除非`kill -9`杀掉shell进程)都会被捕捉，并做相关处理。(0信号不执行任何动作，但执行错误检查：当检查发现给定的pid进程存在，则返回0，否则返回1)

下面两个信号处理等价：

```bash
trap 'cmds' INT HUP QUIT TERM
trap 'cmds' EXIT
```

**ERR陷阱是在shell(比如脚本)以非0状态码退出时生效的**。特别是在设置了`set -e`时，只要某命令的状态码非0，就会直接退出当前shell(比如shell脚本)，有了它就不用再在脚本中书写对`$?`是否(不)等于0的判断语句，不过它主要用于避免脚本中产生错误时，错误被滚雪球式的不断放大。很多人将这一设置当作写shell脚本的一项行为规范，但我个人不完全认同，很多时候非0退出状态码是无关紧要的，甚至有时候非0状态码才是继续执行的必要条件。

回到话题上。先看看`set -e`的效果。以下面的脚本为例，在脚本中，mv命令少给了一个参数，它是错误命令，返回的是非0状态码。

```bash
[root@xuexi ~]# vim trap8.sh
#!/bin/bash
set -e
echo "right here"
mv ~/a.txt
[ "$?" -eq 0 ] && echo "right again" || echo "wrong here"
```

如果不设置`set -e`，那么会被下一条语句判断，但因为设置了`set -e`，使得在mv错误发生时，就立即退出脚本所在的shell。也就是说，对`$?`的判断语句根本就是多余的。结果如下。

```bash
[root@xuexi ~]# ./trap8.sh 
right here
mv: missing destination file operand after '/root/a.txt'
Try 'mv --help' for more information
```

可以设置ERR陷阱，专门捕获`set -e`起作用时的信号。例如，当命令错误时，做一些临时文件清理动作等。注意两点：(1).ERR陷阱和`set -e`没关系，只要shell的退出状态码非0，就会生效，`set -e`只不过是将shell中(如shell脚本)任何非0退出码的命令都使shell立即非0退出，从而使ERR陷阱生效；(2).当捕获到了ERR信号时，脚本不会再继续向下运行，而是trap处理结束后就立即退出。例如：

```bash
[root@xuexi ~]# vim trap8.sh
#!/bin/bash
set -e
trap 'echo continue' ERR
echo "right here"
mv ~/a.txt
[ "$?" -eq 0 ] && echo "right again" || echo "wrong here"
echo haha
```

执行结果如下：

```bash
[root@xuexi ~]# ./trap8.sh  
right here
mv: missing destination file operand after ‘/root/a.txt’
Try 'mv --help' for more information.
continue
```

**(8).在trap中两个很好用的变量：BASH_COMMAND和LINENO。BASH_COMMAND变量记录的是当前正在执行的命令行，如果是用在陷阱中，则记录的是陷阱触发时正在运行的命令行。LINENO记录的是正在执行的命令所处行号。**

例如：

```bash
[root@xuexi ~]# vim trap8.sh 
#!/bin/bash
set -e
trap 'echo "error line: $LINENO,error cmd: $BASH_COMMAND"' ERR
echo "right here"
mv ~/a.txt
```

执行结果。

```bash
[root@xuexi ~]# ./trap8.sh   
right here
mv: missing destination file operand after ‘/root/a.txt’
Try 'mv --help' for more information.
error line: 5,error cmd: mv ~/a.txt
```

**(9).处理脚本中启动的后台进程。**

通常trap在脚本中的作用之一是在突然被中断时清理一些临时文件然后退出，虽然它会等待脚本中当前正在运行的命令结束，然后清理并退出。

但是，很多时候会在脚本中使用后台进程，以加快脚本的速度。而子shell中的后台子进程在其父shell进程中断时会独立挂在init/systemd进程下称为孤儿进程，所以它不会受shell进程终止的影响，更不受shell环境的影响。换句话说，当脚本突然被中断时，即使陷阱捕获到了该信号，并清理了临时文件后退出，但是那些脚本中启动的后台进程还会继续运行。

例如，执行如下脚本并终止，会发现sleep进程并未终止，且其父进程PID为1。
```bash
[root@xuexi ~]# cat sub.sh
#!/bin/bash

sleep 200 &

sleep 10
```
执行并立即按下`Ctrl+C`，会发现虽然脚本进程已经终止了，但后台子进程sleep进程仍在运行。
```bash
[root@xuexi ~]# ./sub.sh
^C            # 按下 CTRL + C
[root@xuexi ~]# ps -l | grep -iE 'sleep|ppid' | grep -v grep
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000  2315     1  0  80   0 -  3819 -      tty1     00:00:00 sleep
```

亦或者严重一点的：

```bash
!/bin/bash

# 将while循环加入后台，脚本终止后，该while循环还会继续运行
# 该while所在的bash进程已经称为pid=1的进程的子进程
while true;do
  sleep 1
  date +"%F"
done &

sleep 2
```


该脚本执行2秒之后，会每隔1秒向终端上输出一个时间信息，扰乱终端界面。可以通过`killall bash`的方式终止后台还在运行的进程。

这就给脚本带来了一些不可预测性，一个健壮的脚本必须能够正确处理这种情况。trap可以实现比较好的解决这种问题，方法是在trap的命令行中加上向后台进程发送信号的语句，然后再退出。

以下面的脚本为例。

```bash
[root@xuexi ~]# vim trap10.sh
#!/bin/bash
trap 'echo first trap $(date +"%F %T");exit' SIGTERM
echo first sleep $(date +"%F %T")
sleep 20 &
echo second sleep $(date +"%F %T")
sleep 5
```

该脚本中首先将一个sleep放入后台运行。正常情况下，该脚本执行5秒后就会退出，但在20秒后后台进程sleep才会结束，即使突然发送中断信号TERM触发trap也一样。

于是现在的目标是，在sleep 5的过程中突然中断脚本时，能杀死后台sleep进程。可以使用`!`这个特殊的Bash内置变量，它记录了当前Shell环境下的上一个后台子进程的PID。修改后的脚本如下。

```bash
[root@xuexi ~]# vim trap10.sh
#!/bin/bash
trap 'echo first trap $(date +"%F %T");**kill $pid**;exit' SIGTERM

echo first sleep $(date +"%F %T")
sleep 20 &
pid="$!"
sleep 30 &
pid="$! $pid"
echo second sleep $(date +"%F %T")
sleep 5
```

执行该脚本，并在另一个会话窗口发送SIGTERM信号给该脚本进程。

```bash
[root@xuexi ~]# ./trap10.sh ; ps aux | grep sleep        
[root@xuexi ~]# kill trap10.sh    # 另一个会话窗口执行
```

执行结果如下。可见sleep被正常终止。

```bash
first sleep 2017-08-14 21:29:19
second sleep 2017-08-14 21:29:19
first trap 2017-08-14 21:29:24
root  69096  0.0  0.0 112644  952 pts/0  S+  21:29  0:00 grep --color=auto sleep
```

除了通过`$!`收集后台进程PID值，还可以直接使用kill发送终止信号给`0`，通过`man kill`查看，可以知道发送信号给0时，会将信号发送给当前进程的进程组内的所有进程。
```bash
#!/bin/bash
trap 'echo first trap $(date +"%F %T");kill 0;exit' SIGTERM

echo first sleep $(date +"%F %T")
sleep 20 &
pid="$!"
sleep 30 &
pid="$! $pid"
echo second sleep $(date +"%F %T")
sleep 5
```

**(10).通过发送信号去终止脱离了终端的后台进程时，只能发送`SIGHUP SIGTERM`信号，不能发送`SIGINT SIGQUIT`信号，因为SIGINT信号和SIGQUIT信号都是通过键盘发送的信号，对脱离了终端的后台进程无效。**

例如，下面的 sleep 进程会脱离终端称为PID=1的进程的子进程，发送SIGINT信号和SIGQUIT信号给它是无法终止它的，但是发送SIGHUP信号或SIGTERM信号可以终止。
```bash
[root@xuexi ~]# (sleep 300 &)

# 发送SIGINT，无效
[root@xuexi ~]# killall -INT sleep
[root@xuexi ~]# ps -l | grep -iE 'sleep|ppid' | grep -v grep
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000  2612     1  0  80   0 -  3819 -      tty1     00:00:00 sleep

# 发送SIGTERM信号，有效
[root@xuexi ~]# killall -TERM sleep
[root@xuexi ~]# ps -l | grep -iE 'sleep|ppid' | grep -v grep
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
```

所以，在shell脚本中想要让后台进程随脚本终止而终止时，不应向后台进程发送SIGINT或SIGQUIT，而是发送SIGTERM或SIGHUP信号。
```bash
#!/bin/bash

# 发送TERM信号给当前进程组，会终止所有后台进程
trap 'echo trapped;kill 0;exit' EXIT

sleeep 300 &
sleep 200 &
sleep 10
```
