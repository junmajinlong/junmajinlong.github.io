---
title: 使用LVM
p: linux/lvm.md
date: 2019-07-07 12:29:43
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 使用LVM

## lvm相关的概念和机制

LVM(Logical Volume Manager)可以让分区变得弹性，可以随时随地的扩大和缩小分区大小。

lvm需要使用的软件包为lvm2，一般在CentOS发行版中都已经预安装了。

下面是LVM相关的一些基本概念：

- **PV(Physical Volume)即物理卷**

   硬盘分区后(还未格式化为文件系统)使用pvcreate命令可以将分区创建为pv，要求分区的system ID为8e，即为LVM格式的系统标识符。

- **VG(Volume Group)即卷组**

   将多个PV组合起来，使用vgcreate命令创建成卷组，这样卷组包含了多个PV就比较大了，相当于重新整合了多个分区后得到的磁盘。虽然VG是整合多个PV的，但是创建VG时会将VG所有的空间根据指定的PE大小划分为多个PE，在LVM模式下的存储都以PE为单元，类似于文件系统的Block。

- **PE(Physical Extend)**

   PE是VG中的存储单元。实际存储的数据都是存储在这里面的。

- **LV(Logical Volume)**

   VG相当于整合过的硬盘，那么LV就相当于分区，只不过该分区是通过VG来划分的。VG中有很多PE单元，可以指定将多少个PE划分给一个LV，也可以直接指定大小(如多少兆)来划分。划分为LV之后就相当于划分了分区，只需再对LV进行格式化即可变成普通的文件系统。

   通俗地讲，非LVM管理的分区步骤是将硬盘分区，然后将分区格式化为文件系统。而使用LVM，则是在硬盘分区为特定的LVM标识符的分区后将其转变为LVM可管理的PV，其实PV仍然类似于分区，然后将几个PV整合为类似于磁盘的VG，最后划分VG为LV，此时LV就成了LVM可管理的分区，只需再对其格式化即可成为文件系统。

- **LE(logical extent)**

   PE是物理存储单元，而LE则是逻辑存储单元，也即为lv中的逻辑存储单元，和pe的大小是一样的。从vg中划分lv，实际上是从vg中划分vg中的pe，只不过划分lv后它不再称为pe，而是成为le。


LVM之所以能够伸缩容量，其原因就在于能够将LV里空闲的PE移出，或向LV中添加空闲的PE。

## LVM的写入机制

LV是从VG中划分出来的，LV中的PE很可能来自于多个PV。在向LV存储数据时，有多种存储机制，其中两种是：  
- 线性模式(linear)：先写完来自于同一个PV的PE，再写来自于下一个PV的PE。
- 条带模式(striped)：一份数据拆分成多份，分别写入该LV对应的每个PV中，所以读写性能较好，类似于RAID 0。


尽管striped读写性能较好也不建议使用该模式，因为lvm的着重点在于弹性容量扩展而非性能，要实现性能应该使用RAID来实现，而且使用striped模式时要进行容量的扩展和收缩将比较麻烦。默认的是使用线性模式。

## LVM实现图解

![](/img/linux/wps12.jpg) 

## LVM的实现

以上图为例。

首先需要将/dev/sdb下的分区`/dev/sdb{1,2,3,5}`修改为LVM格式的标识符(/dev/sdb4在后面扩容实验中使用)。mbr格式下标识符是8e，gpt格式下是8300。

以下是gpt分区表格式的部分分区信息。
```
[root@server2 ~]# gdisk -l /dev/sdb
GPT fdisk (gdisk) version 0.8.6
 
Number  Start (sector)    End (sector)  Size       Code 
   1            2048        20000767   9.5 GiB     8E00 
   2        20000768        26292223   3.0 GiB     8E00 
   3        26292224        29296639   1.4 GiB     8E00 
   4        29296640        33202175   1.9 GiB     8300 
   5        33202176        37109759   1.9 GiB     8E00 
```

### 管理PV

管理PV有几个命令：pvscan、pvdisplay、pvcreate、pvremove和pvmove。

命令很简单，基本都不需要任何选项。

| 功能               | 命令      |
| ------------------ | --------- |
| 创建PV             | pvcreate  |
| 扫描并列出所有的pv | pvscan    |
| 列出pv属性信息     | pvdisplay |
| 移除pv             | pvremove  |
| 移动pv中的数据     | pvmove    |

