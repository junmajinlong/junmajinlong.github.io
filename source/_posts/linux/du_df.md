---
title: 详细分析du和df的统计结果为什么不一样
p: linux/du_df.md
date: 2019-07-06 18:20:42
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------


# 详细分析du和df的统计结果为什么不一样

今天有个人问我du和df的统计结果为什么会不同。给他解析了一番，想想还是写篇文章从原理上来分析分析。

我们常常使用du和df来获取目录或文件系统已占用空间的情况。但它们的统计结果是不一致的，大多数时候，它们的结果相差不会很大，但有时候它们的统计结果会相差非常大。

例如：

```
##### df的统计结果
[root@xuexi ~]# df -hT 
Filesystem          Type   Size  Used Avail Use% Mounted on
/dev/sda2           ext4    18G  1.7G   15G  11% /
tmpfs               tmpfs  491M     0  491M   0% /dev/shm
/dev/sda1           ext4   239M   68M  159M  30% /boot
//192.168.0.124/win cifs   381G  243G  138G  64% /mnt

##### du对根目录的统计结果
[root@xuexi ~]# du -sh /  2>/dev/null
244G    /
```

df中"/"的使用空间是1.7G，但是du的结果却是244G。这里du的统计结果大于df。

再看看对/boot分区的统计结果。

```
[root@xuexi ~]# df -hT /boot;echo;du -sh /boot
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda1      ext4  239M   68M  159M  30% /boot

66M     /boot
```

du的结果是66M，df的结果是68M，相差不大，但df的结果大于du。

![](/img/referer.jpg)

<a name="blog1"></a>

## 1.文件删除的底层过程

这里简单说明下文件系统相关的底层机制，详细的内容参见：[ext文件系统机制](/linux/ext_filesystem)。

首先说明下文件是怎么存储到文件系统中的。假如要存储a.txt到/tmp目录下。

![](/img/linux/733013-20180327171608567-1495159252.jpg)

当a.txt文件要存储到/tmp下时：  
- (1).首先从inode table中找一个空闲的inode号分配给a.txt，例如2222。再将inode map(imap)中2222这个inode号标记为已使用。  
- (2).在/tmp的data block中添加一条a.txt文件的记录。该记录中包括一个指向inode号的指针，例如"0x2222"。  
- (3).然后从block map(bmap)中找出空闲的data block，并开始将a.txt中的数据写入到data block中。每写一段空间(每次分配一段空间)就从bmap中找一次空闲的data block，直到存完所有数据。  
- (4).设置inode table中关于2222这条记录的data block指针，通过该指针可以找到a.txt使用了哪些data block。

当要删除a.txt文件时：  
- (1).在inode table中删除指向a.txt的data block指针。这里只要一删除，外界就找不到a.txt的数据了。但是这个文件还存在，只是它是被"损坏"的文件，因为没有任何指针指向数据块。  
- (2).在imap中将2222的inode号标记为未使用。于是这个inode号就被释放，可以被后续的文件重用。  
- (3).删除父目录/tmp的data block中关于a.txt的记录。这里只要一删除，外界就看不到也找不到这个文件了。  
- (4).在bmap中将a.txt占用的block标记为未使用。这里被标记为未使用后，这些data block就可以被后续文件覆盖重用。  

考虑一种情况，当一个文件被删除时，但此时还有进程在使用这个文件，这时是怎样的情况呢？**外界是看不到也找不到这个文件的，所以删除的过程已经进行到了第(3)步。但进程还在使用这个文件的数据，也能找到这个文件的数据，是因为进程在加载这个文件的时候就已经获取到了该文件占用哪些data block，虽然删除了文件，但bmap中这些data block还没有标记为未使用。**

<a name="blog2"></a>

## 2.du统计的原理

du是通过stat命令来统计每个文件(包括子目录)的空间占用总和。因为会对每个涉及到的文件使用stat命令，所以速度较慢。

**1.如果统计目录下挂载了其他文件系统，那么也会对这个文件系统进行统计。**

例如"du -sh /"的时候，统计了所有分区的文件，包括挂载上来的。正如本文开头统计的"/"一样，du的结果是244G，明显比df统计的结果大，就是因为将某个分区挂载到了/mnt目录下。

```
##### df的统计结果
[root@xuexi ~]# df -hT 
Filesystem          Type   Size  Used Avail Use% Mounted on
/dev/sda2           ext4    18G  1.7G   15G  11% /
tmpfs               tmpfs  491M     0  491M   0% /dev/shm
/dev/sda1           ext4   239M   68M  159M  30% /boot
//192.168.0.124/win cifs   381G  243G  138G  64% /mnt

##### du对根目录的统计结果
[root@xuexi ~]# du -sh /  2>/dev/null
244G    /
```

**2.如果文件被删除，即使被其他进程引用了，du命令也无法对其统计。因为stat命令找不到这个文件。**

**3.可以跨分区统计某些你想统计的文件大小总和。因为它们都能被stat找到并统计。**

例如：

统计Linux下所有img文件的大小。

