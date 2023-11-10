---
title: Linux管理文件系统(2)：分区和创建文件系统
p: linux/fsmgr_mkpart_mkfs.md
date: 2019-07-07 11:20:43
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------


# Linux管理文件系统(2)：分区和创建文件系统


分区是为了在逻辑上将某些柱面隔开形成边界。它是以柱面为单位来划分的，但是从CentOS 7开始，是按照扇区进行划分的。

在磁盘数据量非常大的情况下，划分分区的好处是扫描块位图等更快速：不用再扫描整块磁盘的块位图，只需扫描对应分区的块位图。

## 分区方法(MBR和GPT)

MBR格式的磁盘中，会维护磁盘第一个扇区——MBR扇区，在该扇区中第446字节之后的64字节是分区表，每个分区占用16字节，所以限制了一块磁盘最多只能有4个主分区(Primary,P)，如果多于4个区，只能将主分区少于4个，通过建立扩展分区(Extend,E)，然后在扩展分区建立逻辑分区(Logical,L)的方式来突破4个分区的限制，逻辑分区的数量不受限制。

在Linux中，MBR格式的磁盘主分区号从1-4，扩展分区号从2-4，逻辑分区号从5开始。

例如，一块盘想分成6个分区，可以：

```
1P+5L:sda1+sda5+sda6+sda7+sda8+sda9
2P+4L:sda1+sda2+sda5+sda6+sda7+sda8
3P+3L:sda1+sda2+sda3+sda5+sda6+sda7
```

而GPT格式突破了MBR的限制，它不再限制只能存储4个分区表条目，而是使用了类似MBR扩展分区表条目的格式，它允许有128个主分区，这也使得它可以对超过2TB的磁盘进行分区。

## MBR和GPT分区表信息

在MBR格式分区表中，前446个字节是主引导记录，即boot loader。中间64字节记录着分区表信息，每个主分区信息占用16字节，因此最多只能有4个主分区。如果使用扩展分区，则扩展分区对应的16字节记录的是指向扩展分区中扩展分区表的指针。

![](/img/linux/wps8.jpg) 

在MBR磁盘上，分区和启动信息是保存在一起的，如果这部分数据被覆盖或破坏，只能重建MBR。而GPT在整个磁盘上保存多个这部分信息的副本，因此它更为健壮，并可以恢复被破坏的这部分信息。GPT还为这些信息保存了循环冗余校验码(CRC)以保证其完整和正确，如果数据被破坏，GPT会发现这些破坏，并从磁盘上的其他地方进行恢复。

下面是GPT格式的分区表信息。

![](/img/linux/wps9.jpg) 

EFI部分可以分为4个区域：EFI信息区(GPT头)、分区表、GPT分区区域和备份区域。

- EFI信息区(GPT头)：起始于磁盘的LBA1，通常也只占用这个单一扇区。其作用是定义分区表的位置和大小。GPT头还包含头和分区表的校验和，这样就可以及时发现错误。
- 分区表：分区表区域包含分区表项。这个区域由GPT头定义，一般占用磁盘LBA2～LBA33扇区，每扇区可存储4个主分区的分区信息，所以共能分128个主分区。分区表中的每个分区项由起始地址、结束地址、类型值、名字、属性标志、GUID值组成。分区表建立后，128位的GUID对系统来说是唯一的。
- GPT分区：最大的区域，由分配给分区的扇区组成。这个区域的起始和结束地址由GPT头定义。
- 备份区：备份区域位于磁盘的尾部，包含GPT头和分区表的备份。它占用GPT结束扇区和EFI结束扇区之间的33个扇区。其中最后一个扇区用来备份1号扇区的EFI信息，其余的32个扇区用来备份LBA2～LBA33扇区的分区表。

## 在线添加磁盘

正常情况下，添加磁盘后需要重启系统才能被内核识别，在/dev/下才有对应的设备号，使用`fdisk -l`才会显示出来。但是有时候不方便重启。此时可以使用下面的方法。

```
[root@node1 ~]# ls /sys/class/scsi_host/  # 查看主机scsi总线号
host0  host1  host2
```

重新扫描scsi总线来添加新设备，之所以扫描scsi总线是因为虚拟机中添加硬盘插入的是scsi硬盘。

```
[root@node1 ~]# echo "- - -" > /sys/class/scsi_host/host0/scan
[root@node1 ~]# echo "- - -" > /sys/class/scsi_host/host1/scan
[root@node1 ~]# echo "- - -" > /sys/class/scsi_host/host2/scan
[root@node1 ~]# fdisk -l      # 再查看就有了
```

如果scsi_host目录系很多hostN目录，则使用循环来完成。

```
[root@xuexi scsi_host]# ls /sys/class/scsi_host/
host0   host11  host14  host17  host2   host22  host25  host28  host30  host4  host7
host1   host12  host15  host18  host20  host23  host26  host29  host31  host5  host8
host10  host13  host16  host19  host21  host24  host27  host3   host32  host6  host9
[root@xuexi scsi_host]# for i in /sys/class/scsi_host/host*/scan;do echo "- - -" >$i;done
```

## 使用fdisk分区工具

fdisk工具用来分MBR磁盘上的区。要分GPT磁盘上的区，可以使用gdisk。parted工具对这两种格式的磁盘分区都支持。

