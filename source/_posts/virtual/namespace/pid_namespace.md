---
title: Linux namespace之：pid namespace
p: virtual/namespace/pid_namespace.md
date: 2020-10-10 17:37:32
tags: Virtualization
categories: Virtualization
---

--------

**目录**：  
- **[Linux namespace概述](/virtual/namespace/ns_overview)**  
- **[Linux namespace之：uts namespace](/virtual/namespace/uts_namespace)**  
- **[Linux namespace之：mount namespace](/virtual/namespace/mount_namespace)**  
- **[Linux namespace之：pid namespace](/virtual/namespace/pid_namespace)**  
- **[Linux namespace之：network namespace](/virtual/namespace/network_namespace)**  
- **[Linux namespace之：user namespace](/virtual/namespace/user_namespace)**  

--------


# 理解pid namespace

PID namespace表示隔离一个具有独立PID的运行环境。在每一个pid namespace中，进程的pid都从1开始，且和其他pid namespace中的PID互不影响。这意味着，不同pid namespace中可以有相同的PID值。

因为PID namespace中的PID是独立的，每一个PID namespace都允许一些特殊的操作：允许pid namespace挂起、迁移以及恢复，就像虚拟机一样。

在介绍PID namespace之前，有必要回顾一下创建其他类型namespace时的进程关系。

```bash
# 在root namespace中
$ echo $$
12314
$ sudo unshare -u /bin/bash
[ns1]$ pstree -p | grep grep
   |   `-sshd(12313)---bash(12314)---sudo(14930)---bash(14931)-+-grep(14942)
```

其中sudo在root namespace中，其子进程bash(14931)在新创建的uts namespace ns1中。从上面输出的结果可知，创建其他类型namespace时，unshare进程会在创建新的namespace后被该namespace中的第一个进程给替换掉。如果忘记了，请回到前文[uts namespace](/virtual/namespace/uts_namespace#process_relationship)复习复习。

了解了创建其他类型namespace进程关系的基础后，再来对比介绍pid namespace。

创建新的pid namespace的方式：

```bash
# unshare --pid --fork [--mount-proc] <CMD>
# --pid或-p表示创建pid namespace
# --fork或-f表示创建pid namespace时，不是直接
#   替换unshare进程，而是fork unshare进程，
#   并使用CMD替换fork出来的子进程
# --mount-proc表示创建pid namespace时重新挂载procfs
sudo unshare -p -f -m -u --mount-proc /bin/bash
```

`--fork`以及`--mount-proc`选项稍后再解释。先看看创建pid namespace后的进程关系：

```bash
$ sudo unshare -p -f -m -u --mount-proc /bin/bash
root@longshuai-vm:/home/longshuai# hostname ns1
root@longshuai-vm:/home/longshuai# exec bash
root@ns1:/home/longshuai# pstree -p | grep bash
bash(1)-+-grep(11)    # pid namespace中，第一个进程bash其PID=1
                      # 该namespace中没有其他进程

# 在第二个shell窗口会话中执行
$ pstree -p | grep sudo
 |  `-sshd(12313)---bash(12314)---sudo(15070)---unshare(15071)---bash(15072)
```

注意上面输出的进程关系，和之前创建普通的namespace的进程关系不同，创建PID namespace时，sudo的子进程unshare进程保留了，这就是命令行中使用`--fork`的效果：在unshare中创建出pid namespace后，它将fork出它的子进程加入到新的pid namespace中，并在该子进程中exec加载指定的/bin/bash进程作为该pid namespace中的第一个进程。

所以使用`--fork`后导致的结果是：unshare进程被保留，且保留在原来的pid namespace中，而不是加入新的pid namespace中(在`man pid_namespaces`中明确指出了创建pid namespace的unshare或setns进程不会也不能进入新的pid namespace)。

```bash
# unshare进程和bash(12314)、sudo(15070)都在同一个pid namespace
$ pstree -p | grep sudo
 |  `-sshd(12313)---bash(12314)---sudo(15070)---unshare(15071)---bash(15072)