其中：  
- pvscan搜索目前有哪些pv，扫描之后将结果放在缓存中；  
- pvdisplay会显示每个pv的详细信息，如PV name和pv size以及所属的VG等。  

直接将上述`/dev/sdb{1,2,3,5}`创建为pv。
```
#  -y选项用于自动回答yes
[root@server2 ~]# pvcreate -y /dev/sdb{1,2,3,5}
  Wiping ext4 signature on /dev/sdb1.
  Physical volume "/dev/sdb1" successfully created
  Wiping ext2 signature on /dev/sdb2.
  Physical volume "/dev/sdb2" successfully created
  Physical volume "/dev/sdb3" successfully created
  Physical volume "/dev/sdb5" successfully created
```

使用pvscan来查看哪些pv和基本属性。
```
[root@server2 ~]# pvscan
  PV /dev/sdb1         lvm2 [9.54 GiB]
  PV /dev/sdb2         lvm2 [3.00 GiB]
  PV /dev/sdb5         lvm2 [1.86 GiB]
  PV /dev/sdb3         lvm2 [1.43 GiB]
  Total: 4 [15.83 GiB] / in use: 0 [0   ] / in no VG: 4 [15.83 GiB]
```

注意最后一行显示的是【pv的总容量/已使用的pv容量/空闲的pv容量】

使用pvdisplay查看其中一个pv的属性信息。
```
[root@server2 ~]# pvdisplay /dev/sdb1
  "/dev/sdb1" is a new physical volume of "9.54 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb1
  VG Name               
  PV Size               9.54 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               fRChUf-CL8d-2UwC-d94R-xa8a-MRYa-yvgFJ9
```
pvdisplay还有一个很重要的选项`-m`，可以查看该设备中PE的使用分布图。以下是某次显示结果。
```
[root@server2 ~]# pvdisplay -m /dev/sdb2
  --- Physical volume ---
  PV Name               /dev/sdb2
  VG Name               firstvg
  PV Size               3.00 GiB / not usable 16.00 MiB
  Allocatable           yes 
  PE Size               16.00 MiB
  Total PE              191
  Free PE               100
  Allocated PE          91
  PV UUID               uVgv3q-ANyy-02M1-wmGf-zmFR-Y16y-qLgNMV
   
  --- Physical Segments ---
    Physical extent 0 to 0:  # 说明第0个PE正被使用。PV中PE的序号是从0开始编号的
    Logical volume      /dev/firstvg/first_lv
    Logical extents     450 to 450  # 该PE在LV中的第450个LE位置上
    Physical extent 1 to 100:       # 说明/dev/sdb2中1-100序号的PE空闲未被使用
    FREE
  Physical extent 101 to 190:  # 说明101-190的PE正使用，其在LV中的位置是551-640
    Logical volume      /dev/firstvg/first_lv
    Logical extents     551 to 640   
```

