---
title: Linux namespace之：user namespace
p: virtual/namespace/user_namespace.md
date: 2020-10-10 17:37:33
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

# 理解user namespace

user namespace涉及namespace的权限和安全问题，是内容最多也最复杂的一种namespace。本文不深入太多理论细节，而是只介绍user namespace机制导致的现象，这样可以足够简单地了解user namespace，也可以控制好篇幅。

user namespace是唯一一种不要求root权限就可以创建的namespace。换句话说，如果是非root用户，可以不使用sudo创建user namespace。

```bash
# --user或-U表示创建user namespace
$ unshare -U /bin/bash

# 非root用户创建其他类型的namespace都需要sudo
## 创建uts namespace
# sudo unshare -u /bin/bash
```

用户A创建user namespace ns1后，在ns1中活动的仍然是用户A。但是，ns1中的用户A有点特殊：它的用户名将变成nobody，UID和GID都为65534，65534对应的用户名和组名分别是nobody、nogroup。
```bash
$ echo $USER
longshuai
$ echo $HOME
/home/longshuai
$ id
uid=1000(longshuai) gid=1000(longshuai) groups=1000(longshuai)
$ readlink /proc/39523/ns/user
user:[4026531837]

# 创建user namespace ns1
$ sudo unshare -U /bin/bash

# 进入了ns1
# $USER看到的用户名仍然为longshuai，家目录也没有变
# 但是uid/gid都变了
# 且whoami和id看到的都是nobody
$ echo $USER
longshuai
$ echo $HOME
/home/longshuai
$ whoami
nobody
$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
$ readlink /proc/39650/ns/user
user:[4026532588]
```

不仅如此，此时ns1中的文件、目录的owner/group也全都是nobody/nogroup。
```bash
$ echo $HOME
/home/longshuai

$ ls -l ~
total 40
-rw-rw-r-- 1 nobody nogroup    0 Oct  9 18:05 ax.log
drwxr-xr-x 2 nobody nogroup 4096 Jun  6 12:51 Desktop
drwxr-xr-x 2 nobody nogroup 4096 Jun  6 12:51 Documents
drwxr-xr-x 2 nobody nogroup 4096 Jun  6 12:51 Downloads
drwxrwxr-x 6 nobody nogroup 4096 Oct  6 10:10 fs
......

$ ls -l / 
total 2097236
lrwxrwxrwx   1 nobody nogroup          7 Jun  6 12:25 bin -> usr/bin
drwxr-xr-x   8 nobody nogroup       4096 Jun 26 14:18 blog
drwxr-xr-x   4 nobody nogroup       4096 Jul 10 09:22 boot
drwxrwxr-x   2 nobody nogroup       4096 Jun  6 12:26 cdrom
drwxr-xr-x  17 nobody nogroup       4120 Oct  5 20:26 dev
drwxr-xr-x 140 nobody nogroup      12288 Oct  4 10:29 etc
drwxr-xr-x   3 nobody nogroup       4096 Jul  2 11:25 home
......

$ ls -l /etc/*.conf
-rw-r--r-- 1 nobody nogroup  3028 Apr 23 15:32 /etc/adduser.conf
-rw-r--r-- 1 nobody nogroup   769 Jan 19  2020 /etc/appstream.conf
-rw-r--r-- 1 nobody nogroup 26916 Mar  4  2020 /etc/brltty.conf
-rw-r--r-- 1 nobody nogroup  5714 Jun  6 13:16 /etc/ca-certificates.conf
-rw-r--r-- 1 nobody nogroup  2969 Aug  3  2019 /etc/debconf.conf
-rw-r--r-- 1 nobody nogroup   604 Sep 16  2018 /etc/deluser.conf
-rw-r--r-- 1 nobody nogroup   685 Feb 14  2020 /etc/e2scrub.conf
......
```

也就是说，从文件、目录的所有者以及所属组来看，当前的用户(nobody)在ns1中是具有对应权限的。但实际上，user namespace中的权限非常受限：**只具备创建user namespace的用户(即longshuai)权限**。
```bash
$ pwd
/home/longshuai
# 在longshuai家目录以及/tmp下创建文件成功
$ touch ~/temp.txt
$ touch /tmp/a.log

# 不能在/etc、/root、/下创建文件
$ touch /etc/hello.conf
touch: cannot touch '/etc/hello.conf': Permission denied
$ touch /root/temp.txt
touch: cannot touch '/root/temp.txt': Permission denied
$ touch /temp.txt     
touch: cannot touch '/temp.txt': Permission denied
```

