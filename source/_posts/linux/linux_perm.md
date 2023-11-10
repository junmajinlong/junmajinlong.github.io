---
title: Linux上文件的权限管理
p: linux/linux_perm.md
date: 2019-07-06 18:20:42
tags: Linux
categories: Linux
---


--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------


# 文件/目录的权限

## 文件的权限

每个文件都有其所有者(u:user)、所属组(g:group)和其他人(o:other)对它的操作权限，a:all则同时代表这3者。权限包括读(r:read)、写(w:write)、执行(x:execute)。在不同类型的文件上读、写、执行权限的体现有所不同，所以目录权限和普通文件权限要区分开来。

**在普通文件上：**

r：可读，可以使用类似cat等命令查看文件内容；读是文件的最基本权限，没有读权限，普通文件的一切操作行为都被限制。

w：可写，可以编辑此文件；

x：可执行，表示文件可由特定的解释器解释并运行。可以理解为windows中的可执行程序或批处理脚本，双击就能运行起来的文件。

**在目录上：**

r：可以对目录执行ls以列出目录内的所有文件；读是文件的最基本权限，没有读权限，目录的一切操作行为都被限制。

w：可以在此目录创建或删除文件/子目录；

x：可进入此目录，可使用ls -l查看文件的详细信息。可以理解为windows中双击就进入目录的动作。

如果目录没有x权限，其他人将无法查看目录内文件属性(只能查看到文件类型和文件名，至于为什么，见后文)，所以一般目录都要有x权限。而如果只有执行却没有读权限，则权限拒绝。

一般来说，普通文件的默认权限是644(没有执行权限)，目录的默认权限是755(必须有执行权限，否则进不去)，链接文件的权限是777。当然，默认文件的权限设置方法是可以通过umask值来改变的。

##  权限的表示方式

权限的模式有两种体现：8进制数值体现方式和字符体现方式。

权限的数字表示："-"代表没有权限,用0表示。

​                r-----4

​                w-----2

​                x-----1

例如：`rwx rw- r--`对应的数字权限是764，732代表的权限数值表示为`rwx -wx -w-`。

![](/img/referer.jpg)

## chmod修改权限

能够修改权限的人只有文件所有者和超级管理员。

```
chmod [OPTION]... MODE[,MODE]... FILE...
chmod [OPTION]... num_mode FILE...
chmod [OPTION]... --reference=RFILE FILE...

选项说明：
--reference=RFILE：引用某文件的权限作为权限值
-R：递归修改，只对当前已存在的文件有效
```

(1). 使用数值方式修改权限

```
$ chmod 755 /tmp/a.txt
```

(2). 使用字符方式修改权限

由于权限属性附在文件所有者、所属组和其它上，它们三者都有独立的权限位，所有者使用字母"u"表示，所属组使用"g"来表示，其他使用"o"来表示，而字母"a"同时表示它们三者。所以使用字符方式修改权限时，需要指定操作谁的权限。

```
chmod [ugoa][+ - =] [权限字符] 文件/目录名
"+"是加上权限，"-"是减去权限，"="是直接设置权限
```

```
# 将ugo都去掉x权限，等价于chmod -x test
[root@xuexi tmp]# chmod u-x,g-x,o-x test
# 为ugo都加上x权限，等价于chmod +x test
[root@xuexi tmp]# chmod a+x test
```

## chgrp

更改文件和目录的所属组，要求组已经存在。

注意，对于链接文件而言，修改组的作用对象是链接的源文件，而非链接文件本身。

```
chgrp [OPTION]... GROUP FILE...
chgrp [OPTION]... --reference=RFILE FILE..

选项说明：
-R：递归修改
--reference=dest_file file_list：引用某文件的group作为文件列表的组,
                                 即将file文件列表的组改为dest_file的组
```

## chown

chown可以修改文件所有者和所属组。

注意，对于链接文件而言，默认不会穿过链接修改源文件，而是直接修改链接文件本身，这和chgrp的默认是不一样的。

```
chown [OPTION]... [OWNER][:[GROUP]] FILE...
chown [OPTION]... [OWNER][.[GROUP]] FILE...
chown [OPTION]... --reference=RFILE FILE...

选项说明：
--from=CURRENT_OWNER:CURRENT_GROUP：只修改当前所有者或所属组为此处指定的值的文件
--reference=RFILE：引用某文件的所有者和所属组的值作为新的所有者和所属组
-R：递归修改。注意，当指定-R时，且同时指定下面某一个选项时对链接文件有不同的行为
       -H：如果chown的文件参数是一个链接到目录的链接文件，则修改其源目录所有者和所属组
       -L：目录中遇到的所有链接文件都穿越过去，修改它们的源文件的所有者和所属组
       -P：不进行任何穿越，只修改链接文件本身的所有者和所属组。(这是默认值)
       这3项若同时指定多项时，则最后一项生效
```