如果一个存储设备已经分过区，那么它可能是mbr格式的，也可能是gpt格式的，如果已经是mbr格式的，则只能继续使用fdisk进行分区，如果已经是gpt格式的，则只能使用gdisk进行分区。当然，无论什么格式的都可以使用parted进行分区，只不过也只能划分和已存在分区格式一样的分区，因为无论何种格式的分区，它的分区表和分区标识是已经固定的。

使用fdisk分区，它只能实现MBR格式的分区。

```
[root@xuexi ~]# fdisk /dev/sdb   # sdb后没加数字
Command (m for help): m          # 输入m查看可用命令帮助
Command action                  
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition        # 删除分区，如果删除扩展分区同时会删除里面的逻辑分区
   l   list known partition types # 列出分区类型
   m   print this menu           # 显示帮助信息
   n   add a new partition       # 创建新分区
   o   create a new empty DOS partition table
   p   print the partition table      # 输出分区信息
   q   quit without saving changes    # 不保存退出
   s   create a new empty Sun disklabel
   t   change a partition's system id  # 修改分区类型
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit    # 保存分区信息并退出
   x   extra functionality (experts only)
```

新建第一个主分区：

```
Command (m for help): n                   # 添加分区
Command action
   e   extended                           # 添加扩展分区
   p   primary partition (1-4)            # 添加主分区
p                                         # 输入p来创建第一个主分区
Partition number (1-4): 1                 # 输入分区号，从1开始
First cylinder (1-1305, default 1):     # 输入柱面号，不输人默认是1
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-1305, default 1305): +2G
# 给第一个主分区/dev/sdb1分2G，也可以使用柱面号来指定大小

Command (m for help): p     # 第一个分区结束，p查看下已分区信息

Disk /dev/sdb: 10.7 GB, 10737418240 bytes
255 heads, 63 sectors/track, 1305 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x2d8d64eb
 
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         262     2104483+  83  Linux
```

新建扩展分区：

```
Command (m for help): n      # 再建一个分区
Command action
   e   extended
   p   primary partition (1-4)
e      # 创建扩展分区
Partition number (1-4): 2     # 扩展分区号为2
First cylinder (263-1305, default 263): 
Using default value 263
Last cylinder, +cylinders or +size{K,M,G} (263-1305, default 1305): # 剩余空间全部给扩展分区
Using default value 1305
Command (m for help): p  
Disk /dev/sdb: 10.7 GB, 10737418240 bytes
255 heads, 63 sectors/track, 1305 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x2d8d64eb
 
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         262     2104483+  83  Linux
/dev/sdb2             263        1305     8377897+   5  Extended
```

新建逻辑分区：

```
Command (m for help): n      # 新建逻辑分区
Command action
   l   logical (5 or over)   # 这里不再是扩展分区标识e，只有l。
                             # 如果已有3个主分区，这里连l都没有
   p   primary partition (1-4)
l        # 新建逻辑分区
First cylinder (263-1305, default 263):   # 这里也不能选逻辑分区号了
Using default value 263
Last cylinder, +cylinders or +size{K,M,G} (263-1305, default 1305): +3G
Command (m for help): p
Disk /dev/sdb: 10.7 GB, 10737418240 bytes
255 heads, 63 sectors/track, 1305 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x2d8d64eb
 
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         262     2104483+  83  Linux
/dev/sdb2             263        1305     8377897+   5  Extended
/dev/sdb5             263         655     3156741   83  Linux
分区结束，保存。如果不保存，则按q。
Command (m for help): w   
The partition table has been altered!
 
Calling ioctl() to re-read partition table.
Syncing disks.
```

分区的过程，实质上是划分柱面以及修改分区表。

上面的fdisk操作全部是在内存中执行的，必须保存生效。保存后，内核还未识别该分区，可以查看`/proc/partition`目录下存在的文件，这些文件是能被内核识别的分区。运行`partprobe`或`partx`命令重新读取分区表让内核识别新的分区，内核识别后才可以格式化。而且分区结束时按w保存分区表有时候会失败，提示重启，这时候运行partprobe命令可以代替重启就生效。

```
[root@xuexi ~]# partprobe    # 执行partprobe，下面一堆信息，不理它
Warning: WARNING: the kernel failed to re-read the partition table on /dev/sda (Device or resource busy).  As a result, it may not reflect all of your changes until after reboot.
Warning: Unable to open /dev/sr0 read-write (Read-only file system).  /dev/sr0 has been opened read-only.
Warning: Unable to open /dev/sr0 read-write (Read-only file system).  /dev/sr0 has been opened read-only.
Error: Invalid partition table - recursive partition on /dev/sr0.
```

也可指定在/dev/sdb上重加载分区表，省的无法读取正忙的/dev/sda磁盘，给出上面一堆信息。

```
[root@xuexi ~]# partprobe /dev/sdb 
```

分区之后再使用`fdisk -l`查看新的分区状态。

将上面分区所需要使用的命令总结如下，以后便于使用脚本分区。

```
fdisk /dev/sdb  # 选择要分区的设备
n  # 创建分区
p/e/l # 选择分区类型
```

如果主分区数有3个，且已经划分了扩展分区，再继续分区时将只能划分逻辑分区，这种情况下`l`子命令会直接跳过进入下一个阶段。

```
N  # 指定分区号
\n  # 指定起始柱面号，使用默认值就直接回车即换行
+N  # 指定分区大小为N
w   # 分区结束保存退出
partprobe /dev/sdb &>/dev/null # 重读分区表
fdisk -l | grep "^/dev/sdb"  &>/dev/null # 检查分区状态
```

