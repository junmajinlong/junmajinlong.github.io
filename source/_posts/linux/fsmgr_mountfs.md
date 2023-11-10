---
title: Linux管理文件系统(3)：mount挂载各种文件系统
p: linux/fsmgr_mountfs.md
date: 2019-07-07 11:20:44
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux管理文件系统(3)：mount挂载各种文件系统


在此，只简单介绍mount和umount的用法，至于实现挂载和卸载的机制和原理细节，参看[挂载文件系统的细节](https://www.junmajinlong.com/linux/ext_filesystem#fs_mount_details)。

## mount基本用法

mount用来显示挂载信息或者进行文件系统挂载，它的功能及其的强大(强大到离谱)，它不仅支持挂载非常多种文件系统，如ext/xfs/nfs/smbfs/cifs (win上的共享目录)等，还支持共享挂载点、继承挂载点(父子关系)、绑定挂载点、移动挂载点等等功能。在本文只介绍其最简单的挂载功能。

不同的文件系统挂载选项是有所差别的，在挂载过程中如果出错，应该man mount并查看对应文件系统的挂载选项。

mount并非只能挂载文件系统，也可以将目录挂载到另一个目录下，其实它实现的是目录【硬链接】，默认情况下，是无法对目录建立硬链接的，但是通过mount可以完成绑定，绑定后两个目录的inode号是完全相同的，但尽管建立的是目录的【硬链接】，但其实也仅是拿来当软链接用。

以下是ext类文件系统的一部分选项，可能有些选项是不支持其他文件系统的。

```
mount # 将显示当前已挂载信息

mount [-t 欲挂载文件系统类型 ] [-o 特殊选项] 设备名 挂载目录
-a  将/etc/fstab文件里指定的挂载选项重新挂载一遍。
-t  支持ext2/ext3/ext4/vfat/fat/iso9660(光盘默认格式)。不用-t时默认会调用blkid来获取文件系统类型。
-n  不把挂载记录写在/etc/mtab文件中，一般挂载会在/proc/mounts中记录下挂载信息，然后同步到/etc/mtab，指定-n表示不同步该挂载信息。
-o  指定挂载特殊选项。下面是两个比较常用的：
  loop  挂载镜像文件，如iso文件
  ro    只读挂载
  rw    读写挂载
  auto  相当于mount -a
  dev   如果挂载的文件系统中有设备访问入口则启用它，使其可以作为设备访问入口
  default rw,suid,dev,exec,auto,nouser,async,and relatime
  async 异步挂载，只写到内存
  sync  同步挂载，通过挂载位置写入对方硬盘
  atime 修改访问时间，每次访问都修改atime会导致性能降低，所以默认是noatime
  noatime 不修改访问时间，高并发时使用这个选项可以减少磁盘IO
  nodiratime   不修改文件夹访问时间，高并发时使用这个选项可以减少磁盘IO
  exec/noexec  挂载后的文件系统里的可执行程序是否可执行，默认是可以执行exec，优先级高于权限的限定
  remount  重新挂载，此时可以不用指定挂载点。
  suid/nosuid 对挂载的文件系统启用或禁用suid，对于外来设备最好禁用suid
  _netdev 需要网络挂载时默认将停留在挂载界面直到加载网络了。使用_netdev可以忽略网络正常挂载。如NFS开机挂载。
  user  允许普通用户进行挂载该目录，但只允许挂载者进行卸载该目录
  users  允许所有用户挂载和卸载该目录
  nouser  禁止普通用户挂载和卸载该目录，这是默认的，默认情况下一个目录不指定user/users时，将只有root能挂载
```

一般user/users/nouser都用在/etc/fstab中，直接在命令行下使用这几个选项意义不是很大。

例如：

(1).挂载CentOS的安装镜像到/mnt。

```
mount /dev/cdrom /mnt
```

其实/dev/cdrom是/dev/sr0的一个软链接，/dev/sr0是光驱设备，所以也可以用/dev/sr0进行挂载。

```
mount /dev/sr0 /mnt
```

(2).重新挂载。

```
[root@xuexi ~]# mount -t ext4 -o remount /dev/sdb1 /data1
```

(3).重新挂载文件系统为可读写。

```
mount -t ext4 -o rw remount /dev/sdb1 /data1
```

(4).挂载windows的共享目录。

win上共享文件的文件系统是cifs类型，要在Linux上挂载，必须得有mount.cifs命令，如果没有则安装cifs-utils包。

假设win上共享目录的unc路径为`\\192.168.100.8\test`，共享给的用户名和密码分别为long3:123，要挂在linux上的/mydata目录上。

```
$ mount.cifs -o username="long3",password="123" //192.168.100.8/test /mydata
```

注意，如果是比较新版本的win10(2017年之后更新的版本)或较新版本的win server，直接mount.cifs会报错：

```
[root@xuexi ~]# mount.cifs -o username="long3",password="123" //192.168.100.8/test /mnt         
mount error(112): Host is down
Refer to the mount.cifs(8) manual page (e.g. man mount.cifs)
```

这是因为2017年微软的一个补丁禁用了SMBv1协议，通过smbclient的报告可知：

```
$ yum -y install samba-client
$ smbclient -L //192.168.100.8
Enter root's password: 
protocol negotiation failed: NT_STATUS_CONNECTION_RESET
```

因此，在mount的时候指定cifs(SMB)的版本号为2.0即可。

```
$ mount.cifs -o username="long3",password="123",vers=2.0 //192.168.100.8/test /mnt
```

但是需要注意，在CentOS 4,5,6下的模块cifs.ko版本(较低)只能使用SMBv1协议，因此即使指定版本号也一样无效。只有在CentOS 7上才能使用SMBv2或SMBv3。

(5).基于ssh挂载远程目录。

如何基于ssh像NFS一样挂载远程主机上的目录？可以通过sshfs工具，该工具在fuse-sshfs包中，这个包在epel源中提供。

```
yum -y install fuse-sshfs
```

例如，挂载192.168.100.8上的根目录到本地的/mnt上。

```
sshfs 192.168.100.8:/ /mnt
```

卸载时直接umount即可。

关于sshfs，详细内容见：<https://www.junmajinlong.com/linux/sshfs>

(6).挂载目录到另一个目录下。挂载目录时，挂载目录和挂载点的inode是相同的，它们两者的内容也是完全相同的。

```
mount --bind /mydata /mnt
```

(7).查看某个目录是否是挂载点，使用mountpoint命令。

```
[root@xuexi ~]# mountpoint /mydata/
/mydata/ is a mountpoint

[root@xuexi ~]# echo $?            
0

[root@xuexi ~]# mountpoint /mnt
/mnt is not a mountpoint

[root@xuexi ~]# echo $?        
1
```

挂载的参数信息存放在/proc/mounts(是/proc/self/mounts的软链接)中，在/proc/self/mountstats和/proc/mountinfo里则记录了更详细的挂载信息。

```
[root@xuexi ~]# cat /proc/mounts 
rootfs / rootfs rw 0 0
proc /proc proc rw,relatime 0 0
sysfs /sys sysfs rw,relatime 0 0
devtmpfs /dev devtmpfs rw,relatime,size=491000k,nr_inodes=122750,mode=755 0 0
devpts /dev/pts devpts rw,relatime,gid=5,mode=620,ptmxmode=000 0 0
tmpfs /dev/shm tmpfs rw,relatime 0 0
/dev/sda2 / ext4 rw,relatime,barrier=1,data=ordered 0 0
/proc/bus/usb /proc/bus/usb usbfs rw,relatime 0 0
/dev/sda1 /boot ext4 rw,relatime,barrier=1,data=ordered 0 0
none /proc/sys/fs/binfmt_misc binfmt_misc rw,relatime 0 0
```

文件系统是需要驱动支持的，没有驱动的文件系统也无法挂载，Linux中支持的文件系统驱动在/lib/modules/$(uname -r)/kernel/fs下。

```
[root@xuexi ~]# ls /lib/modules/$(uname -r)/kernel/fs/
autofs4  cachefiles  configfs  dlm    exportfs
ext3     fat  fuse   jbd       jffs2  mbcache.ko
nfs_common    nls    ubifs     xfs    btrfs cifs cramfs
ecryptfs      ext2   ext4      fscache  gfs2  jbd2
lockd    nfs  nfsd   squashfs  udf
```

## mount直接挂载iso镜像文件

有时候需要挂载CentOS的镜像文件，在虚拟机中经常是将镜像放入虚拟机的CD/DVD虚拟光驱中，然后在Linux上对/dev/cdrom进行挂载。其实/dev/cdrom是/dev/sr0的一个软链接，/dev/sr0是Linux中的光驱，所以上面的过程相当于是将镜像文件通过虚拟软件的虚拟光驱和linux的光驱连接起来，这样只需要挂载Linux中的光驱就可以了。但是，在非虚拟环境中没有虚拟光驱，而且在Linux中的一个镜像文件难道一定要拷贝到主机上通过虚拟光驱进行连接吗？

mount是一个极其强大的挂载工具，它支持挂载很多种文件类型，其中就支持挂载镜像文件，其实它连挂载目录都支持。

```
[root@xuexi ~]# mount -o loop CentOS-6.6-x86_64-bin-DVD2.iso /mnt
[root@xuexi ~]# lsblk
NAME     MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0      7:0    0   1.2G  0 loop /mnt
sda        8:0    0    20G  0 disk 
├─sda1   8:1    0   250M  0 part /boot
├─sda2   8:2    0  17.8G  0 part /
└─sda3   8:3    0     2G  0 part [SWAP]
sr0       11:0    1  1024M  0 rom
```

## umount

```
umount 设备名或挂载目录
umount -lf 强制卸载
```

卸载时，既可以使用设备名也可以使用挂载点卸载。有时候挂载网络系统(如NFS)时，设备名很长，这时候可以使用挂载点来卸载就方便多了。

如果用户正在访问某个目录或文件，使得卸载一直显示Busy，使用fuser -v DIR可以知道谁正在访问该目录或文件。

```
[root@xuexi ~]# fuser -v /root
                     USER        PID ACCESS COMMAND
/root:               root      37453 ..c.. bash
```

使用-k选项kill掉正在使用目录或文件的进程，使用-km选项kill掉文件系统上的所有进程，然后再umount。

```
[root@xuexi ~]# fuser -km /mnt/cdrom;umount /mnt/cdrom
```

## 开机自动挂载/etc/fstab

通过将挂载选项写入到/etc/fstab中，系统会自动挂载该文件中的配置项。但要注意，该文件在开机的前几个过程中就被读取，所以配置错误很可能会导致开机失败。

![](/img/linux/wps11.jpg) 

其中最后两列，它们分别表示备份文件系统和开机自检，一般都可以设置为0。

由于能用的备份工具众多，没人会在这里设置备份，所以备份列设置为0。

最后一列是开机自检设置列，开机自检调用的是fsck程序，所有有些ext类文件系统作为`/`时，可能会设置为1，但是fsck是不支持xfs文件系统的，所以对于xfs文件系统而言，该项必须设置为0。

其实无需考虑那么多，直接将这两列设置为0就可以了。

### 修复错误的/etc/fstab

万一/etc/fstab配置错误，导致开机无法加载。这时提示输入root密码进入单用户模式，只不过当用户模式下根文件系统是只读的，哪怕是root也无法直接修改/etc/fstab，所以应该重新挂载`/`文件系统。

执行下面的命令，重挂载根分区，并给读写权限，再去修改错误的fstab文件记录，再重启。

```
[root@xuexi ~]# mount -n -o remount,rw /
```

## 按需自动挂载(autofs)

使用autofs实现需要挂载时就挂载，不需要挂载时5分钟后自动卸载。在实际生产环境中基本不会使用按需挂载，在Linux桌面系统中用的较多。

autofs是一个服务程序，需要让其运行在后台，可以用来挂NFS，也可挂本地的文件系统。

默认不装autofs，需要自己装。

```
[root@xuexi ~]# yum install -y autofs
```

autofs实现按需挂载的方式是指定监控目录，可在其配置文件/etc/auto.master中指定。

/etc/auto.master里面只有两列：第一列是监控目录；第二列是记录挂载选项的文件，该文件可以随便取名。

```
[root@xuexi ~]# cat /etc/auto.master
/share  /etc/auto.mount    # 监控/share目录
```

上述监控的/share目录，其实这是监控的父目录，在此目录下的目录如/share/data目录可以作为挂载点，当访问到/share/data时就被监控到，然后会按照挂载选项将挂载设备挂载到/share/data上。

上述配置中配置的挂载选项文件是/etc/auto.mount，所以建立此文件，写入挂载选项。

```
[root@xuexi ~]# cat /etc/auto.mount
#cd              -fstype=iso9660,ro,nosuid,nodev :/dev/cdrom
 
# the following entries are samples to pique your imagination
#linux          -ro,soft,intr           ftp.example.org:/pub/linux
#boot           -fstype=ext2            :/dev/hda1
#floppy         -fstype=auto            :/dev/fd0
#floppy         -fstype=ext2            :/dev/fd0
#e2floppy       -fstype=ext2            :/dev/fd0
#jaz            -fstype=ext2            :/dev/sdc1
#removable      -fstype=ext2            :/dev/hdd
```

该文件有3列：

- 第一列：指定的是在/etc/auto.master指定的/share下的目录/share/data，它是真正的被监控路径，也是挂载点。可使用相对路径data表示/share/data。
- 第二列：是mount的选项，前面使用一个`-`表示，该列可有可无。
- 第三列：是待挂载设备，可以是NFS服务端的共享目录，也可以本地设备。

```
[root@xuexi ~]# vim /etc/auto.mount
data  -rw,bg,soft,rsize=32768,wsize=32768  192.168.100.61:/data
```

上面的配置表示当访问到/share/data时，自动使用参数(rw,bg,soft,rsize=32768,wsize=32768)挂载远端`192.168.100.61`的/data目录到/share/data上。

剩下的步骤就是启动autofs服务。

```
[root@xuexi data]# /etc/init.d/autofs restart
```