chown指定所有者和所属组的方式有两种，使用冒号和点。

```
$ chown root.root test
$ chown root:root test
$ chown root test     # 只修改所有者
$ chown :root test    # 自修改组
$ chown .root test
```

![](/img/referer.jpg)

# 实现权限的本质

涉及文件系统的知识点，若不理解，可以先看看文件系统的内容。此处是以ext4文件系统为例的，在其他文件系统上结果可能会有些不一样(centos 7上使用xfs文件系统时结果可能就不一样)，但本质是一样的。

不同的权限表示对文件具有不同能力，如读写执行(rwx)权限，但是它是怎么实现的呢？描述文件权限的数据放在哪里呢？

首先，权限的元数据放在inode中，严格地说是放在inode table中，因为每个块组的所有inode组成一个inode table。在inode table中使用一列来存放数字型的权限，比如某文件的权限为644。每次用户要对文件进行操作时系统都会先查看权限，确认该用户是否有对应的权限来执行操作。当然，inode table一般都已经加载到内存中，所以每次查询权限的资源消耗是非常小的。

![](/img/linux/733013-20170615002744759-1390955567.jpg)

无论是读、写还是执行权限，所体现出来的能力究其本质都是因为它作用在对应文件的data block上。

## 读权限(r)

对普通文件具有读权限表示的是具有读取该文件内容的能力，对目录具有读权限表示具有浏览该目录中文件或子目录的能力。其本质都是具有读取其data block的能力。

对于普通文件而言，能够读取文件的data block，而普通文件的data block存储的直接就是数据本身， 所以对普通文件具有读权限表示能够读取文件内容。

对于目录文件而言，能够读取目录的data block，而目录文件的data block存储的内容包括但不限于：目录中文件的inode号(并非直接存储，而是存储指向inode table中该inode号的指针)以及这些文件的文件类型、文件名。所以能够读取目录的data block表示仅能获取到这些信息。

目录的data block内容示例如下：

![](/img/linux/733013-20170615002927931-890548759.jpg)

例如：

```
$ mkdir -p /mydata/data/testdir/subdir  # 创建testdir测试目录和其子目录subdir
$ touch /mydata/data/testdir/a.log      # 再在testdir下创建一个普通文件
$ chmod 754 /mydata/data/testdir        # 将testdir设置为对其他人只有读权限
```

然后切换到普通用户查看testdir目录的内容。

```
$ su - wangwu

$ ll -ai /mydata/data/testdir/
ls: cannot access /mydata/data/testdir/..: Permission denied
ls: cannot access /mydata/data/testdir/a.log: Permission denied
ls: cannot access /mydata/data/testdir/subdir: Permission denied
ls: cannot access /mydata/data/testdir/.: Permission denied
total 0
? d????????? ? ? ? ?            ? .
? d????????? ? ? ? ?            ? ..
? -????????? ? ? ? ?            ? a.log
? d????????? ? ? ? ?            ? subdir
```

从结果中看出，testdir下的文件名和文件类型是能够读取的，但是其他属性都不能读取到。而且也读取不到inode号，因为它并没有直接存储inode号，而是存储了指向Inode号的指针，要定位到指针的指向需要执行权限。

## 执行权限(x)

执行权限表示的是能够执行。如何执行？执行这个词不是很好解释，可以简单的类比Windows中的双击行为。例如对目录双击就能进入到目录，对批处理文件双击就能运行(有专门的解释器解释)，对可执行程序双击就能运行等。

当然，读权限是文件的最基本权限，执行权限能正常运行必须得配有读权限。

![](/img/referer.jpg)

对目录有执行权限，表示可以通过目录的data block中指向文件inode号的指针定位到inode table中该文件的inode信息，所以可以显示出这些文件的全部属性信息。

## 写权限(w)

写权限很简单，就是能够将数据写入分配到的data block。

对目录文件具有写权限，表示能够创建和删除文件。目录的写操作实质是能够在目录的data block中创建或删除关于待操作文件的记录。它要求对目录具有执行权限，因为无论是创建还是删除其内文件，都需要将其data block中inode号和inode table中的inode信息关联或删除。

对普通文件具有写权限，实质是能够改写该文件的data block。