## 使用gdisk分区工具

gdisk用来划分gpt分区，需要单独安装这个工具包。

```
yum -y install gdisk
```

分区的时候直接带上设备即可。以下是对新硬盘划分gpt分区的过程。

```
[root@xuexi ~]# gdisk /dev/sdb
GPT fdisk (gdisk) version 0.8.10
 
Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present
 
Creating new GPT entries.

Command (? for help): ?
b       back up GPT data to a file
c       change a partition's name
d       delete a partition                               # 删除分区
i       show detailed information on a partition         # 列出分区详细信息
l       list known partition types                       # 列出所以已知的分区类型
n       add a new partition                              # 添加新分区
o       create a new empty GUID partition table (GPT)    # 创建一个新的空的guid分区表
p       print the partition table                        # 输出分区表信息
q       quit without saving changes                      # 退出gdisk工具
r       recovery and transformation options (experts only)  
s       sort partitions                                 
t       change a partition's type code                   # 修改分区类型
v       verify disk 
w       write table to disk and exit                     # 将分区信息写入到磁盘
x       extra functionality (experts only)              
?       print this menu
```

添加一个新分区。

```
Command (? for help): n     
Partition number (1-128, default 1): 
First sector (34-41943006, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-41943006, default = 41943006) or {+-}size{KMGTP}: +10G
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'
 
Command (? for help): p
Disk /dev/sdb: 41943040 sectors, 20.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): F8AE925F-515F-4807-92ED-4109D0827191
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 41943006
Partitions will be aligned on 2048-sector boundaries
Total free space is 20971453 sectors (10.0 GiB)
 
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20973567   10.0 GiB    8300  Linux filesystem
 
Command (? for help): i
Using 1
Partition GUID code: 0FC63DAF-8483-4772-8E79-3D69D8477DE4 (Linux filesystem)
Partition unique GUID: B2452103-4F32-4B60-AEF7-4BA42B7BF089
First sector: 2048 (at 1024.0 KiB)
Last sector: 20973567 (at 10.0 GiB)
Partition size: 20971520 sectors (10.0 GiB)
Attribute flags: 0000000000000000
Partition name: 'Linux filesystem'
```

保存分区表到磁盘。

```
Command (? for help): w  
 
Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!
 
Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sdb.
The operation has completed successfully.
```

执行partprobe重新读取分区表信息。

```
[root@server2 ~]# partprobe /dev/sdb
```

gdisk还有几个expert only的命令，其实没什么专家不专家可用的，只需要知道命令何时能用，它们的作用是什么？

在gdisk交互过程命令行下，按下x表示进入扩展功能模式，该模式下的功能大部分都和gpt分区表相关，在不是非常了解gpt分区表结构的时候不建议做修改动作，但是查看信息类是没问题的。以下是扩展功能模式下的命令。

```
Command (? for help): x
 
Expert command (? for help): ?
a       set attributes
c       change partition GUID
d       display the sector alignment value
e       relocate backup data structures to the end of the disk
g       change disk GUID
h       recompute CHS values in protective/hybrid MBR
i       show detailed information on a partition
l       set the sector alignment value
m       return to main menu
n       create a new protective MBR
o       print protective MBR data
p       print the partition table
q       quit without saving changes 
r       recovery and transformation options (experts only)
s       resize partition table    # 修改分区表大小，注意不是分区大小
t       transpose two partition table entries
u       Replicate partition table on new device  # 将分区表导出
v       verify disk
w       write table to disk and exit
z       zap (destroy) GPT data structures and exit     # 损毁gpt上的数据
?       print this menu
```

## 使用parted分区工具

parted支持mbr格式和gpt格式的磁盘分区。它的强大在于可以一步到位而不需要不断的交互式输入(也可以交互式)。

parted分区工具是实时的，所以每一步操作都是直接写入磁盘而不是写进内存，它不像fdisk/gdisk还需要w命令将内存中的结果保存到磁盘中。

```
[root@xuexi ~]# parted /dev/sdc
GNU Parted 2.1
Using /dev/sdc
Welcome to GNU Parted! Type 'help' to view a list of commands.
 
(parted) help
  align-check TYPE N                     check partition N for TYPE(min|opt) alignment
  check NUMBER (centos 7上已删除该功能) do a simple check on the file system
  cp [FROM-DEVICE] FROM-NUMBER TO-NUMBER (centos 7上已删除该功能)   copy file system to another partition
  help [COMMAND]                        print general help, or help on COMMAND
  mklabel,mktable LABEL-TYPE             create a new disklabel (partition table)
  mkfs NUMBER FS-TYPE (centos 7上已删除该功能)  make a FS-TYPE file system on partition NUMBER
  mkpart PART-TYPE [FS-TYPE] START END    make a partition
  mkpartfs PART-TYPE FS-TYPE START END  (centos 7上已删除该功能)   make a partition with a file system
  move NUMBER START END    (centos 7上已删除该功能)  move partition NUMBER
  name NUMBER NAME                       name partition NUMBER as NAME
  print [devices|free|list,all|NUMBER]   display the partition table,available devices,free space, all                                            found partitions,or a particular partition
  quit                                     exit program
  rescue START END                      rescue a lost partition near START and END
  resize NUMBER START END (修改分区大小(centos 7上已删除该功能))  resize partition NUMBER and its file system
  rm NUMBER      (删除分区)             delete partition NUMBER
  select DEVICE (重选磁盘进入parted状态)  choose the device to edit
  set NUMBER FLAG STATE  (设置分区状态，如将其off或on)   change the FLAG on partition NUMBER
  toggle [NUMBER [FLAG]] (修改文件系统类型，如swap、lvm)   toggle the state of FLAG on partition NUMBER
  unit UNIT  (修改默认单位，kB/MB/GB等)   set the default unit to UNIT
  version     display the version number and copyright information of GNU Parted
```