所以，原namespace中的用户和当前user namespace中的用户是有对应关系的：**默认情况下，创建user namespace的用户映射为新建user namespace中的用户**。

其实，用户可以指定如何映射用户，而且这通常也是创建user namespace后应该做的第一件事。

UID和GID映射涉及到的文件分别为`/proc/<PID>/uid_map`和`/proc/<PID>/gid_map`，它们的格式一样，都只能写入一次，第二次写入将报错。在某些老版本(内核4.14及之前的版本)里，这两个文件可能不存在，且最多只能写入5行。

这两个文件的格式为：
```
ID_child ID_parent length
```
以`/proc/13333/uid_map`文件为例，假如向该文件写入`0 1000 500`，这表示将父级user namespace中1000-1500之间的UID逐一映射到PID=13333所在user namespace中0-500的UID。GID的映射方式也一样如此。

尽管这两个map文件的owner和group可能正是当前用户，但是非root用户却无法直接修改该文件。要修改该文件，需要具备cap_setuid和cap_setgid能力(capability)或者直接使用具有最大权限的root用户。
```bash
# 在第一个窗口创建user namespace ns1
$ unshare -U /bin/bash
$ echo $$     # ns1的bash进程号为44101
44101
# 当前用户为longshuai
$ echo $USER
longshuai

# 在第二个窗口修改uid_map和gid_map
# 直接echo '0 1000 500' >/proc/44101/uid_map将报错
# 需使用sudo或者赋予cap_setuid、cap_setgid能力

# 使用sudo设置uid_map，稍后使用capability设置gid_map文件
$ echo '0 1000 500' | sudo tee /proc/44101/uid_map

# 或者如下设置capability
# 表示将setuid和setgid两种能力添加到/bin/bash文件的有效集(e,effective)和许可集(p,permitted)上，还有一个i(inheritable)表示继承集
# +ep表示将能力添加到集合，即提升对应能力
# -ep表示从集合中剔除，即降权
# =ep表示直接设置为此能力
$ sudo setcap cap_setuid,cap_setgid+ep /bin/bash

# 设置后可查看/bin/bash的能力
$ getcap /bin/bash
/bin/bash = cap_setgid,cap_setuid+ep

# 回到第一个窗口的user namespace中，重新执行/bin/bash
# 使得user namespace中的/bin/bash获取已赋予的能力
$ exec bash
# 查看当前bash进程已获取的能力，发现许可位和有效位都设置了值
$ grep 'Cap' /proc/$$/status
CapInh: 0000000000000000
CapPrm: 00000000000000c0  # 这个c位是新增的
CapEff: 00000000000000c0  # 这个c位是新增的
CapBnd: 0000003fffffffff
CapAmb: 0000000000000000

# 回到第二个窗口，在user namespace外修改gid_map
echo '0 1000 500' >/proc/44101/gid_map
```

只要完成了uid和gid的映射，在user namespace中的用户名和组就会改变，其有能力修改的的文件所有者以及所属组也都会改变。

例如上面的示例中是将uid=1000的longshuai用户映射到user namespace中的uid=0(即root)用户。
```bash
# 在 user namespace中
$ id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)

# 有权限操作的文件owner、group也会随之发生改变
$ ls -l /home
total 4
drwxr-xr-x 21 root root 4096 Oct  9 21:57 longshuai

# 无权限操作的文件owner、group不会改变
$ ls -l /etc/*.conf
-rw-r--r-- 1 nobody nogroup ... /etc/adduser.conf
-rw-r--r-- 1 nobody nogroup ... /etc/appstream.conf
-rw-r--r-- 1 nobody nogroup ... /etc/brltty.conf
......

$ ls -l /
total 2097236
lrwxrwxrwx   1 nobody nogroup ... bin -> usr/bin
drwxr-xr-x   8 nobody nogroup ... blog
drwxr-xr-x   4 nobody nogroup ... boot
drwxrwxr-x   2 nobody nogroup ... cdrom
drwxr-xr-x  17 nobody nogroup ... dev
drwxr-xr-x 140 nobody nogroup ... etc
drwxr-xr-x   3 nobody nogroup ... home
......
```