还是要说明的是，对文件有写权限不代表能够删除该文件，因为删除文件是要在目录的data block中删除该文件的记录，也就是说删除权限是在目录中定义的。

 

所以，对目录文件和普通文件而言，读、写、执行权限它们的依赖关系如下图所示。

![](/img/linux/733013-20180301141708456-267142161.jpg)



# umask说明

umask值用于设置用户在创建文件时的默认权限。对于root用户(实际上是UID小于200的user)，系统默认的umask值是022；对于普通用户和系统用户，系统默认的umask值是002。

默认它们的设置是写在/etc/profile和/etc/bashrc两个环境配置文件中。

```
$ grep -C 5 -R 'umask 002'  /etc | grep 'umask 022'  
/etc/bashrc-       umask 022
/etc/csh.cshrc-    umask 022
/etc/profile-      umask 022
```

相关设置项如下：

```
if [ $UID -gt 199 ] && [ "`id -gn`" = "`id -un`" ]; then
   umask 002
else
   umask 022
fi
```

执行umask命令可以查看当前用户的umask值。

```
[root@xuexi tmp]# umask
0022
[longshuai@xuexi tmp]$ umask
0002
```

执行umask num可以临时修改umask值为num，但这是临时的，要永久有效，需要写入到环境配置文件中，至于写入到/etc/profile、/etc/bashrc、~/.bashrc还是~/.bash_profile中，看你自己的需求了。不过一般来说，不会去永久修改umask值，只会在特殊条件下临时修改下umask值。

umask是如何决定创建文件的默认权限的呢？

如果创建的是目录，则使用777-umask值，如root的umask=022，则root创建目录时该目录的默认权限为777-022=755，而普通用户创建目录时，权限为777-002=775.

![](/img/referer.jpg)

如果创建的是普通文件，在Linux中，深入贯彻了一点：文件默认不应该有执行权限，否则是危险的。所以在计算时，可能会和想象中的结果不一样。如果umask的三位都为偶数，则直接使用666去减掉umask值，因为6减去一个偶数还是偶数，任何位都不可能会有执行权限。如root创建普通文件时默认权限为666-022=644，而普通用户创建普通文件时默认权限为666-002=664。

如果umask值某一位为奇数，则666减去umask值后再在奇数位上加1。如umask=021时，创建文件时默认权限为666-021=645，在奇数位上加1，则为646。

```
[longshuai@xuexi tmp]$ umask 021
[longshuai@xuexi tmp]$ touch b.txt
[longshuai@xuexi tmp]$ ls -l b.txt
-rw-r--rw- 1 longshuai longshuai 0 Jun  7 12:02 b.txt
```

总之计算出后默认都是没有执行权限的。



# 文件的扩展ACL权限

在计算机相关领域，所有的ACL(access control list)都表示访问控制列表。

文件的owner/group/others的权限就是一种ACL，它们是基本的ACL。很多时候，只通过这3个权限位是无法完全合理设置权限问题的，例如如何仅设置某单个用户具有什么权限。这时候需要使用扩展ACL。

扩展ACL是一种特殊权限，它是文件系统上功能，用于解决所有者、所属组和其他这三个权限位无法合理设置单个用户权限的问题。所以，扩展ACL可以针对单一使用者，单一档案或目录里的默认权限进行r,w,x的权限规范。

需要明确的是，扩展ACL是文件系统上的功能，且工作在内核，默认在ext4/xfs上都已开启。

在下文中，都直接以ACL来表示代替扩展ACL的称呼。

## 查看文件系统是否开启ACL功能

对于ext家族的文件系统来说，要查看是否开启acl功能，使用dumpe2fs导出文件系统属性即可。

```
$ dumpe2fs -h /dev/sda2 | grep -i acl
dumpe2fs 1.41.12 (17-May-2010)
Default mount options:    user_xattr acl
```

对于xfs文件系统，则没有直接的命令可以输出它的相关信息，需要使用dmesg来查看。其实无需关注它，因为默认xfs会开启acl功能。

```
$ dmesg | grep -i acl
[    1.465903] systemd[1]: systemd 219 running in system mode. (+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ -LZ4 -SECCOMP +BLKID +ELFUTILS +KMOD +IDN)
[    2.517705] SGI XFS with ACLs, security attributes, no debug enabled
```

开启ACL功能后，不代表就使用ACL功能。是否使用该功能，不同文件系统控制方法不一样，对于ext家族来说，通过mount挂载选项来控制，而对于xfs文件系统，mount命令根本不支持acl参数(xfs文件系统如何关闭或启用的方法本人也不知道)。

## 设置和查看ACL

设置使用setfacl命令。

