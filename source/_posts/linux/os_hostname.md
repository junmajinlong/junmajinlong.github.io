---
title: centOS7主机名的弯弯绕绕
p: linux/os_hostname.md
date: 2020-09-27 18:20:42
tags: Linux
categories: Linux
---

# CentOS7主机名的弯弯绕绕

在CentOS 6中，修改主机名方式很简单，临时修改主机名使用`hostname`命令，永久修改主机名直接写进文件`/etc/sysconfig/network`中即可。

但在CentOS 7中，主机名就没那么简单了，它涉及了一些弯弯绕绕。

**在CentOS 7中，主机名分3类：static(静态主机名)、pretty(好看、易读的主机名)和transient(短暂临时的)。CentOS 7中和主机名有关的文件为`/etc/hostname`，它是在系统初始化的时候被读取的，并且内核根据它的内容设置transient主机名。**

其中：  
1. static类的主机名就是我们常说的主机名，由`/etc/hostname`文件决定。  
2. transient类的主机名也就是我们常说的临时主机名，它是由内核动态维护的主机名。默认在系统启动的时候会根据`/etc/hostname`文件中的静态主机名进行初始化。  
3. pretty类的主机名是给人看的，它可以提供非标准的主机名，以前版本(例如CentOS 6)没有这功能。它可以包含特殊符号，例如空格。例如将pretty名称命为`MaYun's Host`，这种名称在以前的主机名(即static类主机名)里是不允许存在的。  

`/etc/hostname`文件中的static主机名是瞬时生效的，也是永久生效的。修改后使用`hostname`命令或者`uname -n`直接就可以读取，重启后也按照此文件的主机名进行初始化。

`/etc/hostname`文件没有主机名的时候，在系统启动的时候，内核会将transient初始化为`localhost.localdomain`。

`/etc/sysconfig/network`文件已经失效。

## CentOS 7主机名修改、查看

1. 使用`hostname`命令修改主机名，它修改是transient主机名，即临时生效的主机名。  
2. 直接修改`/etc/hostname`文件，它瞬时生效，重启后也生效(因为内核会根据它初始化transient主机名)。  
3. 使用`nmtui`命令在图形化界面修改主机名。它会直接修改`/etc/hostname`文件，因此也是瞬时生效+永久生效的。  
4. 使用`hostnamectl`命令。它可以修改并查看static、transient或pretty三种主机名。当它修改了static主机名时，会直接写入`/etc/hostname`文件中，因此它也是瞬时生效+永久生效的。


## hostnamectl命令

**1.查看主机名**

```bash
hostnamectl 
#或
hostnamectl status
#或
hostnamectl [--pretty|--static|--transient] status
```

例如，当前主机名为`xuexi.longshuai.com`。
```
[root@xuexi ~]# uname -n
xuexi.longshuai.com
[root@xuexi ~]# hostname name1
[root@xuexi ~]# hostnamectl 
   Static hostname: xuexi.longshuai.com
Transient hostname: name1
         Icon name: computer-vm
           Chassis: vm
        Machine ID: d13bce5e247540a5b5886f2bf8aabb35
           Boot ID: d34a4222469e4f1cbe20c27aca174e10
    Virtualization: vmware
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-327.el7.x86_64
      Architecture: x86-64
```

可以看到使用hostname命令修改主机名后，transient已经改变了。


**2.同时修改3种主机名**

当同时修改了pretty和(static或transient)中的一种时，将取pretty名的简化部分作为static主机名。

```
hostnamectl set-hostname NAME
```

例如：

```
[root@xuexi ~]# hostnamectl set-hostname name2
[root@xuexi ~]# hostname
name2
[root@xuexi ~]# cat /etc/hostname 
name2
[root@xuexi ~]# hostnamectl status
   Static hostname: name2
         Icon name: computer-vm
           Chassis: vm
        Machine ID: d13bce5e247540a5b5886f2bf8aabb35
           Boot ID: d34a4222469e4f1cbe20c27aca174e10
    Virtualization: vmware
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-327.el7.x86_64
      Architecture: x86-64
[root@xuexi ~]# hostnamectl  --pretty

[root@xuexi ~]#
```
可以从结果中看到，只改变了static和transient(内核动态维护的，一定会改变)，而pretty却没设置成功。这是因为这里给出的主机名"name2"是一个符合主机名标准的名称。如果指定一个非标准的主机名，例如包含特殊符号，那么也会设置pretty。

例如：
```
[root@xuexi ~]# hostnamectl set-hostname "name22 name22"
[root@xuexi ~]# hostnamectl
   Static hostname: name22name22
   Pretty hostname: name22 name22
         Icon name: computer-vm
           Chassis: vm
        Machine ID: d13bce5e247540a5b5886f2bf8aabb35
           Boot ID: d34a4222469e4f1cbe20c27aca174e10
    Virtualization: vmware
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-327.el7.x86_64
      Architecture: x86-64
```
pretty hostname已经改变，且static hostname是它的"简化版"。


**3.修改某种类型的主机名**

```
hostnamectl set-name NAME --static
hostnamectl set-name NAME --transient
hostnamectl set-name NAME --pretty
```

用法如上面的例子。

**4.同时修改其中两种名称。**

```
hostnamectl set-name NAME --static --transient
hostnamectl set-name NAME --static --pretty
hostnamectl set-name NAME --transient --pretty
```

用法如上面的例子。但同样注意，当修改了pretty主机名和其他一种时，将取pretty的"简化版"。

**5.修改、查看远程主机的主机名，使用`-H`或`--host`选项。连接基于SSH。**

注意，无法远程修改CentOS 5或6主机名，因为它使用的是systemd类的命令进行修改的。

```
hostnamectl -H [USER@]HOST set-hostname NAME
hostnamectl -H [USER@]HOST status
```

例如，使用root用户连接到192.168.100.59主机上并修改它的主机名。

```
hostnamectl -H root@192.168.100.59 set-hostname hello59
hostnamectl -H root@192.168.100.59 status
```