再做一个简单的实验，不要让映射起始UID正好是创建user namespace的用户。

例如，使用UID=1200的xiaofang用户创建user namespace：
```bash
$ sudo useradd -u 1200 -m xiaofang
$ sudo passwd xiaofang
$ su - xiaofang

# uid=1200的xiaofang用户创建user namespace
$ unshare -U /bin/bash
$ echo $$
48441
```

再设置如下映射方式`0 1000 500`：
```bash
# 新建一个会话窗口，在父级user namespace中执行
$ echo '0 1000 500' | sudo tee /proc/48441/uid_map >/dev/null
$ echo '0 1000 500' | sudo tee /proc/48441/gid_map >/dev/null
```

如此设置映射后，user namespace中的用户将变为哪个呢？实际上，谁创建user namespace，谁就是这个user namespace中的活动用户。也就是说，映射之前的UID=1200的xiaofang是user namespace当前的用户。

所以，父级user namespace中UID=1200映射到新的user namespace中就是UID=200：
```bash
# 在新建的user namespace中执行
$ id
uid=200 gid=200 groups=200
```
从id的输出结果看，uid=200的用户不存在，它没有对应的用户名，同理组名也一样不存在。
```bash
# 在新建的user namespace中执行
$ exec bash
groups: cannot find name for group ID 200
```

从这个现象可以分析，在映射用户和组时，最可能也最合理的映射方式是将user namespace的创建者映射为UID=0的root。

因此，对于UID=x的用户创建的user namespace来说，可能需要如下方式映射UID：
```bash
0 x <Length>
```

对于这种最常见的映射方式，unshare提供了一个快捷选项`-r, --map-root-user`，可以帮助用户在创建user namespace时自动将当前用户映射为user namespace内UID=0的root。
```bash
$ unshare -U -r /bin/bash
$ id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
```

当将创建user namespace的用户A映射为root后，在这个新的user namespace中它将具有当前namespace的【所有权】，比如可以直接创建所有类型的namespace。
```bash
$ unshare -U -r /bin/bash

# 映射为root后，可直接创建uts+mount+pid namespace
# 不需要再提权，因为当前就是root
$ unshare -u -p -f -m --mount-proc /bin/bash
```

但实际上这个root权限是非常受限的，因为在和其他user namespace交互时(比如修改父级user namespace的内容)，内核仍然使用用户A来做权限判断。

例如，即使映射为root，也无法修改主机名：
```bash
$ unshare -U -r /bin/bash

$ whoami
root
$ hostname ns1
hostname: you must be root to change the host name
```

之所以在user namespace中已经映射为root仍然无法修改主机名，是因为主机名资源是父级uts namespace的内容。其实，任何一个非user namespace类型的namespace，都有一个与之关联的user namespace，这样才能管理这些user namespace的权限。

比如系统启动后，所有初始的非user namespace的namespace，它们所关联的user namespace就是与之同层次的root user namespace。

再例如，在系统启动后创建的任意非user namespace类型的namespace，它们关联的user namespace都是root uset namespace。

再比如，在user namespace ns1中创建uts namespace ns2，ns2关联的user namespace是ns1。

因此可做出总结：**创建非user namespace类型的namespace时，这些namespace关联的user namespace是创建者所属的user namespace**。

因此，如果想要在user namespace中修改主机名，需要在创建user namespace的时候创建uts namespace，或者在user namespace内部创建uts namespace，这样一来，uts namespace所关联的user namespace就是这个新建的user namespace：
```bash
$ unshare -u -U -r /bin/bash
$ whoami
root
$ hostname ns1    #  修改成功
$ hostname
ns1

# 或者
$ unshare -U -r /bin/bash
$ whoami 
root
$ unshare -u /bin/bash
$ hostname ns1  # 也可以修改成功
$ hostname
ns1
```

最后，用户创建的user namespace中无法挂载块设备，块设备的挂载操作只能在关联了初始root user namespace的namespace中挂载。也就是说，只要某个namespace的祖先有一个是用户创建的user namespace，都将无法挂载块设备。但允许挂载以下类型的文件系统：
```
/proc (since Linux 3.8)
/sys (since Linux 3.8)
devpts (since Linux 3.9)
tmpfs(5) (since Linux 3.9)
ramfs (since Linux 3.9)
mqueue (since Linux 3.9)
bpf (since Linux 4.4)
```