```
setfacl [options] u:[用户列表]:[rwx] 目录/文件名    # 对用户设置使用u
setfacl [options] g:[组列表]:[rwx]   目录/文件名    # 对组设置使用g

选项说明：
-m：设定ACL权限(modify)
-x：删除指定的ACL权限，可以指定用户、组和文件来删除(remove)
-M：写了ACL条目的文件，将从此文件中读取ACL条目，需要配合-m，所以-M指定的是modify file
-X：写了ACL条目的文件，将从此文件中读取ACL条目，需要配合-x，所以-X指定的是remove file
-n：不重置mask
-b：删除所有的ACL权限
-d：设定默认ACL权限，只对目录有效，设置后子目录(文件)继承默认ACL，只对未来文件 有效
-k：删除默认ACL权限
-R：递归设定ACL权限，只对目录有效，只对已有文件有效
```

查看使用getfacl命令
```
getfacl filename
```

案例：假设现有目录/data/videos专门存放视频，其中有一个a.avi的介绍性视频。该目录的权限是750。现在有一个新用户加入，但要求该用户对该目录只有查看的权限，且只能看其中一部视频a.avi，另外还要求该用户在此目录下没有创建和删除文件的权限。

 ![](/img/linux/733013-20170615003624071-2124860824.jpg)

1.准备相关环境。

```
$ mkdir -p /data/videos
$ chmod 750 /data/videos
$ touch /data/videos/{a,b}.avi
$ echo "xxx" >/data/videos/a.avi
$ echo "xxx" >/data/videos/b.avi
$ chown -R root.root /data/videos
```

2.首先设置用户longshuai对/data/videos目录有读和执行权限。

```
$ setfacl -m u:longshuai:rx /data/videos
```

3.现在longshuai对/data/videos目录下的所有文件都有读权限，因为默认文件的权限为644。要设置longshuai只对a.avi有读权限，先设置所有文件的权限都为不可读。

```
$ chmod 640 /data/videos/*
```

4.然后再单独设置a.avi的读权限。

```
$ setfacl -m u:longshuai:r /data/videos/a.avi
```

到此就设置完成了。查看/data/videos/和/data/videos/a.avi上的ACL信息。

```
$ getfacl /data/videos/
getfacl: Removing leading '/' from absolute path names
# file: data/videos/
# owner: root
# group: root
user::rwx
user:longshuai:r-x         # 用户longshuai在此文件上的权限是r-x
group::r-x
mask::r-x
other::---

$ getfacl /data/videos/a.avi
getfacl: Removing leading '/' from absolute path names
# file: data/videos/a.avi
# owner: root
# group: root
user::rw-
user:longshuai:r--         # 用户longshuai在此文件上的权限是r--
group::r--
mask::r--
other::---
```

![](/img/referer.jpg)

## ACL:mask

设置mask后会将mask权限与已有的acl权限进行与计算，计算后的结果会成为新的ACL权限。

设定mask的方式为：

```
setfacl -m m:[rwx] 目录/文件名
```

![](/img/linux/733013-20170615004000853-1479223179.jpg)

注意：默认每次设置文件的acl都会重置mask为此次给定的用户的值。既然如此，要如何控制文件上的acl呢？如果一个文件上要设置多个用户的acl，重置mask后就会对已有用户的acl重新计算，而使得acl权限得不到有效的控制。使用setfacl的"-n"选项，它表示此次设置不会重置mask值。

例如：

当前的acl权限：

```
$ getfacl /data/videos                     
getfacl: Removing leading '/' from absolute path names
# file: data/videos
# owner: root
# group: root
user::rwx
user:longshuai:rwx
group::r-x
mask::rwx
other::---
```

设置mask值为rx。

```
$ setfacl -m m:rx /data/videos

$ getfacl /data/videos       
getfacl: Removing leading '/' from absolute path names
# file: data/videos
# owner: root
# group: root
user::rwx
user:longshuai:rwx              #effective:r-x
group::r-x
mask::r-x
other::---
```

设置mask后，它提示有效权限是r-x。这是rwx和r-x做与运算之后的结果。

再设置longshuai的acl为rwx，然后查看mask，会发现mask也被重置为rwx。

```
$ setfacl -m u:longshuai:rwx /data/videos

$ getfacl  /data/videos
getfacl: Removing leading '/' from absolute path names
# file: data/videos
# owner: root
# group: root
user::rwx
user:longshuai:rwx
group::r-x
mask::rwx
other::---
```

所以，在设置文件的acl时，要使用-n选项来禁止重置mask。