常用的命令是mklabel/rm/print/mkpart/help/quit，至于parted中一些看上去很好的功能如mkfs/mkpartfs/resize等可能会损毁当前数据而不够安全，所以只要使用它的5个常用命令即可。

parted分区的前提是磁盘已经有分区表(partition table)或磁盘标签(disk label)，否则将显示`unrecognised disk label`，这是和fdisk/gdisk不同的地方，所以需要先使用mklabel创建标签或分区表，最常见的标签(分区表)为`msdos`和`gpt`，其中msdos分区就是MBR格式的分区表，也就是会有主分区、扩展分区和逻辑分区的概念和限制。

下面使用parted对/dev/sdc创建msdos的新分区。

```
[root@xuexi ~]# parted /dev/sdc
GNU Parted 2.1
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel             # 创建磁盘分区标签(分区表类型)                                                
New disk label type? msdos   # 选择msdos即MBR类型     
                             # 上面的两步也可以直接一步进行：(parted) mklabel msdos       
(parted) mkpart              # 开始进行分区      
Partition type?  primary/extended? p  # 创建主分区
File system type?  [ext2]? ext4       # 创建ext4文件系统
                                      # (这里虽指明了文件系统，但没有意义，仍需手动格式化并选择文件系统类型)
Start? 1                              # 分区开始位置，默认是M为单位，表示从1M开始，也可直接指定1G这种方式
End? 1024                             # 分区结束位置，1024-1=1023M
 
(parted) p                   # print，查看分区信息
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdc: 21.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
 
Number  Start   End     Size    Type     File system  Flags
 1      1049kB  1024MB  1023MB  primary
 
# 可以一步完成一个命令中的多个动作
(parted) mkpart p ext4 1026M 4096M    # 可一步完成，也可一步完成到任何位置，然后继续交互下一步
# 可能会提示分区未对齐"Warning: The resulting partition is not properly aligned for best performance."，忽略它
 
(parted) mkpart e 4098 -1  # 创建扩展分区，注意创建扩展分区时不指定文件系统类型；-1表示剩余的全部分配给该分区 
(parted) p                                                                
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdc: 21.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
 
Number  Start   End     Size    Type      File system  Flags
 1      1049kB  1024MB  1023MB  primary
 2      1026MB  4096MB  3070MB  primary
 3      4098MB  21.5GB  17.4GB  extended               lba
 
(parted) mkpart l ext4 4099 8194     # 创建逻辑分区，指定ext4
(parted) mkpart l ext4 8195 -1       # 继续创建逻辑分区
(parted) p
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdc: 21.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
 
Number  Start   End     Size    Type      File system  Flags
 1      1049kB  1024MB  1023MB  primary
 2      1026MB  4096MB  3070MB  primary
 3      4098MB  21.5GB  17.4GB  extended               lba
 5      4099MB  8194MB  4095MB  logical
 6      8195MB  21.5GB  13.3GB  logical
 
(parted) rm 5    # 删除5号分区
(parted) p
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdc: 21.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
 
Number  Start   End     Size    Type      File system  Flags
 1      1049kB  1024MB  1023MB  primary
 2      1026MB  4096MB  3070MB  primary
 3      4098MB  21.5GB  17.4GB  extended               lba
 5      8195MB  21.5GB  13.3GB  logical
 
(parted) quit                                    # 退出parted工具
Information: You may need to update /etc/fstab.  # 提示要更新/etc/fstab中的配置，说明该工具可在线分区
```

mkfs和mkpartfs等命令不完善，下面的警告信息已经给出了提示。

```
(parted) mkfs 1                                                           
WARNING: you are attempting to use parted to operate on (mkfs) a file system.
parted's file system manipulation code is not as robust as what you'll find in
dedicated, file-system-specific packages like e2fsprogs.  We recommend
you use parted only to manipulate partition tables, whenever possible.
Support for performing most operations on most types of file systems
will be removed in an upcoming release.
Warning: The existing file system will be destroyed and all data on the partition will be lost. Do you want to
continue?
parted: invalid token: 1
Yes/No? n
```

使用parted工具进行的分区无需运行partprobe重新读取分区表，内核会即时识别已经分区的分区信息。如下所示。

```
[root@xuexi tmp]# cat /proc/partitions  | grep "sdc"
   8       32   20971520 sdc
   8       33     999424 sdc1
   8       34    2998272 sdc2
   8       35         1  sdc3
   8       37   12967936 sdc5
```

一定要注意，虽然parted工具中指定了文件系统，但是并没有意义，它仍需要手动进行格式化并指定分区类型。实际上，在parted中文件系统是可以不用指定的，即使是非交互模式下也可以省略。

## fdisk/gdisk以及parted非交互式操作分区

使用非交互分区时，最重要的是待分的区的起始点不能是已使用的。可以使用lsblk或fdisk -l或parted DEV print来判断将要从哪个地方开始分区。其实parted在非交互分区是最佳的工具，不仅是因为其书写方式简洁，而且待分区的起点如不合理它会自动提示是否要自动调整。