$ sudo ls -l /proc/12314/ns/pid
lrwxrwxrwx ... /proc/12314/ns/pid -> 'pid:[4026531836]'
$ sudo ls -l /proc/15070/ns/pid
lrwxrwxrwx ... /proc/15070/ns/pid -> 'pid:[4026531836]'
$ sudo ls -l /proc/15071/ns/pid
lrwxrwxrwx ... /proc/15071/ns/pid -> 'pid:[4026531836]'

# 但unshare的子进程bash在新的pid namespace中
$ sudo ls -l /proc/15072/ns/pid
lrwxrwxrwx ... /proc/15072/ns/pid -> 'pid:[4026532590]'
```

在新创建的pid namespace中使用ps命令查看进程信息：

```bash
root@ns1:~# ps j
   PPID   PID  PGID  SID TTY    TPGID STAT  UID   TIME COMMAND
      0     1     1    0 pts/0     21 S       0   0:00 bash
      1    21    21    0 pts/0     21 R+      0   0:00 ps j
```

关注输出结果中的两点：

1.pid namespace中的第一个进程bash，其PID=1  

2.PID=1的进程的PPID=0，这和root namespace中pid=1的init进程的PPID=0是一样的  

```bash
$ ps -elf
F S UID   PID    PPID  ... CMD
4 S root    1       0  ... /sbin/init splash
1 S root    2       0  ... [kthreadd]
...
```

在pid namespace中，PID=1的进程作为该pid namespace环境的【init】进程，当该pid namespace中的某个进程A的父进程退出了，进程A将称为孤儿进程，孤儿进程将被该pid namespace中PID=1的进程收养。

另外，观察旧namespace以及新pid namespace中的进程：

```bash
# 原来的namespace
$ pstree -p | grep sudo
 |  `-sshd(12313)---bash(12314)---sudo(15070)---unshare(15071)---bash(15072)
 
# 新创建的pid namespace
root@ns1:/home/longshuai# pstree -p | grep bash
bash(1)-+-grep(11)
```

其实unshare的子进程bash(15072)和pid namespace中的bash(1)是同一个进程。查看它们的pid namespace的inode可知：

```bash
# 原来的namespace中执行
$ sudo readlink /proc/16158/ns/pid
pid:[4026532593]

# 新的pid namespace中执行
root@longshuai-vm:/home/longshuai# readlink /proc/$$/ns/pid
pid:[4026532593]
```

所以可得知一个结论：当在namespace ns1中创建一个pid namespace ns2，祖先ns1中可查看ns2中的所有进程，但ns2中无法查看祖先ns1中的进程。

另外需要注意的是，`--fork`并非一定是结合`--pid`创建pid namespace使用的，它也可以直接使用或结合其他类型的namespace选项使用。例如：

```bash
$ sudo unshare --fork sleep 3
```

这仅仅表示在一个子进程中执行sleep程序，这里的unshare并没有创建出namespace，所以上面的命令和在子shell中执行sleep命令没什么区别，例如`(sleep 3)`。

## 嵌套pid namespace

其实，对于pid namespace来说，是存在嵌套关系的。所有的子孙pid namespace中的进程信息都会保存在父级以及祖先级namespace中，只不过在不同嵌套层级中，同一个进程对应的PID不同。

例如，创建嵌套了2层的pid namespace：

```bash
# 当前namespace为ns0

# 创建pid namespace ns1
$ sudo unshare -mupf --mount-proc /bin/bash

# 在ns1中创建pid namespace ns2
root@longshuai-vm:/home/longshuai# unshare -mupf --mount-proc /bin/bash

# 在ns2中创建pid namespace ns3
root@longshuai-vm:/home/longshuai# unshare -mupf --mount-proc /bin/bash
```

现在ns0有3个子孙级的pid namespace：ns1、ns2和ns3。