```
$ setfacl -m m:rx /data/videos

$ setfacl -n -m u:longshuai:rwx /data/videos

$ getfacl  /data/videos
getfacl: Removing leading '/' from absolute path names
# file: data/videos
# owner: root
# group: root
user::rwx
user:longshuai:rwx              #effective:r-x
group::r-x
mask::r-x
other::---
```

## 设置递归和默认ACL权限

递归ACL权限只对目录里已有文件有效，默认权限只对未来目录里的文件有效。

设置递归ACL权限：

```
setfacl -m u:username:[rwx] -R 目录名
```

设置默认ACL权限：

```
setfacl -m d:u:username:[rwx] 目录名
```

## 删除ACL权限

```
setfacl -x u:用户名 文件名       # 删除指定用户ACL
setfacl -x g:组名 文件名         # 删除指定组名ACL
setfacl -b 文件名                # 指定文件删除ACL，会删除所有ACL
```



# 文件隐藏属性

- chattr：change file attributes  
- lsattr：list file attributes  

```
chattr [+ - =] [ai] 文件或目录名
```

常用的参数是a(append，追加)和i(immutable，不可更改)，其他参数略。

设置了a参数时，文件中将只能增加内容，不能删除数据，且不能打开文件进行任何编辑，哪怕是追加内容也不可以，所以像sed等需要打开文件的再写入数据的工具也无法操作成功。文件也不能被删除。只有root才能设置。

设置了i参数时，文件将被锁定，不能向其中增删改内容，也不能删除修改文件等各种动作。只有root才能设置。可以将其理解为设置了i后，文件将是永恒不变的了，谁都不能动它。

例如，对/etc/shadow文件设置i属性，任何用户包括root将不能修改密码，而且也不能创建用户。

```
$ chattr +i /etc/shadow
```

此时如果新建一个用户。

```
$ useradd newlongsuai
$ useradd: cannot open /etc/shadow  # 提示文件不能打开，被锁定了
```

lsattr查看文件设置的隐藏属性。

```
$ lsattr /etc/shadow
----i--------e- /etc/shadow   # i说明被锁定了，e是另一种文件属性，忽略它
```

删除隐藏属性：

```
$ chattr -i /etc/shadow
$ lsattr /etc/shadow
-------------e- /etc/shadow
```

再来一例：

```
$ chattr +a test1.txt        # 对test1.txt设置a隐藏属性
$ echo 1234>>test1.txt       # 追加内容是允许的行为
$ cat /dev/null >test1.txt   # 但是清空文件内容是不允许的
-bash: test1.txt: Operation not permitted
```

![](/img/referer.jpg)

# suid/sgid/sbit

## suid

suid只针对可执行文件，即二进制文件。它的作用是对某个命令(可执行文件)授予所有者的权限，命令执行完成权限就消失。一般是提权为root权限。

例如/etc/shadow文件所有人都没有权限(root除外)，其他用户连看都不允许。

```
$ ls -l /etc/shadow
----------. 1 root root 752 Apr  8 12:42 /etc/shadow
```

但是他们却能修改自己的密码，说明他们一定有一定的权限。这个权限就是suid控制的。

```
$ ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root 30768 Feb 22  2012 /usr/bin/passwd
```

其中的"s"权限就是suid，它出现在所有者位置上(是root)，其他用户执行passwd命令时，会暂时拥有所有者位的rwx权限，也就是root的权限，所以能向/etc/shadow写入数据。

suid必须和x配合，如果没有x配合，则该suid是空suid，仍然没有执行命令的权限，所有者都没有了x权限，suid依赖于它所以更不可能有x权限。空的suid权限使用大写的"S"表示。

数字4代表suid，如4755。

## sgid

针对二进制文件和目录。

- 针对二进制文件时，权限升级为命令的所属组权限。
- 针对目录时，目录中所建立的文件或子目录的组将继承默认父目录组，其本质还是提升为目录所属组的权限。此时目录应该要有rx权限，普通用户才能进入目录，如果普通用户有w权限，新建的文件和目录则以父目录组为默认组。

以2代表sgid，如2755，和suid组合如6755。

## sbit

只对目录有效。对目录设置sbit，将使得目录里的文件只有所有者能删除，即使其他用户在此目录上有rwx权限，即使是root用户。

以1代表sbit。

 

**2017-08-17补充：suid/sgid/sbit的标志位都作用在x位，当原来的x位有x权限时，这些权限位则为s/s/t，如果没有x权限，则变为S/S/T。例如，/tmp目录的权限有个t位，使得该目录里的文件只有其所有者本身能删除。**