### parted实现非交互

parted命令只能一次非交互一个命令中的所有动作。如下所示：

```
parted /dev/sdb mklabel msdos                # 设置硬盘flag
parted /dev/sdb mkpart primary ext4 1 1000   # MBR格式分区，分别是partition type/fstype/start/end
parted /dev/sdb mkpart 1 ext4 1M 10240M  # gpt格式分区，分别是name/fstype/start/end 
parted /dev/sdb mkpart 1 10G 15G         # 省略fstype的交互式分区
parted /dev/sdb rm 1                     # 删除分区
parted /dev/sdb p                        # 输出信
```

如果不确定分区的起点大小，可以加上-s选项使用script模式，该模式下parted将回答一切默认值，如yes、no。

```
$ parted -s /dev/sdb mkpart 3 14G 16G 
Warning: You requested a partition from 14.0GB to 16.0GB.                 
The closest location we can manage is 15.0GB to 16.0GB.
Is this still acceptable to you?
Information: You may need to update /etc/fstab.
```

### fdisk实现非交互

fdisk实现非交互的原理是从标准输入中读取，每读取一行传递一次操作。

所以可以有两种方式：使用echo和管道传递；将操作写入到文件中，从文件中读取。

例如：下面的命令创建了两个分区。使用默认值时传递空行即可。

```
echo -e "n\np\n1\n\n+5G\nn\np\n2\n\n+1G\nw\n"  | fdisk /dev/sdb
```

如果要传递的操作很多，则可以将它们写入到一个文件中，从文件中读取。

```
echo -e "n\np\n1\n\n+5G\nn\np\n2\n\n+1G\nw\n" >/tmp/a.txt
fdisk /dev/sdb </tmp/a.txt
```

### gdisk实现非交互

原理同fdisk。例如： 

```
echo -e "n\n1\n\n+3G\n\nw\nY\n" | gdisk /dev/sdb
```

上面传递的各参数意义为：新建分区，分区number为1，使用默认开始扇区位置，分区大小+3G，使用默认分区类型，保存，确认。

## 格式化分区

分区结束后就需要格式化创建文件系统了，格式化分区的过程就是创建文件系统的过程。可以使用mkfs(make filesystem)工具进行格式化，也可以使用该工具家族的其他工具如mkfs.ext4/mkfs.xfs等专门针对文件系统的工具。

要查看支持的文件系统类型，只需简单的输入mkfs然后按两下tab键，就可以列出各文件系统对应的格式化命令，这些就是支持的文件系统类型。

CentOS 6上支持的：

```
[root@xuexi ~]# mkfs
mkfs        mkfs.cramfs   mkfs.ext2
mkfs.ext3   mkfs.ext4     mkfs.ext4dev
mkfs.msdos  mkfs.vfat
```

CentOS 7上支持的：

```
[root@server2 ~]# mkfs
mkfs       mkfs.btrfs   mkfs.cramfs
mkfs.ext2  mkfs.ext3    mkfs.ext4
mkfs.fat   mkfs.minix   mkfs.msdos
mkfs.vfat    mkfs.xfs
```

### mkfs工具

```
mkfs [-t fstype] 分区
```

该工具非常简单，它只需指定一个可选的`-t`选项指定要创建的文件系统类型，如果省略则默认创建ext2文件系统。该工具指定的"-t"选项其实是在调用对应文件系统专属的格式化工具。

### mke2fs工具

mkfs.ext2/mkfs.ext3/mkfs.ext4或mkfs -t extX其实都是在调用mke2fs工具。

该工具创建文件系统时，会从/etc/mke2fs.conf配置中读取默认的配置项。

```
mke2fs [ -c ] [ -b block-size ] [ -f fragment-size ] [ -g blocks-per-group ] [ -G number-of-groups ] [ -i bytes-per-inode ] [ -I inode-size ] [ -j ] [ -N number-of-inodes ] [ -m reserved-blocks-percentage ] [ -q ] [ -r fs-revision-level ] [ -v ] [ -L volume-label ] [ -S ] [ -t fs-type ] device [ blocks-count ]

选项说明：
-t fs-type
指定要创建的文件系统类型(ext2,ext3 ext4)，若不指定，则从/etc/mke2fs.conf中获取默认的文件系统类型。

-b block-size
指定每个block的大小，有效值有1024、2048和4096，单位是字节。

-I inode-size
指定inode大小，单位为字节。必须为2的幂次方，且大于等于128字节。值越大，说明inode的集合体inode table占用越多的空间，这不仅会挤占文件系统中的可用空间，还会降低性能，因为要扫描inode table需要消耗更多时间，但是在linux kernel 2.6.10之后，由于使用inode存储了很多扩展的额外属性，所以128字节已经不够用了，因此ext4默认的inode size已经变为256，尽inode大小增大了，但因为使用inode存储扩展属性带来的性能提升远高于inode size变大导致的负面影响，所以仍建议使用256字节的inode。

-i bytes-per-inode
指定每多少个字节就为其分配一个inode号。值越大，说明一个文件系统中分配的inode号越少，更适用于存储大量大文件，值越小，inode号越多，更适用于存储大量小文件。该值不能小于一个block的大小，因为这样会造成inode多余。注意，创建文件系统后该值就不能再改变了。

-c
创建文件系统前先检查设备是否有bad blocks。

-f fragment-size
指定fragments的大小，单位字节。

-g blocks-per-group
指定每个块组中的block数量。不建议修改此项。

-G number-of-groups
该选项用于ext4文件系统(严格地说是启用了flex_bg特性)，指定虚拟块组(即一个extent)中包含的块组个数，必须为2的幂次方。对于ext4文件系统来说，使用extent的功能能极大提升其性能。

-j
创建带有日志功能的文件系统，即ext3。如果要指定关于日志方面的设置，在-j的基础上再使用-J指定，不过一般默认即可，具体可指定的选项看man文档。  

-L new-volume-label
指定卷标名称，名称不得超出16字节。

-m reserved-blocks-percentage
指定文件系统保留block数量的比例，保留一部分block，可以降低物理碎片。默认比例为5%。

-N number-of-inodes
强制指定该文件系统应该分配多少个inode号，它会覆盖通过计算得出inode数量的结果(根据block大小、数量和每多少字节分配一个inode得出Inode数量)，但是不建议这么做。

-q
安静模式，可用于脚本中 

-S
重建superblock和group descriptions。在所有的superblock和备份的superblock都损坏时有用。它会重新初始化superblock和group descriptions，但不会改变inode table、bmap和imap(若真的改变，该分区数据就全丢了，还不如重新格式化)。在重建superblock后，应该执行e2fsck来保证文件系统的一致性。但要注意，应该完全正确地指定block的大小，其改选项并不能完全保证数据不丢失。
 
-v
输出详细执行过程
```

 所以，有可能用到的选项一般是`-t`指定文件系统类型，`-b`指定block大小，`-I`指定inode大小，`-i`指定分配inode的比例。

