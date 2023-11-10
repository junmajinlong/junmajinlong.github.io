---
title: sshfs基于ssh挂载远程目录
p: linux/sshfs.md
date: 2019-07-06 12:20:43
tags: Linux
categories: Linux
---

------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

------

# sshfs基于ssh挂载远程目录

为了像本地一样访问远程主机上的目录，通常我们会在远程主机上使用nfs来导出目录，并在本地主机上mount这个nfs文件系统。如果是windows系统，则使用cifs或samba的方式来访问。

但可能我们忽略了一个远程连接最通用的工具：ssh。其实很多和远程有关的行为，基于ssh都能完成，即使是实现像NFS一样的功能。

如何通过ssh来挂载远程目录？需要安装`fuse-sshfs`包，这个包在epel中提供。使用fuse-sshfs包提供的sshfs工具可以基于ssh直接挂载远程目录，不用像NFS一样还要export。

```
$ yum -y install fuse-sshfs

$ rpm -ql fuse-sshfs
/usr/bin/sshfs
/usr/share/doc/fuse-sshfs-2.5
/usr/share/doc/fuse-sshfs-2.5/AUTHORS
/usr/share/doc/fuse-sshfs-2.5/COPYING
/usr/share/doc/fuse-sshfs-2.5/ChangeLog
/usr/share/doc/fuse-sshfs-2.5/FAQ.txt
/usr/share/doc/fuse-sshfs-2.5/NEWS
/usr/share/doc/fuse-sshfs-2.5/README
/usr/share/man/man1/sshfs.1.gz
```

例如，挂载192.168.100.150上的根目录"/"到本地的/mnt上。**注意：只能挂载远程目录，像普通文件、块设备(如/dev/sda2)等无法挂载。**

```
sshfs root@192.168.100.150:/ /mnt
```

如此一来，以后可以直接访问本地/mnt来访问远程的根目录。例如复制文件、移动文件、新建文件等等操作。

当然，更多时候是非root用户，不应直接挂载在/mnt上，否则会有权限问题。通常来说，应该创建一个子目录，然后以指定用户名挂载：

```bash
sudo mkdir /mnt/bian

# 将Windows上的`E:/bian`目录挂载到/mnt/bian上，
# 且指定挂载目录中文件的用户名和umask值，
# 要求Windows上开启sshd服务
sshfs -o umask=0022,gid=1000,uid=1000 \
malong@192.168.200.1:'/E:/onedrive/code/bian' /mnt/bian
```

如果要卸载挂载点。直接umount即可。

```
umount /mnt/bian
```

相比于NFS，sshfs更简洁，它是基于fuse模块来实现的，可以认为sshfs所挂载的文件系统是fuse文件系统的一种实现。所谓fuse文件系统，它全称为`filesystem in userspace`，显然，它是用户空间的文件系统(其实是一个虚拟文件系统)，其功能非常强大，可用于实现自己的文件系统。详细信息可以`sshfs -h`，`man sshfs`，`man fusermount`，`man mount.fuse`。

但是NFS比sshfs要完整的多，nfs毕竟是【小型】分布式文件系统，对数据的一致性、完整性实现的都比较完美，访问权限控制也比sshfs要丰富的多。

总的来说，sshfs可以临时用来快速访问远程文件。更详细的sshfs，参见 <https://linux.cn/article-7855-1.html>。