```
[root@xuexi ~]# find / -type f -name "*.img" -print0 | xargs -0 du -csh 
19M     /boot/initramfs-2.6.32-504.el6.x86_64.img
13M     /mnt/linux工具/cirros-0.3.4-x86_64-disk.img
31M     total
```
这里统计的两个img文件就是在不同分区内的。

<a name="blog3"></a>

## 3.df统计的原理

df是读取每个分区的superblock来获取空闲数据块、已使用数据块，从而计算出空闲空间和已使用空间，因此df统计的速度极快(superblock才占用1024字节)。

**1.当某个文件系统下挂载了其他分区，df不会把这个分区也统计进去。**

这很容易理解，因为df读取的是各自分区的superblock，即使分区1挂载在分区0的目录下，df统计分区0的时候，也只能读取分区0的superblock。

例如，下面的/mnt、/boot都没有统计在"/"中。
```
[root@xuexi ~]# df -hT 
Filesystem          Type   Size  Used Avail Use% Mounted on
/dev/sda2           ext4    18G  1.7G   15G  11% /
tmpfs               tmpfs  491M     0  491M   0% /dev/shm
/dev/sda1           ext4   239M   68M  159M  30% /boot
//192.168.0.124/win cifs   381G  243G  138G  64% /mnt
```

**2.由于df每次统计都是读取superblock，所以df对文件系统中的某个文件进行统计时，会自动转为统计这个文件系统的信息。**

```
[root@xuexi ~]# df -hT /etc/fstab
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda2      ext4   18G  1.7G   15G  11% /
```

**3.df会统计因子挂载点而被隐藏的原目录文件大小。**

如果在/mnt目录下有3G的文件，然后在/mnt上挂载了其他文件系统，/mnt下原本那3G的文件就被隐藏起来无法访问，du当然无法统计这部分数据大小(但du会统计挂载在/mnt上的文件)，但df会统计这部分信息。

**4.df会统计已删除但却仍有进程引用的文件。**

正常情况下，删除文件会立刻释放相关指针，并将imap和bmap中相关的位图标记为未使用。**bmap只要一改变，文件系统立刻就能知道每个块组中哪些数据块是空闲的，哪些数据块是被使用的，这些信息都会更新到分区的superblock中**。于是df能立刻统计到实时的空间信息。

但是当一个文件被删除时，如果还有进程在引用这个文件，根据前文的分析，bmap中不会将这个文件的data block标记为未使用，也就不会将数据块的使用情况更新到superblock中。由于df是根据superblock中空闲和使用数据块的数量来计算空闲空间和已使用空间的，所以df统计的时候会将这个已被"删除"的文件统计到已使用空间中。

![](/img/referer.jpg)

例如，创建一个较大一点的文件放在"/"目录下，并du和df统计根目录的已使用空间。

```
[root@xuexi ~]# dd if=/dev/zero of=/my.iso bs=1M count=1000

[root@xuexi ~]# df -hT /
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda2      ext4   18G  2.7G   14G  17% /

[root@xuexi ~]# du -sh --exclude="/mnt" / 2>/dev/null
2.7G    /
```
它们在GB级的单位上是相等的。

现在使用一个进程来引用这个文件，然后删除这个文件，再du和df统计。

```
[root@xuexi ~]# tail -f /my.iso &

[root@xuexi ~]# rm -rf /my.iso 
[root@xuexi ~]# ls /my.iso
ls: cannot access /my.iso: No such file or directory

[root@xuexi ~]# du -sh --exclude="/mnt" / 2>/dev/null
1.8G    /

[root@xuexi ~]# df -hT /
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda2      ext4   18G  2.7G   14G  17% /
```

可以发现，外界已经获取不到my.iso文件了，所以du无法统计这个文件。而df却将该文件大小统计进去了，因为my.iso占用的data block还未被标记为未使用。

再关掉tail进程，然后df再统计空间，结果将和du一样显示为正常的大小。

```
[root@xuexi ~]# jobs
[1]+  Running                 tail -f /my.iso &
[root@xuexi ~]# kill %1

[root@xuexi ~]# df -hT /
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda2      ext4   18G  1.7G   15G  11% /
```

如果不知道文件系统中哪些已被删除，但却还被进程引用的文件，可以使用lsof来获取。通过它还能获取到文件的大小，看看到底是哪个文件在"占着茅坑以及占了多少茅坑"。

例如，关掉tail进程前，使用lsof查看。可以看到tail进程占用了/my.iso，且这个文件的大小为1048576000字节。

```
[root@xuexi ~]# lsof | grep deleted   
php-fpm   12597      root  txt     REG   8,2    4058416   931143 /usr/sbin/php-fpm (deleted)
php-fpm   12657    nobody  txt     REG   8,2    4058416   931143 /usr/sbin/php-fpm (deleted)
php-fpm   12707    nobody  txt     REG   8,2    4058416   931143 /usr/sbin/php-fpm (deleted)
php-fpm   12708    nobody  txt     REG   8,2    4058416   931143 /usr/sbin/php-fpm (deleted)
tail      14437      root    3r    REG   8,2 1048576000     7171 /my.iso (deleted)
```

经过上面的分析，想必对du和df的结果不会再有任何疑惑了吧。