在ns0中查看进程关系：

```bash
$ pstree -lp | grep -oE 'sudo.*'
sudo(16510)---unshare(16511)---bash(16513)---unshare(16531)---bash(16532)---unshare(16539)---bash(16540)
```

其中每一对`unshare---bash`代表一个pid namespace(但注意，unshare不在其子进程bash所在的pid namespace中)。

```bash
unshare(16511)---bash(16513)
unshare(16531)---bash(16532)
unshare(16539)---bash(16540)
```

这三个bash进程在各自的pid namespace中的PID=1。但从输出结果可知，在ns0中，这三个pid namespace中的PID分别为：

```bash
bash(16513)
bash(16532)
bash(16540)
```

PID=16513对应ns1中的bash，PID=16532对应ns2中的bash，PID=16540对应ns3中的bash。

在`/proc/<PID>/status`中的NSPID字段记录了当前进程`<PID>`在各父级pid namespace中对应的PID值。

例如，在ns0中查看ns3中的bash(pid=16540)进程：

```bash
$ grep 'NSpid' /proc/16540/status
NSpid:  16540    25      9       1
#       当前ns   ns1     ns2     ns3
```

这表示pid=16540这个进程对应ns1中的pid=25，对应ns2中的pid=9，对应ns3中的pid=1。

同样地，可以进入到ns1中(bash pid=16513)去查看进程pid=25的进程对应关系：

```bash
# nsenter命令可进入一个已存在的namespace，
#   -t PID指定进入哪一个目标(target)namespace，
#      PID可以是属于目标namespace中的任何一个进程
#   -F 表示不fork nsenter进程，而是直接使用/bin/bash替换nsenter进程
#      所以，nsenter允许不fork，但unshare创建pid namespace时必须fork
# 就像登录系统一样，比如ssh登录时会启动bash(或其他shell)，
# 这可以看作是进入root namespace并启动一个bash环境
# nsenter用法很简单，可查看nsenter --help或man nsenter
$ sudo nsenter -m -u -p -F -t 16513 /bin/bash

# 在ns1中查看进程关系
# bash(1)是当前ns1中的第一个进程
# bash(17)是ns2中的第一个进程
# bash(25)是ns3中的第一个进程
root@longshuai-vm:/# pstree -pl | grep unshare
bash(1)---unshare(16)---bash(17)---unshare(24)---bash(25)

# 在ns1中查看ns3中的bash进程在各父级pid namespace中的对应关系
# ns3 bash在ns1中的进程PID=25
#         在ns2中的进程PID=9
#         在ns3中的进程PID=1
root@longshuai-vm:/# grep 'NSpid' /proc/25/status
NSpid:  25      9       1
```

## pid namespace和procfs(/proc)

`/proc`目录是内核对外暴露的可供用户查看或修改的内核中所记录的信息，包括内核自身的部分信息以及每个进程的信息。比如对于pid=N的进程来说，它的信息保存在`/proc/<N>`目录下。

`/proc`是一个挂载点，是伪文件系统procfs的挂载点，文件系统类型是proc：

```bash
$ mount | grep proc
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
```

在操作系统启动的过程中，会挂载procfs到/proc目录，它存在于root namespace中。

但是，创建新的pid namespace时不会自动重新挂载procfs，而是直接拷贝父级namespace的挂载点信息。这使得在新的pid namespace中仍然保留了父级namespace的`/proc`目录，也就是在新创建的这个pid namespace中仍然保留了父级的进程信息。

```bash
# 在root namespace中查看pid namespace的inode
$ readlink /proc/self/ns/pid
pid:[4026531836]

# 创建pid namespace，注意没有加上--mount-proc选项
$ sudo unshare -m -u -p -f -w"/" /bin/bash

# 进入pid namespace后，查看PID=1的进程pid inode
# 在pid namespace中，pid=1的进程是bash，查看结果
# 本该是新的pid namespace中的pid inode值，但实际
# 结果是root namespace中pid=1的init进程的pid inode
root@longshuai-vm:/# readlink /proc/1/ns/pid  
pid:[4026531836]

# 通过ps查看pid=1的进程，发现是systemd(init)，而不是bash
root@longshuai-vm:/# ps -p 1 -o pid,ppid,comm
    PID    PPID COMMAND
      1       0 systemd
```

