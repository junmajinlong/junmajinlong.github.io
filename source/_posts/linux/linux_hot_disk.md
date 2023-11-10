---
title: Linux上磁盘热插拔
p: linux/linux_hot_disk.md
date: 2020-07-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux上磁盘热插拔

首先获取scsi设备的信息。

```
[root@server2 ~]# lsscsi
[2:0:0:0]    disk    VMware,  VMware Virtual S 1.0   /dev/sda
[4:0:0:0]    cd/dvd  NECVMWar VMware SATA CD01 1.00  /dev/sr0
```

有些操作系统没有lsscsi命令，则可以使用下面的方法获取scsi设备信息。

```
[root@server2 ~]# ll /sys/bus/scsi/drivers/sd/

total 0
lrwxrwxrwx 1 root root    0 Jun 22 17:29 2:0:0:0 -> ../../../../devices/pci0000:00/0000:00:10.0/host2/target2:0:0/2:0:0:0
--w------- 1 root root 4096 Jun 22 17:29 bind
--w------- 1 root root 4096 Jun 22  2017 uevent
--w------- 1 root root 4096 Jun 22 17:29 unbind
[root@server2 ~]# ll /sys/bus/scsi/drivers/sd/2\:0\:0\:0/block/

total 0
drwxr-xr-x 10 root root 0 Jun 22  2017 sda
```

然后查看/proc/scsi/scsi文件，获取对应scsi设备的详细信息。

```
[root@server2 ~]# cat /proc/scsi/scsi

Attached devices:
Host: scsi2 Channel: 00 Id: 00 Lun: 00
  Vendor: VMware,  Model: VMware Virtual S Rev: 1.0
  Type:   Direct-Access                    ANSI  SCSI revision: 02
Host: scsi4 Channel: 00 Id: 00 Lun: 00
  Vendor: NECVMWar Model: VMware SATA CD01 Rev: 1.00
  Type:   CD-ROM                           ANSI  SCSI revision: 05
Host: scsi2 Channel: 00 Id: 01 Lun: 00
  Vendor: VMware,  Model: VMware Virtual S Rev: 1.0
  Type:   Direct-Access                    ANSI  SCSI revision: 02
```

在此处，有两块直连(Direct-Access)的scsi磁盘，一块通过光驱cd-rom连接的光盘。我们只考虑scsi磁盘，所以这两块磁盘在scsi中的定位符为`2:0:0:0`和`2:0:1:0`。**如果继续插入一块盘，那么新盘在scsi中的定位符为`2:0:2:0`**，这个数值串非常重要。

## 热插

在向计算机中插入一块磁盘后，内核因为识别不了它所以不会产生任何事件通知，因此在/sys目录中不会产生任何文件，任何工具也就读取不了它。重启系统肯定是可以解决的，但是Linux支持热插。

热插新盘的方式是向/proc/scsi/scsi中写入新scsi设备的信息。方式如下：

```
echo "scsi add-single-device a b c d" >/proc/scsi/scsi
```

其中：

```
a == hostadapter id (first one being 0)
b == SCSI channel on hostadapter (first one being 0)
c == ID
d == LUN (first one being 0)
```

例如上面的例子，应该添加如下信息：

```
[root@server2 ~]# echo "scsi add-single-device 2:0:2:0" >/proc/scsi/scsi
```

当然，重新扫描scsi总线也可以实现热插的功能。因为上面的例子中，scsi host id为2(即host2)，所以扫描的是host2，这样host2这个scsi上的所有设备都会被重新扫描。

```
[root@server2 ~]# echo "- - -" > /sys/class/scsi_host/host2/scan
```

如果不知道要扫描哪个host，直接使用循环全部扫描。

```
[root@xuexi ~]# for i in /sys/class/scsi_host/host*/scan;do echo "- - -" >$i;done
```

热插之后，`fdisk -l`等命令就可以识别到该磁盘了。

## 热拔

热拔磁盘的方式是在/proc/scsi/scsi中移除对应scsi设备的信息。方式如下：

```
echo "scsi remove-single-device a b c d" >/proc/scsi/scsi
```

例如删除`2:0:2:0`这块磁盘。

```
[root@server2 ~]# echo "scsi remove-single-device 2 0 2 0" >/proc/scsi/scsi
```

因为要删除的设备已经存在，/sys中已经有它完整的信息，所以也从其自身设备上进行删除。

首先查看scsi设备信息。

```
[root@server2 ~]# lsscsi
[2:0:0:0]    disk    VMware,  VMware Virtual S 1.0   /dev/sda
[2:0:1:0]    disk    VMware,  VMware Virtual S 1.0   /dev/sdb
[4:0:0:0]    cd/dvd  NECVMWar VMware SATA CD01 1.00  /dev/sr0
```

 例如要删除/dev/sdb，即`2:0:1:0`。先看看它的文件信息。

```
[root@server2 ~]# ls /sys/bus/scsi/drivers/sd/2\:0\:1\:0/

queue_ramp_up_period          evt_lun_change_reported
evt_soft_threshold_reached    evt_inquiry_change_reported
evt_capacity_change_reported  evt_mode_parameter_change_reported
block/         model          scsi_level      bsg/
power/         state          delete          vpd_pg83
queue_depth    subsystem/     device_blocked  timeout
device_busy    generic/       queue_type      type
dh_state       iocounterbits  rescan          uevent
driver/        iodone_cnt     rev             unpriv_sgio
eh_timeout     ioerr_cnt      scsi_device/    vendor
iorequest_cnt  scsi_disk/     vpd_pg80        modalias
scsi_generic/  evt_media_change
```

在其中有3个文件：delete、rescan和state。其中state记录了该设备是否正在运行中。而delete和rescan文件则用于删除和重新扫描该设备。

例如，删除该设备，即热拔。

```
[root@server2 ~]# echo 1 > /sys/bus/scsi/drivers/sd/2\:0\:1\:0/delete
```