例如：

```
$ mke2fs -t ext4 -I 256 /dev/sdb2 -b 4096
mke2fs 1.41.12 (17-May-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
655360 inodes, 2621440 blocks
131072 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2684354560
80 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
       32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632
 
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
 
This filesystem will be automatically checked every 39 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```

提示使用tune2fs修改自动检测文件系统的频率。见下文。

### tune2fs修改ext文件系统属性

该工具其实没什么太大作用，文件系统创建好后很多属性是固定不能修改的，能修改的属性很有限，且都是无关紧要的。

但有些时候还是可以用到它做些事情，例如刚创建完ext文件系统后会提示修改自检时间。

```
tune2fs [-c  max-mount-counts] [-i interval-between-checks] [-j] device
-j：将ext2文件系统升级为ext3；
-c：修改文件系统最多挂载多少次后进行自检，设置为0或-1将永不自检；
-i：修改过了多少时间进行自检。时间单位可以指定为天(默认)/月/星期[d|m|w]，设置为0将永不自检。
```

例如：

```
$ tune2fs -i 0 /dev/sdb1
```

## 快速创建文件系统

有时候仅仅为了实验而插入新磁盘、扫描SCSI设备、再分区、格式化，整个过程挺麻烦的。好在，有更为便捷的方式：

```
dd if=/dev/zero of=sdx bs=1M count=32
mke2fs sdx
```

现在sdx文件就是一个ext家族的文件系统了，相当于已经格式化的/dev/sdxN，它可以直接拿来挂载使用。

```
mount sdx /mnt/sdx
```

还可以使用mkisofs命令工具快速将目录创建成一个可挂载的镜像文件：

```
mkdir -p foo/bar/baz
mkisofs -o test.iso foo  # 将foo打包成iso镜像
```

现在，test.iso镜像文件也可以直接挂载使用：

```
mount test.iso /mnt/test
```

## 查看文件系统状态信息

### lsblk

lsblk(list block devices)用于列出设备及其状态，主要列出非空的存储设备。其实它只会列出/sys/dev/block中的主次设备号文件，且默认只列出非空设备。

```
[root@server2 ~]# lsblk 
```

![](/img/linux/wps10.jpg) 

其中上面的几列意义如下：

```
NAME：设备名称；
MAJ:MIN：主设备号和此设备号；
RM：是否为可卸载设备，1表示可卸载设备。可卸载设备如光盘、USB等。并非能够umount的就是可卸载的；
SIZE：设备总空间大小；
RO：是否为只读；
TYPE：是磁盘disk，还是分区part，亦或是rom，还有loop设备；
mountpoint：挂载点。
```

查看某分区或某块设备的信息：

```
[root@server2 ~]# lsblk /dev/sdb
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb        8:16   0   20G  0 disk 
├─sdb1   8:17   0  9.5G  0 part /mydata/data
└─sdb2   8:18   0    3G  0 part 

[root@server2 ~]# lsblk /dev/sdb1
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb1   8:17   0  9.5G  0 part /mydata/data
```

另外常用的一个选项是`-f`，它可以查看到文件系统类型，和文件系统的uuid和挂载点。

```
[root@xuexi ~]# lsblk -f
NAME   FSTYPE LABEL UUID                                 MOUNTPOINT
sda                                                      
├─sda1 ext4         77b5f0da-b0f9-4054-9902-c6cdacf29f5e /boot
├─sda2 ext4         f199fcb4-fb06-4bf5-a1b7-a15af0f7cb47 /
└─sda3 swap         6ae3975c-1a2a-46e3-87f3-d5bd3f1eff48 [SWAP]
sr0                                                      
sdb                                                      
├─sdb1 ext4         95e5b9d5-be78-43ed-a06a-97fd1de9a3fe 
├─sdb2 ext2         45da2d94-190a-4548-85bb-b3c46ae6d9a7 
└─sdb3                                
```

