---
title: Linux终端类型
p: linux/linux_tty.md
date: 2020-07-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux终端类型

unix是一个多用户多任务的操作系统。早期电脑昂贵，所以当时使用便宜的设备连接到电脑上(当时还没有键盘和显示器，使用纸带和卡片来输入输出)来使用操作系统，这个便宜的设备就是终端，也可以认为终端是一种控制台。所以可以认为电脑本身是console终端，便宜的连接设备是物理终端pty。

Linux是类unix系统，所以也继承了终端的特性。但是后来电脑逐渐便宜，也出现了显示器和键盘，所以可以使用键盘当作输入终端，显示器当作输出终端，这些终端就是虚拟终端，虚拟终端其实就是虚拟控制台，或者说是一个虚拟设备。

Linux提供了很多种虚拟终端，使用ttyN表示，使用`Ctrl+Alt+F[1-6]`可以进行虚拟终端的切换，这些终端设备记录在/dev/目录下。

```
[root@xuexi ~]# ls /dev/tty
tty    tty12  tty17  tty21  tty26  tty30  tty35
tty4   tty44  tty49  tty53  tty58  tty62  ttyS0 
tty0   tty13  tty18  tty22  tty27  tty31  tty36
tty40  tty45  tty5   tty54  tty59  tty63  ttyS1 
tty1   tty14  tty19  tty23  tty28  tty32  tty37
tty41  tty46  tty50  tty55  tty6   tty7   ttyS2 
tty10  tty15  tty2   tty24  tty29  tty33  tty38
tty42  tty47  tty51  tty56  tty60  tty8   ttyS3 
tty11  tty16  tty20  tty25  tty3   tty34  tty39
tty43  tty48  tty52  tty57  tty61  tty9
```

tty加上数值的就是虚拟终端，CTRL+ALT+F1表示切换到tty1终端，ctrl+alt+f2表示切换到tty2终端，一般Linux上只提供了`ctrl+alt+f[1-6]`这6个终端之间切换的功能。两个特殊的终端是tty和tty0，**tty表示当前正在使用的终端，tty0表示当前的虚拟终端**。还有ttySN，这些表示串行终端。

还有从ssh或telnet等从网络连接到电脑上的终端，或者从图形虚拟终端打开的命令行终端，都称为伪终端，使用pts/N表示，对应的设备为/dev/pts目录下的数值N文件。

```
[root@xuexi ~]# ls /dev/pts/
0     ptmx 
```

0表示第一个伪终端，1表示第二个伪终端。

伪终端和其它所有终端的管理方式都不一样，它是通过连接电脑的程序管理的，例如ssh连接则由ssh负责申请伪终端资源，并要求输入用户名和密码。如果ssh连接进程被杀，则此伪终端也相应的退出。

另外，有些身份验证的程序并非一定会为连接从程序分配终端，例如执行sudo ssh时，sudo就不一定会为ssh分配伪终端。

在现代Linux上，console终端已经和原始的意义不太一样了，其设备映射在/dev/console上，所有内核输出的信息都输出到console终端，而其他用户程序输出的信息则输出到虚拟终端或伪终端。

总结下：

```
/dev/console：控制台终端
/dev/ttyN：虚拟终端，ctrl+alt+f[1-6]切换的就是虚拟终端
/dev/ttySN：串行终端
/dev/pts/N：伪终端，ssh等工具连接过去的活着图形终端下开启的命令行终端就是伪终端。
```