知道了PE的分布，就可以轻松地使用pvmove命令在设备之间进行PE数据的移动。具体关于pvmove的用法，见[收缩lvm磁盘](#shrink_lvm)部分。

再测试pvremove，移除/dev/sdb5，然后将其添加回pv。
```
[root@server2 ~]# pvremove /dev/sdb5
  Labels on physical volume "/dev/sdb5" successfully wiped

[root@server2 ~]# pvcreate /dev/sdb5
  Physical volume "/dev/sdb5" successfully created
```

### 管理VG

管理VG也有几个命令。

| 功能               | 命令      |
| ------------------ | --------- |
| 创建VG             | vgcreate  |
| 扫描并列出所有的vg | vgscan    |
| 列出vg属性信息     | vgdisplay |
| 移除vg，即删除vg   | vgremove  |
| 从vg中移除pv       | vgreduce  |
| 将pv添加到vg中     | vgextend  |
| 修改vg属性         | vgchange  |

其中：
- vgscan搜寻有几个vg并显示vg的基本属性
- vgcreate是创建vg
- vgdisplay是列出vg的详细信息
- vgremove是删除整个vg
- vgextend用于扩展vg即将pv添加到vg中
- vgreduce是将pv移除出vg
- vgchange用于改变vg的属性，如修改vg的状态为激活状态或未激活状态

创建一个vg，并将上述4个`pv /dev/sdb{1,2,3,5}`都添加到该vg中。注意vg是需要命名的，vg可以等同于磁盘的层次，而磁盘是有名称的，如/dev/sdb，/dev/sdc等。同时创建vg时可以使用-s选项指定pe的大小，如果不指定默认为4M。
```
[root@server2 ~]# vgcreate -s 16M firstvg /dev/sdb{1,2,3,5}
  Volume group "firstvg" successfully created 
```

此处创建的vg名称为firstvg，指定pe大小为16M。创建vg后，是很难再修改pe大小的，只有空数据的vg可以修改，但这样还不如重新创建vg。

注意，lvm1中每个vg中只能有65534个pe，所以指定pe的大小能改变每个vg的最大容量。但在lvm2中已经没有该限制了，而现在说的lvm一般都指lvm2，这也是默认的版本。

创建了vg实际上是在/dev目录下管理了一个vg目录/dev/firstvg，不过只有在创建了lv该目录才会被创建，而该vg中创建lv，将会在该目录下生成链接文件指向/dev/dm设备。

再看看vgscan和vgdisplay。
```
[root@server2 ~]# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "firstvg" using metadata type lvm2

[root@server2 ~]# vgdisplay firstvg  
  --- Volume group ---
  VG Name               firstvg
  System ID             
  Format                lvm2
  Metadata Areas        4
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                4
  Act PV                4
  VG Size               15.80 GiB
  PE Size               16.00 MiB
  Total PE              1011
  Alloc PE / Size       0 / 0   
  Free  PE / Size       1011 / 15.80 GiB
  VG UUID               GLwZTC-zUj9-mKas-CJ5m-Xf91-5Vqu-oEiJGj
```

从vg中移除一个pv，如/dev/sdb5，再vgdisplay，发现pv少了一个，pe相应的也减少了。
```
[root@server2 ~]# vgreduce firstvg /dev/sdb5
  Removed "/dev/sdb5" from volume group "firstvg"

[root@server2 ~]# vgdisplay firstvg 
  --- Volume group ---
  VG Name               firstvg
  System ID             
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               13.94 GiB
  PE Size               16.00 MiB
  Total PE              892
  Alloc PE / Size       0 / 0   
  Free  PE / Size       892 / 13.94 GiB
  VG UUID               GLwZTC-zUj9-mKas-CJ5m-Xf91-5Vqu-oEiJGj 
```

再将/dev/sdb5加入vg。
```
[root@server2 ~]# vgextend firstvg /dev/sdb5 
  Volume group "firstvg" successfully extended
```

vgchange用于设置卷组的活动状态，卷组的激活状态主要影响的是lv。使用-a选项来设置。

将firstvg设置为活动状态(active yes)。
```
vgchange -a y firstvg
```
将firstvg设置为非激活状态(active no)。
```
vgchange -a n firstvg
```

### 管理LV

有了vg之后就可以根据vg进行分区，即创建LV。管理lv也有类似的一些命令。

| 功能               | 命令               |
| ------------------ | ------------------ |
| 创建LV             | lvcreate           |
| 扫描并列出所有的lv | lvscan             |
| 列出lv属性信息     | lvdisplay          |
| 移除lv，即删除lv   | lvremove           |
| 缩小lv容量         | lvreduce(lvresize) |
| 增大lv容量         | lvextend(lvresize) |
| 改变lv容量         | lvresize           |

对于lvcreate命令有几个选项：
```
lvcreate {-L size(M/G) | -l PEnum} -n lv_name vg_name
-L：根据大小来创建lv，即分配多大空间给此lv
-l：根据PE的数量来创建lv，即分配多少个pe给此lv
-n：指定lv的名称
```

前面创建的vg有1011个PE，总容量为15.8G。
```
[root@server2 ~]# vgdisplay | grep PE
  PE Size               16.00 MiB
  Total PE              1011
  Alloc PE / Size       0 / 0   
  Free  PE / Size       1011 / 15.80 GiB
```

使用-L和-l分别创建名称为first_lv和sec_lv的lv。
```
[root@server2 ~]# lvcreate -L 5G -n first_lv firstvg
  Logical volume "first_lv" created.

[root@server2 ~]# lvcreate -l 160 -n sec_lv firstvg     
  Logical volume "sec_lv" created.
```

创建lv后，将在/dev/firstvg目录中创建对应lv名称的软链接文件，同时也在/dev/mapper目录下创建链接文件，它们都指向/dev/dm设备。
```
[root@server2 ~]# ls -l /dev/firstvg/ 
total 0
lrwxrwxrwx 1 root root 7 Jun  9 23:41 first_lv -> ../dm-0
lrwxrwxrwx 1 root root 7 Jun  9 23:42 sec_lv -> ../dm-1

[root@server2 ~]# ll /dev/mapper/
total 0
crw------- 1 root root 10, 236 Jun  6 02:44 control
lrwxrwxrwx 1 root root       7 Jun  9 23:41 firstvg-first_lv -> ../dm-0
lrwxrwxrwx 1 root root       7 Jun  9 23:42 firstvg-sec_lv -> ../dm-1
```

使用lvscan和lvdisplay查看lv信息。需要注意的是，如果lvdisplay要显示某一个指定的lv，需要指定其全路径，而不能简单的指定lv名，当然如果不指定任何参数将显示所有lv的信息。
```
[root@server2 ~]# lvscan
  ACTIVE            '/dev/firstvg/first_lv' [5.00 GiB] inherit
  ACTIVE            '/dev/firstvg/sec_lv' [2.50 GiB] inherit

[root@server2 ~]# lvdisplay /dev/firstvg/first_lv 
  --- Logical volume ---
  LV Path                /dev/firstvg/first_lv
  LV Name                first_lv
  VG Name                firstvg
  LV UUID                f3cRXJ-vucN-aAw3-HRbX-Fhnq-mW6c-kmL7WA
  LV Write Access        read/write
  LV Creation host, time server2.longshuai.com, 2017-06-09 23:41:42 +0800
  LV Status              available
  # open                 0
  LV Size                5.00 GiB
  Current LE             320
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```

关于lv其他的命令留在后文lvm扩容和缩减的部分进行演示。

### 格式化lv为文件系统

再对lv进行格式化，即可形成文件系统，然后进行挂载使用。
```
[root@server2 ~]# mke2fs -t ext4 /dev/firstvg/first_lv  
```
对于lv格式化的文件系统类型如何查看？

可以先挂载再查看。
```
[root@server2 ~]# mount /dev/firstvg/first_lv /mnt

[root@server2 ~]# mount | grep /mnt
/dev/mapper/firstvg-first_lv on /mnt type ext4 (rw,relatime,data=ordered) 
```
也可以使用`file -s`查看，但由于/dev/firstvg和/dev/mapper下的lv都是链接到/dev/下块设备的链接文件，所以只能对块设备进行查看，否则查看的结果也仅仅只是个链接文件类型。
```
[root@server2 ~]# file -s /dev/dm-0
/dev/dm-0: Linux rev 1.0 ext4 filesystem data, UUID=f2a3b608-f4e9-431b-8c34-9c75eaf7d3b5 (needs journal recovery) (extents) (64bit) (large files) (huge files) 
```

再去看看/dev/sdb的情况。
```
[root@server2 ~]# lsblk -f /dev/sdb
NAME                   FSTYPE  LABEL UUID                                   MOUNTPOINT
sdb
├─sdb1               LVM2_member     fRChUf-CL8d-2UwC-d94R-xa8a-MRYa-yvgFJ9
│ ├─firstvg-first_lv ext4            f2a3b608-f4e9-431b-8c34-9c75eaf7d3b5/mnt
│ └─firstvg-sec_lv
├─sdb2               LVM2_member     uVgv3q-ANyy-02M1-wmGf-zmFR-Y16y-qLgNMV
├─sdb3               LVM2_member     L1byov-fbjK-M48t-Uabz-Ljn8-Q74C-ncdv8h 
├─sdb4
└─sdb5               LVM2_member     Lae2vc-VfyS-QoNS-rz2h-IXUv-xKQc-Q6YCxQ
```

## 对lvm磁盘扩容

lvm最大的优势就是其可伸缩性，而其伸缩性又更偏重于扩容，这是使用lvm的最大原因。

扩容的实质是将vg中空闲的pe添加到lv中，所以只要vg中有空闲的pe，就可以进行扩容，即使没有空闲的pe，也可以添加pv，将pv加入到vg中增加空闲pe。

扩容的两个关键步骤如下：

- (1).使用lvextend或者lvresize添加更多的pe或容量到lv中
- (2).使用resize2fs命令(xfs则使用xfs_growfs)将lv增加后的容量增加到对应的文件系统中(此过程是修改文件系统而非LVM内容)

例如，将一直没用到的/dev/sdb4作为first_lv的扩容来源。首先将/dev/sdb4创建成pv，加入到firstvg中。
```
[root@server2 ~]# parted /dev/sdb toggle 4 lvm
[root@server2 ~]# pvcreate /dev/sdb4
[root@server2 ~]# vgextend firstvg /dev/sdb4
```

查看firstvg中 空闲的pe数量。
```
[root@server2 ~]# vgdisplay firstvg  | grep -i pe
  Open LV               1
  PE Size               16.00 MiB
  Total PE              1130
  Alloc PE / Size       480 / 7.50 GiB
  Free  PE / Size       650 / 10.16 GiB
```

现在vg中有650个PE共10.16G容量可用。将其全部添加到first_lv中，有两种方式添加：按容量大小添加和按PE数量添加。
```
[root@server2 ~]# umount /dev/firstvg/first_lv
[root@server2 ~]# lvextend -L +5G /dev/firstvg/first_lv   # 按容量大小添加
[root@server2 ~]# vgdisplay firstvg  | grep -i pe
  Open LV               1
  PE Size               16.00 MiB
  Total PE              1130
  Alloc PE / Size       800 / 12.50 GiB
  Free  PE / Size       330 / 5.16 GiB

[root@server2 ~]# lvextend -l +330 /dev/firstvg/first_lv  # 按PE数量添加
[root@server2 ~]# lvscan
  ACTIVE            '/dev/firstvg/first_lv' [15.16 GiB] inherit
  ACTIVE            '/dev/firstvg/sec_lv' [2.50 GiB] inherit 
```

也可以使用lvresize来增加lv的容量方法和lvextend一样。如：
```
lvresize -L +5G /dev/firstvg/first_lv
lvresize -l +330 /dev/firstvg/first_lv
```

将first_lv挂载，查看该lv对应文件系统的容量。
```
[root@server2 ~]# mount /dev/mapper/firstvg-first_lv /mnt
[root@server2 ~]# df -hT /mnt
Filesystem                   Type  Size  Used Avail Use% Mounted on
/dev/mapper/firstvg-first_lv ext4  4.8G   20M  4.6G   1% /mnt 
```

发现容量并没有增加，为什么呢？因为只是lv的容量增加了，而文件系统的容量却没有增加。所以使用resize2fs工具来改变ext文件系统的大小，如果是xfs文件系统，则使用xfs_growfs。

首先简单看下resize2fs工具的使用说明。

![](/img/linux/wps13.jpg) 

可见，该工具可用于增大和缩减已卸载的设备对应的文件系统大小，对于linux 2.6内核之后的版本，还支持在线resize而无需卸载，但在实验过程中好像不支持在线缩减，只能先卸载。

一般无需使用选项，直接使用resize2fs device的方式即可，如果失败则尝试使用-f选项强制改变大小。
```
[root@server2 ~]# resize2fs  /dev/firstvg/first_lv 
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/firstvg/first_lv is mounted on /mnt; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/firstvg/first_lv is now 3973120 blocks long.

[root@server2 ~]# df -hT | grep -i /mnt
/dev/mapper/firstvg-first_lv ext4       15G   25M   15G   1% /mnt 
```

再查看，size已经变为15G了。

<a name="shrink_lvm"></a>
## 收缩lvm磁盘

不用考虑收缩的功能，且xfs文件系统也不支持收缩。不过，看看如何收缩，可以加深lvm的理解。

目前first_lv的容量为15.16G。
```
[root@server2 ~]# lvscan
  ACTIVE            '/dev/firstvg/first_lv' [15.16 GiB] inherit
  ACTIVE            '/dev/firstvg/sec_lv' [2.50 GiB] inherit
```

而pv的使用情况则如下：
```
[root@server2 ~]# pvscan
  PV /dev/sdb1   VG firstvg   lvm2 [9.53 GiB / 0    free]
  PV /dev/sdb2   VG firstvg   lvm2 [2.98 GiB / 0    free]
  PV /dev/sdb3   VG firstvg   lvm2 [1.42 GiB / 0    free]
  PV /dev/sdb5   VG firstvg   lvm2 [1.86 GiB / 0    free]
  PV /dev/sdb4   VG firstvg   lvm2 [1.86 GiB / 0    free]
  Total: 5 [17.66 GiB] / in use: 5 [17.66 GiB] / in no VG: 0 [0   ]
```
如果想回收/dev/sdb2的2.98G呢？收缩的步骤和扩容的步骤相反。

(1).首先卸载设备并使用resize2fs收缩文件系统的容量为目标大小

这里要收缩2.98G，原有15.16G，所以文件系统的目标容量为12.18G约算做12470M(因为resize2fs不能接受小数点的size参数，所以换算成整数，也可以直接算为12G)，计算大小时应尽量多给出一点点的容量，所以此处算作收缩3.18G，目标12G。

```
[root@server2 ~]# umount /mnt                     

[root@server2 ~]# resize2fs /dev/firstvg/first_lv 12G
resize2fs 1.42.9 (28-Dec-2013)
Please run 'e2fsck -f /dev/firstvg/first_lv' first.
```
提示需要先运行`e2fsck -f /dev/Myvg/first_lv`，主要是为了检查是否修改后的大小会影响数据。

```
[root@server2 ~]# e2fsck -f /dev/firstvg/first_lv
e2fsck 1.42.9 (28-Dec-2013)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/firstvg/first_lv: 11/999424 files (0.0% non-contiguous), 101892/3973120 blocks

[root@server2 ~]# resize2fs /dev/firstvg/first_lv 12G
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/firstvg/first_lv to 3145728 (4k) blocks.
The filesystem on /dev/firstvg/first_lv is now 3145728 blocks long.
```

(2).再收缩lv。可以直接使用`-L`指定收缩容量，也可以使用`-l`指定收缩的PE数量。

例如此处使用-L来收缩。
```
[root@server2 ~]# lvreduce -L -3G /dev/firstvg/first_lv  
  Rounding size to boundary between physical extents: 2.99 GiB
  WARNING: Reducing active logical volume to 12.16 GiB
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce first_lv? [y/n]: y
  Size of logical volume firstvg/first_lv changed from 15.16 GiB (970 extents) to 12.16 GiB (779 extents).
  Logical volume first_lv successfully resized.
```
发现有警告：可能会损毁你的数据。如果在该lv下存储的实际数据大于收缩后的容量，那么肯定会损毁一部分数据，但是如果存储的数据小于收缩后的容量，那么就不会损毁任何数据，这是lvm无损修改分区大小的优点。此处由于在lv下完全没有存储数据，所以无需担心会损毁，直接y确定reduce。

(3).pvmove移动PE

上面的过程已经释放了3G大小的PE，但是这部分PE来源于何处？是否可以判断此时能否移除/dev/sdb2？

首先查看哪些PV上有空闲的PE。
```
[root@server2 ~]# pvdisplay | grep 'PV Name\|Free'
  PV Name               /dev/sdb1
  Free PE               0
  PV Name               /dev/sdb2
  Free PE               0
  PV Name               /dev/sdb3
  Free PE               91
  PV Name               /dev/sdb5
  Free PE               0
  PV Name               /dev/sdb4
  Free PE               100
```
可见，此时空闲的PE分布在/dev/sdb4和/dev/sdb3上，/dev/sdb2并不能卸载。

那么现在需要做的事情是将/dev/sdb2上的PE移到其他设备上，使/dev/sdb2空闲出来。使用pvmove命令，该命令的执行内部会涉及到不少步骤，所以可能会消耗点时间。

因为/dev/sdb4上空闲了100个PE，所以从/dev/sdb2上移动100个PE到/dev/sdb4上。
```
[root@server2 ~]# pvmove /dev/sdb2:0-99 /dev/sdb4
```
这表示将/dev/sdb2上0-99编号的PE共100个移动到/dev/sdb4上。如果不加上[-99]这部分，则表示只移动0编号这一个PE。当然，在目标位置/dev/sdb4上也可以以同样的方式指定移动到的目标PE位置上。

再移动/dev/sdb2上剩余的PE到/dev/sdb3上，但此时应该先看看/dev/sdb2上仍被占用的PE编号是哪些。
```
[root@server2 ~]# pvdisplay -m /dev/sdb2
  --- Physical volume ---
  PV Name               /dev/sdb2
  VG Name               firstvg
  PV Size               3.00 GiB / not usable 16.00 MiB
  Allocatable           yes 
  PE Size               16.00 MiB
  Total PE              191
  Free PE               100
  Allocated PE          91
  PV UUID               uVgv3q-ANyy-02M1-wmGf-zmFR-Y16y-qLgNMV
   
  --- Physical Segments ---
  Physical extent 0 to 99:
     FREE
  Physical extent 100 to 100:
     Logical volume      /dev/firstvg/first_lv
     Logical extents     778 to 778
  Physical extent 101 to 190:
     Logical volume      /dev/firstvg/first_lv
     Logical extents     551 to 640
```
说明从100-190都是被占用的，要移动到/dev/sdb3上的就是这些PE。
```
[root@server2 ~]# pvmove /dev/sdb2:100-190 /dev/sdb3
```
现在/dev/sdb2已经完全空闲了，也就是可以从VG中移除，然后卸载出来。
```
[root@server2 ~]# pvdisplay  /dev/sdb2  
  --- Physical volume ---
  PV Name               /dev/sdb2
  VG Name               firstvg
  PV Size               3.00 GiB / not usable 16.00 MiB
  Allocatable           yes 
  PE Size               16.00 MiB
  Total PE              191
  Free PE               191
  Allocated PE          0
  PV UUID               uVgv3q-ANyy-02M1-wmGf-zmFR-Y16y-qLgNMV
```

(4).从vg中移除pv

```
[root@server2 ~]# vgreduce firstvg /dev/sdb2   
```

(5).移除该pv

```
[root@server2 ~]# pvremove /dev/sdb2 
```
现在/dev/sdb2就完全被移除了。

## 逻辑卷的快照功能

LVM逻辑卷管理器还具备有【快照卷】的功能，这项功能很类其他软件的还原时间点功能。例如可以对某一个LV逻辑卷设备做一次快照，如果今后发现数据被改错了，可以将之前做好的快照卷进行覆盖还原。

LVM逻辑卷管理器的快照功能有两项特点：一是快照卷的大小应该尽量等同于LV逻辑卷的容量；二是快照功能仅一次有效，一旦被还原后则会被自动立即删除。

首先应当查看下卷组的信息：
```
[root@server2 ~]# vgdisplay
  --- Volume group ---
  VG Name               firstvg
  System ID             
  Format                lvm2
  Metadata Areas        4
  Metadata Sequence No  31
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               0
  Max PV                0
  Cur PV                4
  Act PV                4
  VG Size               14.67 GiB
  PE Size               16.00 MiB
  Total PE              939
  Alloc PE / Size       939 / 14.67 GiB
  Free  PE / Size       0 / 0   
  VG UUID               GLwZTC-zUj9-mKas-CJ5m-Xf91-5Vqu-oEiJGj  
```
通过卷组的输出信息可以很清晰的看到卷组中已用120M，空闲资源有39.88G，接下来在逻辑卷设备所挂载的目录中用重定向写入一个文件：
```
echo "Welcome to Linux " > /linux/readme.txt
```

(1).**第1步：使用-s参数来生成一个快照卷，使用-L参数来指定切割的大小，另外要记得在后面写上这个快照是针对哪个逻辑卷做的**。

```
lvcreate -L 120M -s -n SNAP /dev/storage/vo
```

(2).**第2步：在LV设备卷所挂载的目录中创建一个100M的垃圾文件，这样再来看快照卷的状态就会发现使用率上升了**：

```
dd if=/dev/zero of=/linuxprobe/files count=1 bs=100M
lvdisplay
```

(3).**第3步：为了校验SNAP快照卷的效果，需要对逻辑卷进行快照合并还原操作，在这之前记得先卸载掉逻辑卷设备与目录的挂载**

```
umount /linux
lvconvert --merge /dev/storage/SNAP
```

(4).**第4步：快照卷会被自动删除掉，并且刚刚在逻辑卷设备被快照后再创建出来的100M垃圾文件也被清除了**：

```
mount xxxx
ls /linux
```