每个已经格式化的文件系统都有其类型和uuid，而没有格式化的设备(如/dev/sdb3)，将只显示一个Name结果，表示该设备还未进行格式化。

### blkid

虽然它有不少比较强大的功能，但一般只用它一个功能，就是查看器文件系统类型和uuid。

```
[root@xuexi ~]# blkid
/dev/sda1: UUID="77b5f0da-b0f9-4054-9902-c6cdacf29f5e" TYPE="ext4" 
/dev/sda2: UUID="f199fcb4-fb06-4bf5-a1b7-a15af0f7cb47" TYPE="ext4" 
/dev/sda3: UUID="6ae3975c-1a2a-46e3-87f3-d5bd3f1eff48" TYPE="swap" 
/dev/sdb1: UUID="95e5b9d5-be78-43ed-a06a-97fd1de9a3fe" TYPE="ext4" 
/dev/sdb2: UUID="45da2d94-190a-4548-85bb-b3c46ae6d9a7" TYPE="ext2"

[root@xuexi ~]# blkid /dev/sdb1
/dev/sdb1: UUID="95e5b9d5-be78-43ed-a06a-97fd1de9a3fe" TYPE="ext4"
```

### parted print和fdisk -l

parted的print操作和fdisk的-l选项，可以查看分区的基本信息：

```
$ parted /dev/sdb p
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 21.5GB
Sector size (logical/physical): 512B/512B
**Partition Table: gpt
Disk Flags: 
 
Number  Start   End     Size    File system  Name              Flags
 1      1049kB  10.2GB  10.2GB  ext4
 2      10.2GB  13.5GB  3221MB  ext2         Linux filesystem
 
$ fdisk -l /dev/sda
Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000cb657
 
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      514047      256000   83  Linux
/dev/sda2          514048    37847039    18666496   83  Linux
/dev/sda3        37847040    41943039     2048000   82  Linux swap / Solaris 
```

### file -s

```
[root@xuexi ~]# file -s /dev/sdb2
/dev/sdb2: Linux rev 1.0 ext2 filesystem data (large files)
```

### du

du命令用于评估文件的空间占用情况，它会统计每个文件的大小，统计时会递归统计目录中的文件，也就是说，它会遍历整个待统计目录。所以如果要统计的目录中包含大量文件时，统计速度可能并不理想。

```
du [OPTION]... [FILE]...
-a, --all：列出目录中所有文件的统计信息，默认只会列出目录中子目录的统计信息，而不列出文件的统计信息
-h, --human-readable：人性化显示大小
-0, --null：以空字符结尾，即"\0"而非换行的"\n"
-S, --separate-dirs：不包含子目录的大小
-s, --summarize：对目录做总的统计，不列出目录内文件的大小信息
-c,--total：对给出的文件或目录做总计。在统计非同一个目录文件大小时非常有用。见下文例子。
-d,--max-depth=N：只列出给定层次的目录统计，如果N=0，则等价于"-s"
-t,--threshold=SIZE：排除小于SIZE的文件不显示
-x,--one-file-system：忽略不同文件系统上的文件，不对它们进行统计
-X,--exclude-from=FILE：从文件中读取要排除的文件
--exclude=PATTERN：指定要忽略不统计的文件
```

注意:  

- 上面的选项中，有些是不统计某些项，有些是不显示某些项，不显示是在统计之后过滤，它们是不一样的。

- 如果要统计的目录下挂载了一个文件系统，那么这个文件系统的大小也会被计入该目录的大小中。

下面是一些例子：

```
# 统计/etc目录的总大小
[root@xuexi ~]# du -sh /etc 
29M     /etc

# 统计/tmp目录下所有子目录的大小，会递归
[root@xuexi ~]# du -ah /tmp
4.0K    /tmp/b.txt
4.0K    /tmp/a
4.0K    /tmp/.ICE-unix
4.0K    /tmp/testdir/subdir
0       /tmp/testdir/a.log
8.0K    /tmp/testdir
24K     /tmp

# 统计/usr目录下所有子目录和子文件大小，不会递归
[root@xuexi ~]# du -h --max-depth=1 /usr
15M     /usr/include
383M    /usr/lib64
132K    /usr/local
391M    /usr/share
4.0K    /usr/etc
118M    /usr/lib
44M     /usr/libexec
49M     /usr/src
32M     /usr/sbin
4.0K    /usr/games
75M     /usr/bin
1.1G    /usr

# 统计/usr目录下除/usr/lib64外的所有子目录和子文件大小，不会递归
[root@xuexi ~]# du -h --max-depth=1 --exclude=/usr/lib64 /usr
15M     /usr/include
132K    /usr/local
391M    /usr/share
4.0K    /usr/etc
118M    /usr/lib
44M     /usr/libexec
49M     /usr/src
32M     /usr/sbin
4.0K    /usr/games
75M     /usr/bin
721M    /usr

# 统计家目录下除隐藏文件外的各文件、目录大小
[root@xuexi ~]# du -h --max-depth=1 --exclude=".*" ~

# 统计家目录下大于1M的文件
[root@xuexi ~]# du -h --max-depth=1 -t 1M ~


```

搜索符合条件的文件，然后统计它们的总大小。结合find使用，效果极佳。

