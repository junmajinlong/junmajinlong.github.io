---
title: Linux namespace概述
p: virtual/namespace/ns_overview.md
date: 2020-10-10 17:37:29
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

本系列文章不介绍关于Linux namespace的全部，只介绍其中重要的一部分，有了基础之后，更多的内容可去参考man手册，man手册的解释非常详细。

# Linux namespace概述

```
# namespace概念和细节相关man文档
man namespaces
man uts_namespaces
man network_namespaces
man ipc_namespaces
man pid_namespaces
man mount_namespaces
man user_namespaces
man time_namespaces
man cgroup_namespaces

# namespace管理工具
man unshare     # 创建namespace
man nscreate    # 创建namespace，老版本的内核没有该工具
man nsenter     # 切换namespace
man lsns        # 查看当前已创建的namespace
```

## Linux namespace介绍

操作系统通过虚拟内存技术，使得每个用户进程都认为自己拥有所有的物理内存，这是操作系统对内存的虚拟化。操作系统通过分时调度系统，每个进程都能被【公平地】调度执行，即每个进程都能获取到CPU，使得每个进程都认为自己在进程活动期间拥有所有的CPU时间，这是操作系统对CPU的虚拟化。

从这两种虚拟化方式可推知，当使用某种虚拟化技术去管理进程时，进程会认为自己拥有某种物理资源的全部。

虚拟内存和分时系统均是对**物理资源**进行虚拟化，其实操作系统中还有很多**非物理资源**，比如用户权限系统资源、网络协议栈资源、文件系统挂载路径资源等。通过Linux的namespace功能，可以对这些非物理全局资源进行虚拟化。

Linux namespace是在当前运行的系统环境中创建(隔离)另一个进程的运行环境出来，并在此运行环境中将一些必要的系统全局资源进行【虚拟化】。进程可以运行在指定的namespace中，因此，namespace中的每个进程都认为自己拥有所有这些虚拟化的全局资源。

目前，Linux已经支持8种全局资源的虚拟化(每种资源都是随着Linux内核版本的迭代而逐渐加入的，因此有些内核版本可能不具备某种namespace)：  
- cgroup namespace：该namespace可单独管理自己的cgroup  
- ipc namespace：该namespace有自己的IPC，比如共享内存、信号量等  
- network namespace：该namespace有自己的网络资源，包括网络协议栈、网络设备、路由表、防火墙、端口等  
- mount namespace：该namespace有自己的挂载信息，即拥有独立的目录层次  
- pid namespace：该namespace有自己的进程号，使得namespace中的进程PID单独编号，比如可以PID=1  
- time namespace：该namespace有自己的启动时间点信息和单调时间，比如可设置某个namespace的开机时间点为1年前启动，再比如不同的namespace创建后可能流逝的时间不一样  
- user namespace：该namespace有自己的用户权限管理机制(比如独立的UID/GID)，使得namespace更安全  
- uts namespace：该namepsace有自己的主机信息，包括主机名(hostname)、NIS domain name  

用户可以同时创建具有多种资源类型的namespace，比如创建一个同时具有uts、pid和user的namespace。

## 理解Linux namespace

用户可以创建指定类型的namespace并将程序放入该namespace中运行，这表示从当前的系统运行环境中隔离一个进程的运行环境，在此namespace中运行的进程将认为自己享有该namespace中的独立资源。

实际上，即使用户没有手动创建Linux namespace，Linux系统开机后也会创建一个默认的namespace，称为**root namespace**，所有进程默认都运行在root namespace中，每个进程都认为自己拥有该namespace中的所有系统全局资源。

回顾一下Linux的开机启动流程，内核加载成功后将初始化系统运行环境，这个运行环境就是root namespace环境，系统运行环境初始化完成后，便可以认为操作系统已经开始工作了。

**每一个namespace都基于当前内核**，无论是默认的root namespace还是用户创建的每一个namespace，都基于当前内核工作。所以可以认为namespace是内核加载后启动的一个特殊系统环境，用户进程可以在此环境中独立享用资源。更严格地说，**root namespace直接基于内核，而用户创建的namespace运行环境基于当前所在的namespace**。之所以用户创建的namespace不直接基于内核环境，是因为每一个namespace可能都会修改某些运行时内核参数。

比如，用户创建的uts namespace1中修改了主机名为ns1，然后在namespace1中创建uts namespace2时，namespace2默认**将共享namespace1的其他资源并拷贝namespace1的主机名资源**，因此namespace2的主机名初始时也是ns1。当然，namespace2是隔离的，可以修改其主机名为ns2，这不会影响其他namespace，修改后，将只有namespace2中的进程能看到其主机名为ns2。

![](/img/virtual/2020_09_01_1598952786662.png)

可以通过如下方式查看某个进程运行在哪一个namespace中，即该进程享有的独立资源来自于哪一个namespace。
```bash
# ls -l /proc/<PID>/ns
$ ls -l /proc/$$/ns | awk '{print $1,$(NF-2),$(NF-1),$NF}'
lrwxrwxrwx  cgroup            ->  cgroup:[4026531835]
lrwxrwxrwx  ipc               ->  ipc:[4026531839]
lrwxrwxrwx  mnt               ->  mnt:[4026531840]
lrwxrwxrwx  net               ->  net:[4026531992]
lrwxrwxrwx  pid               ->  pid:[4026531836]
lrwxrwxrwx  pid_for_children  ->  pid:[4026531836]
lrwxrwxrwx  user              ->  user:[4026531837]
lrwxrwxrwx  uts               ->  uts:[4026531838]

$ sudo ls -l /proc/1/ns | awk '{print $1,$(NF-2),$(NF-1),$NF}'
lrwxrwxrwx  cgroup            ->  cgroup:[4026531835]
lrwxrwxrwx  ipc               ->  ipc:[4026531839]
lrwxrwxrwx  mnt               ->  mnt:[4026531840]
lrwxrwxrwx  net               ->  net:[4026531992]
lrwxrwxrwx  pid               ->  pid:[4026531836]
lrwxrwxrwx  pid_for_children  ->  pid:[4026531836]
lrwxrwxrwx  user              ->  user:[4026531837]
lrwxrwxrwx  uts               ->  uts:[4026531838]
```

这些文件表示当前进程打开的namespace资源，每一个文件都是一个软链接，所指向的文件是一串格式特殊的名称。冒号后面中括号内的数值表示该namespace的inode。如果不同进程的namespace inode相同，说明这些进程属于同一个namespace。

从结果上来看，每个进程都运行在多个namespace中，且pid=1和pid=$$(当前Shell进程)两个进程的namespace完全一样，说明它们运行在相同的环境下(root namespace)。