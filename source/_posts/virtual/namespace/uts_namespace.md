---
title: Linux namespace之：uts namespace
p: virtual/namespace/uts_namespace.md
date: 2020-10-10 17:37:30
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

# 理解uts namespace

uts(UNIX Time-Sharing System) namespace可隔离hostname和NIS Domain name资源，使得一个宿主机可拥有多个主机名或Domain Name。换句话说，可让不同namespace中的进程看到不同的主机名。

例如，使用unshare命令(较新版本Linux内核还支持nscreate命令)创建一个新的uts namespace：
```bash
# -u或--uts表示创建一个uts namespace
# 这个namespace中运行/bin/bash程序
$ hostname
longshuai-vm      # 当前root namespace的主机名为longshuai-vm
$ sudo unshare -u /bin/bash
root@longshuai-vm:/home/longshuai#   # 进入了新的namespace中的shell
                                     # 其主机名初始时也是longshuai-vm，
                                     # 其拷贝自上级namespace资源
```

上面指定运行的是/bin/bash程序，这会进入交互式模式，当执行exit时，bash退出，回到当前的namespace中。也可以指定在namespace中运行其他程序，例如`unshare -u sleep 3`表示在uts namespace中睡眠3秒后退出并回到当前namespace。

因为是uts namespace，所以可在此namespace中修改主机名：

```bash
# 修改该namespace的主机名为ns1
# 修改后会立即生效，但不会显示在当前Shell提示符下
# 需重新加载Shell环境
root@longshuai-vm:/home/longshuai# hostname ns1
root@longshuai-vm:/home/longshuai# hostname
ns1
root@longshuai-vm:/home/longshuai# exec $SHELL
root@ns1:/home/longshuai#
```

namespace中修改的主机名不会直接修改主机名配置文件(如/etc/hostname)，而是修改内核属性/proc/sys/kernel/hostname：
```bash
root@ns1:/home/longshuai# cat /proc/sys/kernel/hostname
ns1
root@ns1:/home/longshuai# cat /etc/hostname 
longshuai-vm
```

<a name="process_relationship"></a>

创建了新的namespace并在其中运行/bin/bash进程后，再去关注一下进程关系：

```bash
# ns1中的bash进程PID
root@ns1:/home/longshuai# echo $$
14279

# bash进程(PID=14279)和grep进程运行在ns1 namespace中，
# 其父进程sudo(PID=14278)运行在ns1的上级namespace即root namespace中
root@ns1:/home/longshuai# pstree -p | grep $$
    |-sshd(10848)---bash(10850)---sudo(14278)---bash(14279)-+-grep(14506)

# 运行在ns1中当前bash进程(PID=14279)的namespace
root@ns1:/home/longshuai# ls -l /proc/14279/ns
lrwxrwxrwx ... cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx ... ipc -> 'ipc:[4026531839]'
lrwxrwxrwx ... mnt -> 'mnt:[4026531840]'
lrwxrwxrwx ... net -> 'net:[4026531992]'
lrwxrwxrwx ... pid -> 'pid:[4026531836]'
lrwxrwxrwx ... pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx ... user -> 'user:[4026531837]'
lrwxrwxrwx ... uts -> 'uts:[4026532588]'  # 注意这一行，和sudo进程的uts inode不同

# 父进程sudo(PID=14278)不在ns1中，它的namespace信息
root@ns1:/home/longshuai# ls -l /proc/14278/ns
lrwxrwxrwx ... cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx ... ipc -> 'ipc:[4026531839]'
lrwxrwxrwx ... mnt -> 'mnt:[4026531840]'
lrwxrwxrwx ... net -> 'net:[4026531992]'
lrwxrwxrwx ... pid -> 'pid:[4026531836]'
lrwxrwxrwx ... pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx ... user -> 'user:[4026531837]'
lrwxrwxrwx ... uts -> 'uts:[4026531838]'   # 注意这一行，和PID=1的uts inode相同
```

回到创建uts namespace时敲下的unshare命令：
```bash
sudo unshare -u /bin/bash
```
从进程关系`...---sudo(14278)---bash(14279)`可知两个进程PID是连续的，说明unshare程序对应的进程被/bin/bash程序通过execve()替换了。

详细的过程如下：**sudo进程运行在当前namespace中，它将fork一个新进程来运行unshare程序，unshare程序加载完成后，将创建一个新的uts namespace，unshare进程自身将加入到这个uts namespace中，unshare进程内部再exec加载/bin/bash，于是unshare进程被替换为/bin/bash进程，/bin/bash进程也将运行在uts namespace中**。

![](/img/virtual/2020_09_01_1598952844257.png)

当namespace中的/bin/bash进程退出，该namespace中将没有任何进程，该namespace将自动销毁。注意，在默认情况下，namespace中必须要有至少一个进程，否则将被自动被销毁。但也有一些手段可以让namespace持久化，即使已经没有任何进程在其中运行。

如果在ns1中再创建一个namespace ns2，这个ns2初始时将共享ns1的其他资源并拷贝ns1的主机名资源，其初始主机名也为ns1。

```bash
$ sudo unshare -u /bin/bash    # 在root namespace环境下创建一个namespace
root@longshuai-vm:/home/longshuai# hostname ns1 # 修改主机名为ns1
root@longshuai-vm:/home/longshuai# hostname
ns1

# 在ns1中创建一个namespace
############ 注意没有sudo
root@longshuai-vm:/home/longshuai# unshare -u /bin/bash 
root@ns1:/home/longshuai# hostname    # 初始主机名拷贝自上级namespace的主机名ns1
ns1
root@ns1:/home/longshuai# hostname ns2
root@ns1:/home/longshuai# hostname  # 修改主机名为ns2
ns2
root@ns1:/home/longshuai# exit
exit

root@longshuai-vm:/home/longshuai# hostname  # ns2修改主机名不影响ns1
ns1
root@longshuai-vm:/home/longshuai# exit
exit

[~]->$ hostname      # ns1修改主机名不影响root namespace
longshuai-vm
```

注意，即使root namespace当前用户为longshuai，但因为使用了sudo创建ns1，进入ns1后其用户名为root，所以在ns1中执行unshare命令创建新的namespace不需要再使用sudo。

```bash
$ echo $USER      # 当前root namespace的用户为longshuai
longshuai

$ sudo unshare -u /bin/bash
root@longshuai-vm:/home/longshuai# echo $USER  # ns中的用户名变为root
root
root@longshuai-vm:/home/longshuai# id;echo $HOME;echo ~
uid=0(root) gid=0(root) groups=0(root)
/root
/root
```