```
[root@xuexi ~]# find /boot/ -type f -name "*.img" -print0 | xargs -0 du -ch
28K     /boot/grub2/i386-pc/core.img
4.0K    /boot/grub2/i386-pc/boot.img
592K    /boot/initrd-plymouth.img
44M     /boot/initramfs-0-rescue-d13bce5e247540a5b5886f2bf8aabb35.img
17M     /boot/initramfs-3.10.0-327.el7.x86_64.img
16M     /boot/initramfs-3.10.0-327.el7.x86_64kdump.img
76M     total
```

请注意`-c`和`-s`统计的区别。

```
[root@xuexi ~]# find /boot/ -type f -name "*.img" -print0 | xargs -0 du -sh
28K     /boot/grub2/i386-pc/core.img
4.0K    /boot/grub2/i386-pc/boot.img
592K    /boot/initrd-plymouth.img
44M     /boot/initramfs-0-rescue-d13bce5e247540a5b5886f2bf8aabb35.img
17M     /boot/initramfs-3.10.0-327.el7.x86_64.img
16M     /boot/initramfs-3.10.0-327.el7.x86_64kdump.img 
```

### df

df用于报告磁盘空间使用率，默认显示的大小是1K大小block数量，也就是以k为单位。

和du不同的是，df是读取每个文件系统的superblock信息，所以评估速度非常快。由于是读取superblock，所以如果目录下挂载了另一个文件系统，是不会将此挂载的文件系统计入目录大小的。

注意，du和df统计的结果是不一样的，如果对它们的结果不同有兴趣，可参考我的另一篇文章：[详细分析du和df的统计结果为什么不一样](https://www.junmajinlong.com/linux/du_df)。

如果用df统计某个文件的空间使用情况，将会转而统计该文件所在文件系统的空间使用情况。

```
df [OPTION]... [FILE]...
选项说明：
-h：人性化转换大小的显示单位
-i：统计inode使用情况而非空间使用情况
-l, --local：只列出本地文件系统的使用情况，不列出网络文件系统信息
-T, --print-type：同时输出文件系统类型
-t, --type=TYPE：只列出给定文件系统的统计信息
-x, --exclude-type=TYPE：指定不显示的文件系统类型的统计信
```

例如：

```
[root@server2 ~]# df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda2      xfs        18G  2.3G   16G  13% /
devtmpfs       devtmpfs  904M     0  904M   0% /dev
tmpfs          tmpfs     913M     0  913M   0% /dev/shm
tmpfs          tmpfs     913M  8.6M  904M   1% /run
tmpfs          tmpfs     913M     0  913M   0% /sys/fs/cgroup
/dev/sda1      xfs       247M  110M  137M  45% /boot
tmpfs          tmpfs     183M     0  183M   0% /run/user/0
/dev/sdb1      ext4      9.3G   37M  8.8G   1% /mydata/data

[root@server2 ~]# df -i
Filesystem       Inodes  IUsed    IFree IUse% Mounted on
/dev/sda2      18666496 106474 18560022    1% /
devtmpfs         231218    388   230830    1% /dev
tmpfs            233586      1   233585    1% /dev/shm
tmpfs            233586    479   233107    1% /run
tmpfs            233586     13   233573    1% /sys/fs/cgroup
/dev/sda1        256000    330   255670    1% /boot
tmpfs            233586      1   233585    1% /run/user/0
/dev/sdb1        625856     14   625842    1% /mydata/data
```

### dumpe2fs

用于查看ext类文件系统的superblock及块组信息。使用-h选项将只显示superblock信息。

以下是ext4文件系统superblock的信息。

```
[root@xuexi ~]# dumpe2fs -h /dev/sda2
dumpe2fs 1.41.12 (17-May-2010)
Filesystem volume name:   <none>
Last mounted on:          /
Filesystem UUID:          f199fcb4-fb06-4bf5-a1b7-a15af0f7cb47
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              1166880
Block count:              4666624
Reserved block count:     233331
Free blocks:              4196335
Free inodes:              1111754
First block:              0
Block size:               4096
Fragment size:            4096
Reserved GDT blocks:      1022
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8160
Inode blocks per group:   510
Flex block group size:    16
Filesystem created:       Sat Feb 25 11:48:47 2017
Last mount time:          Tue Jun  6 18:13:10 2017
Last write time:          Sat Feb 25 11:53:49 2017
Mount count:              6
Maximum mount count:      -1
Last checked:             Sat Feb 25 11:48:47 2017
Check interval:           0 (<none>)
Lifetime writes:          2657 MB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:               256
Required extra isize:     28
Desired extra isize:      28
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      d4e6493a-09ef-41a1-9d66-4020922f1aa9
Journal backup:           inode blocks
Journal features:         journal_incompat_revoke
Journal size:             128M
Journal length:           32768
Journal sequence:         0x00001bd9
Journal start:            23358
```

查看一个块组信息。

```
[root@xuexi ~]# dumpe2fs /dev/sda2 | tail -7
dumpe2fs 1.41.12 (17-May-2010)
Group 142: (Blocks 4653056-4666623) [INODE_UNINIT, ITABLE_ZEROED]
  Checksum 0x64ce, unused inodes 8160
  Block bitmap at 4194318 (+4294508558), Inode bitmap at 4194334 (+4294508574)
  Inode table at 4201476-4201985 (+4294515716)
  13568 free blocks, 8160 free inodes, 0 directories, 8160 unused inodes
  Free blocks: 4653056-4666623
  Free inodes: 1158721-1166880
```

