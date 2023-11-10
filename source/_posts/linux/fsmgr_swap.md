---
title: Linux管理文件系统(4)：创建并使用swap分区
p: linux/fsmgr_swap.md
date: 2019-07-07 11:20:45
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux管理文件系统(4)：创建并使用swap分区


虽说个人电脑上基本已经无需设置swap分区了，但是在服务器上还是应该准备swap分区，以做到有备无患和防止众多【玄学】问题。实际上，即使内存足够，添加swap分区也只有好处，在内存充足时并不会造成低效问题，因此大可不必因网上的【优化】套路而排斥swap分区。

## 查看swap使用情况

```
[root@xuexi ~]# free
             total      used      free     shared    buffers     cached
Mem:       1906488     349376    1557112      200     16920      200200
-/+ buffers/cache:     132256    1774232
Swap:      2097148          0    2097148

[root@xuexi ~]# free -m        # 以MB显示
              total       used       free      shared    buffers     cached
Mem:          1861        341       1520          0         16        195
-/+ buffers/cache:        129       1732     # 这个是真正的可用内存空间
Swap:         2047          0       2047     # 这个是swap空间，发现完全未使用
```

使用mount/lsblk等可以查看出哪个分区在充当swap分区。使用`swapon -s`也可以直接查看出。

```
[root@server2 ~]# swapon -s
Filename    Type        Size    Used    Priority
/dev/sda3   partition   2047996 37064   -1
```

## 添加swap分区

(1).可以新分一个区，在分区时指定其分区的ID号为SWAP类型。

mbr和gpt格式的磁盘上这个ID可能不太一样，不过一般gpt中的格式是在mbr格式的ID后加上两位数的数值，如mbr中swap的类型ID为82，在gpt中则是8200，在mbr中linux filesystem类型的ID为83，在gpt中则为8300，在mbr中lvm的ID为8e，在gpt中为8e00。

(2).格式化为swap分区：mkswap

```
[root@xuexi ~]# mkswap /dev/sdb5
Setting up swapspace version 1, size = 1951096 KiB
no label, UUID=02e5af44-2a16-479d-b689-4e100af6adf5
```

(3).加入swap分区空间(swapon)：

```
[root@xuexi ~]# swapon /dev/sdb5   
[root@xuexi ~]# free -m
             total       used       free     shared    buffers     cached
Mem:          1861        343       1517          0         16        196
-/+ buffers/cache:        131       1730
Swap:         3953          0       3953 
```

(4).取消swap分区空间(swapoff)：

```
[root@xuexi ~]# swapoff /dev/sdb5
[root@xuexi ~]# free -m
             total       used       free     shared    buffers     cached
Mem:          1861        343       1518          0         16        196
-/+ buffers/cache:        130       1731
Swap:            0          0          0
```

(5).开机自动加载swap分区：修改/etc/fstab，加上一行。

```
/dev/sda3    swap    swap    defaults    0    0 
```

有时候没必要为了使用swap分区而创建一个分区，可以直接通过dd命令创建一个空文件，再将其格式化制作成swap分区：

```
dd if=/dev/zero of=/swap bs=1M count=1024
mkswap /swap   # 现在/swap文件就是swap格式的
swapon /swap
echo "/swap  swap swap defaults 0 0" >> /etc/fstab
```

这种方式可以应用在低配的个人云主机上。低配的云主机内存较小，可能连启动一个博客系统都启动不了，而且云主机无法自由添加磁盘或重新分区，使用dd命令正好可以解此问题。