之所以有上述问题，其原因是在pid namespace中保留了root namespace中的/proc目录，而不是属于pid namespace自己的/proc。

但用户创建pid namespace时希望的是有完全独立的进程运行环境。这时，需要在pid namespace中重新挂载procfs，或者在创建pid namespace时指定`--mount-proc`选项。

```bash
# 重新挂载procfs
root@longshuai-vm:/# mount -t proc proc /proc

# 重新挂载procfs后，/proc目录保存的是当前
# pid namespace中的进程信息
root@longshuai-vm:/# ps -p 1 -o pid,ppid,comm
    PID    PPID COMMAND
      1       0 bash
      
root@longshuai-vm:/# readlink /proc/1/ns/pid  
pid:[4026532591]

# 或者直接在创建pid namespace时加上--mount-proc选项
# sudo unshare -p -f -m -u --mount-proc /bin/bash
```

## pid namespace的信号问题

pid=1的进程是每一个pid namespace(无论是root namespace还是用户自己创建的pid namespace)的核心进程，它不仅负责收养其所在pid namespace中的孤儿进程，还影响整个pid namespace。

当pid namespace中pid=1的进程退出或终止，内核默认会发送SIGKILL信号给该pid namespace中的所有进程以便杀掉它们(如果该pid namespace中有子孙namespace，也会直接被杀)。

在创建pid namespace时可以通过`--kill-child`选项指定pid=1的进程终止后内核要发送给pid namespace中进程的信号，其默认信号便是SIGKILL。

```bash
# 例如指定发送SIGHUP信号
sudo unshare -p -f -m -u --mount-proc --kill-child=SIGHUP /bin/bash
```

在pid namespace内部，只能向pid=1的进程发送那些在pid=1的进程中注册了信号处理程序的信号。

例如，如果pid namespace中/bin/bash作为pid=1的进程，那么在此环境中，只能向该bash进程发送设置了trap的信号，其他信号都被直接忽略，这样可以避免无法信号导致pid namespace被自己内部的进程终止。

但对于pid=1的/bin/bash做测试，发现SIGHUP信号总是有效。

```bash
$ sudo unshare -p -f -m -u --mount-proc -w'/' /bin/bash

# 设置SIGINT信号的信号处理程序，现在bash进程只接收SIGINT信号
root@longshuai-vm:/# trap 'echo int' INT

# 发送INT信号有效
root@longshuai-vm:/# /bin/kill -INT 1
int

# 发送其他信号无效，即便是STOP和KILL信号
root@longshuai-vm:/# /bin/kill -QUIT 1
root@longshuai-vm:/# /bin/kill -STOP 1
root@longshuai-vm:/# /bin/kill -KILL 1

# 发送HUP信号也有效，即使没有设置HUP信号处理程序
root@longshuai-vm:/# /bin/kill -HUP 1
$  # 已退出pid namespace
```

在pid namespace外部，只有其祖先级namespace可发送信号给该pid namespace中的进程，因为其他namespace中看不到该pid namespace中的进程信息。但是发送信号时要找准pid namespace中的进程在祖先namespace中的PID值。

例如：

```bash
# 在root namespace中，创建pid namespace ns1，并睡眠30秒
$ sudo unshare -p -f -m -u --mount-proc -w'/' /bin/bash
root@longshuai-vm:/# sleep 300

# 打开第二个会话窗口，在root namespace中杀掉ns1中的sleep进程
$ pstree -lp | grep -o "sleep.*"
sleep(24685)
$ sudo kill 24685
```




