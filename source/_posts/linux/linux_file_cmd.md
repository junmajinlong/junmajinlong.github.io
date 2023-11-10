---
title: Linux文件类基础命令
p: linux/linux_file_cmd.md
date: 2019-07-06 18:20:42
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux文件类基础命令

本文介绍Linux中最基础的内容，包括文件相关的一些基础知识和一些最常用的命令。

## 关于文件路径

Linux中分绝对路径和相对路径，绝对路径一定是从`/`开始写的，相对路径不从根开始写，还可能使用路径符号。

路径展开符号：
```
.  ：(一个点)表示当前目录
.. ：(两个点)表示上一层目录
-  ：(一个短横线)表示上一次使用的目录，例如从/tmp直接切换到/etc下，"-"就表示/tmp
~  ：(波浪符号)表示用户的家目录，例如"~account"表示account用户的家目录
/dir/和/dir：一般都表示dir目录和dir目录中的文件。但在有些地方会严格区分是否加尾
             随斜线，此时对于加了尾随斜线的表示此目录中的文件，不加尾随斜线的表示
             该目录本身和此目录中的文件
```
切换路径用`cd`命令；

显示当前所在目录用`pwd`命令。若当前所在目录为链接目录，使用`pwd`显示的将是链接自身，使用`-P`选项将定位到链接的原始目录。

```bash
[root@xuexi ~]# ll ; cd tmp; pwd; pwd -P
total 0
lrwxrwxrwx 1 root root 4 May 30 19:17 tmp -> /tmp
/root/tmp
/tmp
```

获取文件名使用`basename`命令，获取文件所在目录使用`dirname`命令。注意，这两个命令其实不太完善，它不会检查文件或目录是否存在，只要写出来了就会去获取。

```bash
[root@xuexi tmp]# basename /etc/shadow
shadow
[root@xuexi tmp]# basename /etc/
etc
[root@xuexi tmp]# dirname /etc/shadow
/etc
[root@xuexi tmp]# dirname /etc/    # 对目录使用dirname获取的是上级目录
/
[root@server1 ~]# dirname /kalsldk/kdkskks/djfjdjdjsj   # 获取不存在的目录
/kalsldk/kdkskks
```

## 关于文件路径通配符

可以使用`* ? []`等通配符来扩展路径或文件名。例如，`ls *.log`将列出当前路径下所有以`.log`字符结尾的文件名 (但不包括`.`开头的隐藏文件)。

默认情况下，bash提供的通配符规则比较弱，例如`*`无法匹配文件名开头的"."，无法匹配路径分隔符号(即斜线"/")，但可以通过set或shopt命令开启额外的通配功能，实现更完善的通配符规则。

例如，默认情况下，想要匹配目录`/path`下所有隐藏文件和非隐藏文件，如下：

```bash
# 但这样也会匹配到两个特殊的文件 . 和 .. 
ls  .*  *
```

开启dotglob功能，`* ?`就可以匹配以"."开头的文件(但不包括`.和..`)：

```bash
# 开启bash的dotglob功能
shopt -s dotglob
ls *

# 关闭bash的dotglob功能
shopt -u dotglob
```

有时想要递归到目录内部，又想要匹配文件名，例如想要递归找出多层目录/path下所有的".css"文件，这时可以开启globstar功能，使用`**`就可以匹配匹配路径斜线。

```bash
shopt -s globstar       # 开启星号匹配模式
ls /path/**/*.css       # 开启后，使用两个星号**就会匹配斜线
```

必须要说明的是，对于非bash内置命令，有些可能也提供了自己的通配符匹配方式，它们的通配模式和shell提供的可能并不一样。例如find的"-name"选项就可以采用自己的通配符，它的星号`*`可以匹配以点开头的隐藏文件，如`find /var/log -name "*.log" `。

## 查看目录内容(ls和tree)

ls命令列出目录中的内容，和dir命令完全等价。tree命令按树状结构递归列出目录和子目录中的内容，而ls使用-R选项时才会递归列出。

注意：ls的结果中是以制表符分隔多个文件的。

### ls命令

ls 的各个选项说明如下：

```
-l：(long) 长格式显示，即显示属性等信息 (包括mtime)。
    注意：显示的目录大小是节点所占大小。像win一样计算
    目录大小时包括文件大小要用du -sh
-c：列出ctime
-u：列出atime
-d：(direcorty) 查看目录本身属性信息，不查看目录里面的东西。
    不加-d会查看里面文件的信息
-a：会显示所有文件，包括两个相对路径的文件"."和".."以及
    以点开头的隐藏文件
-A：会列出绝大多数文件，即忽略两个相对路径的文件"."和".."
-h：(human) 人类可读的格式，将字节换成k, 将K换成M，将M换成G
-i：(inode) 权限属性的前面加上一堆数字
-p：对目录加上/标识符以作区分
-F：对不同类型的文件加上不同标识符以作区分，对目录加的文件也是/
-t：按修改时间排序内容。不加任何改变顺序的选项时，默认字母顺序
-r：反转排序
-R：递归显示
-S：按文件大小排序，默认降序排序
--color：显示颜色
-m：使用逗号分隔各文件，当然，只适用于未使用长格式ls -l的情况
-1：(数字一)，以换行符分隔文件，和-m或-l(L的小写字母)是冲突
-I pattern：忽略被pattern匹配到的文件
```

注意，`ls -h`显示文件大小时，一般显示的都是不带B的单位，如K/M/G，它们的转换比例是1024，如果显示的都是带了B的，如KB/MB/GB，则它们的转换比例为1000而非1024，一般很少显示带B的大小。

不得不说，ls本身不能显示出文件的全路径名是一大缺陷，不过好在使用find命令可以很简单的就获取到。

以下是使用ls -l显示文件长格式的属性。

```bash
[root@xuexi ~]# ll /tmp
drwxr-xr-x  2 root root 4096 Mar 26 16:44 test1
```

可以查出7列属性。

![](/img/linux/733013-20170612214807150-1187771984.png)

### tree命令

有可能tree命令不存在，需要安装tree包才有：

```
yum -y install tree
```

tree 命令的选项说明如下：
```
【 匹配选项：】
-L：用于指定递归显示的深度，指定的深度必须是大于 0 的整数。
-P：用于显示通配符匹配模式的目录和文件，但是不管是否匹配，目录一定显示。
-I：用于显示除被通配符匹配外的所有目录和文件。

【 显示选项：】
-a：用于显示隐藏文件，默认不显示。
-d：指定只显示目录。
-f：指定显示全路径。
-i：不缩进显示。和 -f 一起使用很有用。
-p：用于显示权限位信息。
-h：用于显示大小。
-u：显示 username 或 UID(当没有 username 时只能显示 UID 了)。
-g：显示 groupname 或 GID。
-D：显示文件的最后一次 Mtime。
--inodes：显示 inode 号。
--device：显示文件或目录所属的设备号。
-C：显示颜色。

【 输出选项：】
-o filename：指定将 tree 的结果输出到 filename 文件中。
```

下图是一个较全的输出结果。

![](/img/linux/733013-20170612214809790-252855701.png)

<a name="file_timestamp"></a>

## 文件的时间戳 (atime/ctime/mtime)

文件的时间属性有三种：atime、ctime、mtime。

- atime是access time，即上一次的访问时间；

- mtime是modify time，是文件的修改时间；

- ctime是change time，也是文件的修改时间，只不过这个修改时间计算的inode修改时间，也就是元数据修改时间。

文件还有一个创建时间(create time)，大多数unix系统上都认为这是个无用的属性，一般工具无法获取这个时间，但是对于ext家族文件系统，通过它的底层调试工具debugfs可以获取create time。

对于普通文件而言，只有修改文件内容mtime才会改变，更准确的说是修改了它的data block部分；而ctime是修改文件属性时改变的，确切的说是修改了它的元数据部分，例如重命名文件、修改文件所有者、移动文件(移动文件没有改变data block，只是改变了其inode指针，或文件名)等。当然，修改文件内容时也一定会改变ctime(修改文件内容至少已经修改了inode记录上的mtime，这也是元数据)，所以mtime的改变一定会引起ctime的改变。

> 如果还不知道什么是文件的data block、inode等，暂且忽略即可

对目录而言，考虑目录文件的data block，可知在目录中创建、删除文件以及目录内其他任意文件操作都会改变mtime，因为目录里的任何东西都是目录中的内容；而目录的ctime，除了目录的mtime引起ctime改变之外，对目录本身的元数据修改也会改变ctime。

总结下：  
- (1).atime只在文件被打开访问时才改变，若不是打开文件编辑内容 (如重定向内容到文件中)，则ctime和mtime的改变不会引起atime的改变;  
- (2).mtime的改变一定引起ctime的改变，而访问文件时(例如cat)，atime不一定会改变，所以atime"改变"(这个改变是假象，见下文分析)不一定会影响ctime。(见下面的relatime的说明)

###  关于relatime

atime/ctime/mtime是Posix标准要求操作系统维护的时间戳信息。但是每次将atime、ctime和mtime写入到硬盘中(这些不会写入缓存，只要修改就是写入磁盘，即使从缓存读取文件内容也如此)效率很低。有多低？下图是写ctime消耗的时间，几乎总要花费零点几秒。

![](/img/linux/733013-20180907154025206-140625290.png)

mtime要被修改，必然是修改了文件内容，这时候将mtime写入到硬盘中是应该的。但是atime和ctime呢？很多情况下根本用不到atime和ctime，在频繁访问文件的时候，都要修改atime和ctime，这样效率会降低很多很多，所以mount有个noatime选项来避免这种负面影响。

CentOS6引入了一个新的atime维护机制relatime：除非两次修改atime的时间超过1天(默认设置86400秒)，或者修改了mtime，否则访问文件的inode不会引起atime的改变。换句话说，当cat一个文件的时候，它的atime可能会改变，但是你稍后再cat，它不会再改变。

由于cat文件的时候atime可能不会改变，所以可能也就不会引起ctime的改变。

relatime维护的atime是可以控制的，详见`man mount`的relatime和[redhat 官方手册](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/power_management_guide/relatime?tdsourcetag=s_pctim_aiomsg)。

## 文件/目录的创建和删除


### 创建目录mkdir

```
mkdir [-mp] 目录名
【选项】
-m：表示创建目录时直接设置权限
-p：表示递归创建多层目录，即上层目录不存在时也会直接将其创建出来 (parent)
```

示例：
```bash
# 在tmp目录中创建一个test1目录
[root@xuexi ~]# mkdir /tmp/test1

# 直接创建test2时就赋予权限711
[root@xuexi ~]# mkdir -m 711 /tmp/test2

# 创建test5，此时会将不存在的test3和test4目录也创建好
[root@xuexi ~]# mkdir -p /tmp/test3/test4/test5
```

### 创建文件touch

```bash
touch file_name
```
示例：
```bash
[root@xuexi ~]# touch /tmp/test1/test1.txt

# 创建文件名为1-10的文件
[root@xuexi ~]# touch {1..10}
```

多个`{}`还可以交换扩展。类似`(a+b)(c+d)=ac+ad+bc+bd`。

```bash
# 创建 a_c、a_d、b_c、b_d 四个文件
[root@xuexi ~]# touch {a,b}_{c,d}
```

touch主要是修改文件的时间戳信息，当touch的文件不存在时就自动创建该文件。可以使用`touch –c`来取消创建动作。

touch可以更改最近一次访问时间(atime)，最近一次修改时间(mtime)，文件属性修改时间(ctime)，这些时间可以通过命令`stat file`来查看。其中ctime是文件属性上的更改，即元数据的更改，比如修改权限。

`touch -a`修改atime，-m修改mtime，没有修改ctime的选项。因为使用touch改变atime或mtime，同时也都会改变ctime，虽说atime并不总是会影响 ctime(如cat 文件时)。

-t选项表示使用`[[CC]YY]MMDDhhmm[.ss]`格式的时间替代当前时间。

```bash
# 将file的atime修改为2012年12月21号12点12分
shell> touch -a -t 201212211212 file
```

-d选项表示使用指定的字符串描述时间格式来替代当前时间，如`3 days ago`，`next Sunday`等很多种格式。

所以，touch命令选项说明如下：
```
-c：强制不创建文件
-a：修改文件 access time(atime)
-m：修改文件 modification time(mtime)
-t：使用 "[[CC]YY]MMDDhhmm[.ss]" 格式的时间替代当前时间
-d：使用字符串描述的时间格式替代当前时间
```

### 删除文件/目录

```
rm [-rfi] file_name

-r：表示递归删除，删除目录时需要加此参数
-i：询问是否删除 (yes/no)
-f：强制删除，不进行询问
```
```
[root@xuexi ~]# rm -rf /tmp/test2
```

删除空目录时还可以使用rmdir。

在删除文件之前，一定一定要确定是否真的删除。最好使用`rm -i`(默认已经在`~/.bashrc`中定义了该别名)，除非在脚本中，否则不要轻易使用-f选项。已经有非常多的人不小心`rm -rf *`和`rm -rf /NNNNN`了。例如想删除`rm –rf /abc*`，结果习惯性的多敲了一个空格`rm –rf /abc *`，完了。

## 查看文件类型file命令

这是一个简单查看文件类型的命令，查看文件是属于二进制文件还是数据文件还是ASCII文件。

```bash
[root@xuexi tmp]# file /etc/aliases.db
/etc/aliases.db: Berkeley DB (Hash, version 9, native byte-order)  # 数据文件

[root@xuexi tmp]# file ~/.bashrc
/root/.bashrc: ASCII text    # ASCII文件

[root@xuexi tmp]# file /bin/ls
/bin/ls: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.18, stripped
```

除了基本的查看文件类型的功能外，`file`还有一个"-s"选项，是一个超强力的选项，可以查看设备的文件系统类型。像有些分区工具如parted在分区时是可以指定文件系统的 (虽然不建议这么做，CentOS 7的parted版本中已经取消了该功能)，但在分区后格式化前，一般是比较难查看该分区的文件系统类型的，但使用`file`可以查看到。

```bash
[root@server1 ~]# file -s /dev/sda1
/dev/sda1: Linux rev 1.0 ext4 filesystem data (needs journal recovery) (extents) (huge files)

[root@server1 ~]# file -s /dev/sda
/dev/sda: x86 boot sector; GRand Unified Bootloader, stage1 version 0x3, boot drive 0x80, 1st sector stage2 0x7f86, GRUB version 0.94; partition 1: ID=0x83, 
active, starthead 32, startsector 2048, 512000 sectors; partition 2: ID=0x83, starthead 254, startsector 514048, 37332992 sectors; partition 3: ID=0x82, 
starthead 254, startsector 37847040, 4096000 sectors, code offset 0x48
```

## 文件/目录复制和移动

### cp命令

```
cp [-apdriulfs] src dest # 复制单文件或单目录
cp [-apdriuslf] src1 src2 src3......dest_dir # 复制多文件、目录到一个目录下

选项说明：
-p：文件的属性 (权限、属组、时间戳) 也复制过去。如果不指定 -p 选项，
    谁执行复制动作，文件所有者和组就是谁。
-r 或 -R：递归复制，常用于复制非空目录。
-d：复制的源文件如果是链接文件，则复制链接文件而不是指向的文件本身。
    即保持链接属性，复制快捷方式本身。如果不指定 -d，则复制的是链接
    所指向的文件。
-a：a=pdr三个选项。归档拷贝，常用于备份。
-i：复制时如果目标文件已经存在，询问是否替换。
-u：(update)若目标文件和源文件同名，但属性不一样(如修改时间，大小等)，
    则覆盖目标文件。
-f：强制复制，如果目标存在，不会进行-i选项的询问和-u选项的考虑，直接覆盖。
-l：在目标位置建立硬链接，而不是复制文件本身。
-s：在目标位置建立软链接，而不是复制文件本身(软链接相当于windows的快捷方式)。
```

一般使用`cp -a`即可，对于目录加上`-r`选项即可。

注意，bash内置命令在进行通配符匹配文件的时候，`* ? []`是无法匹配到以"."开头的文件的，所以`*`不会匹配隐藏文件。要通配隐藏文件，可开启bash的`dotglob`功能。 

例如，复制/etc/skel目录下所有文件包括隐藏文件到/tmp目录下。
```bash
shopt -s dotglob
cp -a /etc/skel/* /tmp
```
如果有重复文件，则即使加上-f选项，也一样会交互式询问。解决方法可以是使用`yes`这个工具，它会不断的生成`y`字母直到进程被杀掉，当然也可以自行指定要生成的字符串。

```bash
yes | cp -a /etc/skel/. /tmp
```

### scp命令和执行过程分析

scp是基于ssh的安全拷贝命令(security copy)，它是从古老的远程复制命令rcp改变而来，实现的是在主机与主机之间的拷贝，可以是本地到远程的、本地到本地的，甚至可以远程到远程复制。注意，scp可能会询问密码。

如果scp拷贝的源文件在目标位置上已经存在时(文件同名)，scp会替换已存在目标文件中的内容，但保持其inode号，相当于先清空已存在的目标文件，然后粘贴拷贝的文件内容到目标文件。

如果scp拷贝的源文件在目标位置上不存在，则会在目标位置上创建一个空文件，然后将源文件中的内容填充进去。

之所以解释上面的两句，是为了理解scp的机制，scp拷贝本质是只是填充内容的过程，它不会去修改目标文件的很多属性，对于从远程复制到另一远程时，其机制见后文。
```
scp [-12BCpqrv] [-l limit] [-o ssh_option] [-P port] [[user@]host1:]file1 ... [[user@]host2:]file2

选项说明：
-1：使用ssh v1版本，这是默认使用协议版本
-2：使用ssh v2版本
-C：拷贝时先压缩，节省带宽
-l limit：限制拷贝速度，Kbit/s.
-o ssh_option：指定ssh连接时的特殊选项，一般用不上。偶尔在连接过程中等待
               提示输入密码较慢时，可以设置GSSAPIAuthentication为no
-P port：指定目标主机上ssh端口，大写的字母P，默认是22端口
-p：拷贝时保持源文件的 mtime,atime,owner,group,privileges
-r：递归拷贝，用于拷贝目录。注意，scp拷贝遇到链接文件时，会拷贝链接的源文件
    内容填充到目标文件中 (scp的本质就是填充而非拷贝)
-v：输出详细信息，可以用来调试或查看scp的详细过程，分析scp的机制
```

示例：

1. 把本地文件/home/a.tar.tz拷贝到远程服务器192.168.0.2上的/home/tmp，连接时使用远程的root用户

```bash
scp /home/a.tar.tz root@192.168.0.2:/home/tmp/
```

2. 目标主机不写路径时，表示拷贝到对方的家目录下

```
scp /home/a.tar.tz root@192.168.0.2
```

3. 把远程文件/home/a.tar.gz拷贝到本机

```
# 不接本地目录表示拷贝到当前目录
scp root@192.168.0.2:/home/a.tar.tz 

# 拷贝到本地/tmp目录下
scp root@192.168.0.2:/home/a.tar.tz /tmp
```

4. 拷贝远程机器的/home/目录到本地/tmp目录下

```
scp -r root@192.168.0.2:/home/ /tmp
```

5. 从远程主机192.168.100.60拷贝文件到另一台远程主机192.168.100.62上

```
scp root@192.168.100.60:/tmp/copy.txt root@192.168.100.62:/tmp
```

在远程复制到远程的过程中，例如在本地执行scp命令将A主机(192.168.100.60)上的/tmp/copy.txt复制到B主机(192.168.100.62)上的 /tmp 目录下，如果使用-v选项查看调试信息的话，会发现它的步骤类似是这样的。
```
# 以下是从结果中提取的过程

# 首先输出本地要执行的命令
Executing: /usr/bin/ssh -v -x -oClearAllForwardings yes -t -l root 192.168.100.60 scp -v /tmp/copy.txt root@192.168.100.62:/tmp

# 从本地连接到 A 主机
debug1: Connecting to 192.168.100.60 [192.168.100.60] port 22.
debug1: Connection established. 
# 要求验证本地和 A 主机之间的连接
debug1: Next authentication method: password
root@192.168.100.60's password:
# 将 scp 命令行修改后发送到 A 主机上
debug1: Sending command: scp -v /tmp/copy.txt root@192.168.100.62:/tmp
# 在 A 主机上执行 scp 命令
Executing: program /usr/bin/ssh host 192.168.100.62, user root, command scp -v -t /tmp
# 验证 A 主机和 B 主机之间的连接
debug1: Next authentication method: password
root@192.168.100.62's password:
# 从 A 主机上拷贝源文件到最终的 B 主机上
debug1: Sending command: scp -v -t /tmp
Sending file modes: C0770 24 copy.txt
Sink: C0770 24 copy.txt
copy.txt 100% 24 0.0KB/s 
# 关闭本地主机和 A 主机的连接
Connection to 192.168.100.60 closed.
```

也就是说，远程主机A到远程主机B的复制，实际上是将scp命令行从本地传递到主机A上，由A自己去执行scp命令。因此，本地主机不会和主机B有任何交互行为，本地主机就像是一个代理执行者一样，只是帮助传送scp命令行以及帮助显示信息。

其实从本地主机和主机A上的`~/.ssh/know_hosts`文件中可以看出，本地主机只是添加了主机A的信息，并没有添加主机B的信息，而在主机A上则添加了主机B的信息。

![](/img/linux/733013-20170612214810368-325831247.png)

### mv命令

mv命令移动文件和目录，还可以用于重命名文件或目录。

```
mv [-iuf] src dest # 移动单个文件或目录
mv [-iuf] src1 src2 src3 dest_dir # 移动多个文件或目录

选项说明：
--backup[=CONTROL]：如果目标文件已存在，则对该文件做一个备份，默认备份文件是在文件名后加上波浪线，如/b.txt~
-b：类似于--backup，但不接受参数, 默认备份文件是在文件名后加上波浪线，如/b.txt~
-f：如果目标文件已存在，则强制覆盖文件
-i：如果目标文件已存在，则提示是否要覆盖，这是alias mv的默认选项
-n：如果目标文件已存在，则不覆盖已存在的文件
           如果同时指定了-f/-i/-n，则后指定的生效
-u：(update)如果源文件和目标文件不同，则移动，否则不移动
```

mv默认已经是递归移动, 不需要-r参数。

<a name="mv_problem"></a>

### mv的一个经典问题(mv的本质)

该问题涉及文件系统操作文件的机制，若不理解，请先深入学习[文件系统](https://www.junmajinlong.com/linux/ext_filesystem)，或者先跳过。

mv不能实现里层同名目录覆盖外层同名目录。如/tmp下有a目录，a目录里还有a目录，将不能实现/tmp/a/a移动到/tmp。

```bash
[root@toystory tmp]# tree -L 3 a -fC
a
└── a/a
├── a/a/a
2 directories, 1 file

[root@toystory tmp]# mv a/* .
mv: overwrite `./a'? y
mv: cannot move `a/a' to `./a': Directory not empty

[root@toystory tmp]# mv -f /tmp/a/* /tmp
mv: cannot move `/tmp/a/a' to `/tmp/a': Directory not empty
```

要解释为何会如此，先说明移动和覆盖动作的本质。

同文件系统下移动文件实际上是修改目标文件所在目录的data block，向其中添加一行指向inode table中待移动文件的inode的指针，如果目标路径下有同名文件，则会提示是否覆盖，实际上是覆盖指向该同名文件的inode指针，由于同名文件的inode 记录指针被覆盖，就无法再找到该文件的data block，所以该文件被标记为删除。

跨文件系统移动文件的本质：如果目标路径下没有同名文件，则先为此文件分配一个inode号，并在目标目录的data block中添加一条指向该inode号的新记录 (是全新的)，然后将文件复制到目标位置，复制成功则删除源文件，复制失败则保留源文件；如果目标路径下有同名文件，则提示是否要覆盖，如果选择覆盖，则将该同名文件的inode指针指向新分配的inode号，然后将文件复制到目标位置，复制成功则删除源文件，复制失败则保留源文件。

![](/img/linux/733013-20170619194726257-205385234.png)

也就是说，同文件系统下移动文件时，inode记录不变(如inode号)，当然，时间戳是一定会改变的，因为移动过程中修改了inode指向data block的指针。而跨文件系统下移动文件时，inode记录完全改变，它是新添加的记录。

再考虑上面的问题，同文件系统下移动文件时先在目标位置/tmp的data block中添加一条记录，如果同名则提示覆盖，覆盖时会先删除/tmp的data block中的a对应的记录，再添加将要移动文件的记录。从上面的结果也可以看出是先提示覆盖再提示目录非空的错误。

设想下，如果/tmp/a/a移动到/tmp下并重命名为b，则其动作是直接向/tmp的data block中添加b的记录，如果此时正好/tmp下已有b目录，则先删除/tmp的data block中b目录对应的记录，再添加移动后的b记录。

但是现在不是重命名为b，而是覆盖/tmp/a，此时的动作按原理应该是先提示是否覆盖，如果是，则删除/tmp的data block中a对应的记录，但由于此时/tmp/a目录中还有文件，该记录无法删除 (因为如果要删除了该记录，代表删除了 /tmp/a整个目录，而删除整个/tmp/a目录需要删除里面所有的文件，在删除它们之前的一个动作是把/tmp/a中的所有目录和文件的inode号标记为未使用，但此刻要移动的源目录/tmp/a/a是在使用当中的)，所以提示目录非空而无法删除，这里所指的非空目录指的是/tmp/a，而非是/tmp/a/a非空。

但是在Windows操作系统下，里层目录是可以直接覆盖外层同名目录的，这和文件系统的行为有关。

其实在这个问题中，可以看出mv的很多原理。

## 查看文件内容

### cat命令

输出一个或多个文件的内容。
```
cat [OPTION]... [FILE]...

选项说明
-n：显示所有行的行号
-b：显示非空行的行号
-E：在每行行尾加上 $ 符号
-T：将 TAB 符号输出为 "^I"
-s：压缩连续空行为单个空行
```
通常cat还会用于将分行输入的内容写入到一个文件中去(这属于bash重定向的内容)。

首先测试`<<eof`，这表示将键入的内容追加到标准输入stdin中(不是从标准输入中读取)，eof可以随便使用其他符号代替。

```bash
[root@xuexi tmp]# cat <<eof
> abc.com
> eof
abc.com
```

再测试`< eof`，发现没有输入的机会，并且此时只能使用eof作为符号，EOF或其他任何都不可以。因为`<eof`是读取标准输入，会将eof当成输入文件处理。所以一定要使用`<<eof`，这表示here document，而前后两个eof正是document的起始和结束标志。

```bash
[root@xuexi tmp]# cat <eof
-bash: eof: No such file or directory

[root@xuexi tmp]# cat <EOF
-bash: EOF: No such file or directory
```

再进一步测试`<<eof`的功能，将键入的内容重定向到文件而非标准输入中。这时有两种书写方案(其实还有其他写法，暂时熟悉这两种常见写法即可)：

第一种方案：`>>filename<<eof`或`>filename<<eof`

```bash
[root@xuexi ~]# cat >>/tmp/test.txt<<EOF # 输入到这里按回车键继续输入下一行
> xxxxxxxxxxxxxx # 按回车输入下一行
> yyyyy # 按回车输入下一行
> zz # 按回车输入下一行
> EOF # 顶格写EOF结束输入
```

第二种方案：`<<eof>filename`或`<<eof>>filename`

```bash
[root@xuexi tmp]# cat <<eof>log.txt
> abc.com
> eof
```

两种方案结果是一样的，且总是使用`<<eof`，只不过所写的位置不同而已，不管写在哪个位置，它都表示将键入的内容追加到标准输入。然后再使用`>filename`或`>>filename`控制重定向的方式，将标准输入中的内容重定向到filename文件中。

### tac

tac和cat字母正好是相反的，其作用也是和cat相反的，它会反向输出行，将最后一行放在第一行的位置输出，依此类推。但是，tac没有显示行号的参数。

```bash
shell> echo -e '1\n2\n3\n4\n5' | tac
5
4
3
2
1
```

### head

head打印前面的几行。
```
head [-n num] | [-num] [-v] filename

-n：显示前num行；如果num是负数，则显示除了最后|num|(绝对值)行
    的其余所有行，即显示前"总行数-|num|"
-v：会显示出文件名
```

`-n num`是显示文件的前num行，num可以是`+/-`或不加正负号的整数，如果是正整数或不写`+`号，则显示前num行。如果是负整数，则从后向前数num行，并打印除了这些行的前面所有的行，即打印除了最后num行的所有行，也即总行数减num的前正数行。不写-n时默认是前10行。在有些版本的head命令中，正整数时`-n num`可以直接简写为`-num`。

不管怎么样，它取的都是前几行，哪怕是负整数也是前几行。

示例：

```bash
# 取出默认前10行，但总共才有5行
[root@xuexi ~]# echo -e '1\n2\n3\n4\n5' | head
1
2
3
4
5

# 取出前2行
[root@xuexi ~]# echo -e '1\n2\n3\n4\n5' | head -2
1
2
```
或者
```bash
# 取出前2行
[root@xuexi ~]# echo -e '1\n2\n3\n4\n5' | head -n 2
1
2

# 取出前5-1=4行
[root@xuexi ~]# echo -e '1\n2\n3\n4\n5' | head -n -1
1
2
3
4
```

### tail

tail和head相反，是显示后面的行，默认是后10行。
```
tail [OPTION]... [FILE]...

选项说明：
-n：输出最后num行，如果使用-n +num则表示输出从第num行开始的所有行
-f：监控文件变化
--pid=PID：和-f一起使用，在给定PID的进程死亡后，终止文件监控
-v：显示文件名
```

`-n -num`或`-num`或`-n num`(num为正整数)表示输出最后的num行。使用`-n +num`(num为正整数) 则表示输出从第num行开始的所有行。

```bash
# 等价于 tail -n 3和tail -n -3
[root@xuexi ~]# echo -e '1\n2\n3\n4\n5' | tail -3
3
4
5

# 打印除了前3-1=2行的所有行
[root@xuexi tmp]# seq 6 | tail -n +3
3
4
5
6
```

tail还有一个重要的参数-f，监控文件的内容变化。当一个用户不断修改某个文件的尾部，另一个用户就可以通过这个命令来刷新并显示这些修改后的内容。

### nl

以行号的方式查看内容。

常用`-b a`，表示不论是否空行都显示行号，等价于`cat -n`。不写选项时，默认`-b t`，表示空行不显示行号，等价于`cat -b`。

```bash
# 默认空行不显示行号
[root@xuexi ~]# nl /etc/issue
1 CentOS release 6.6 (Final)
2 Kernel \r on an \m

[root@xuexi ~]# nl -b a /etc/issue
1 CentOS release 6.6 (Final)
2 Kernel \r on an \m
3
```

### more和less

按页显示文件内容。使用more时，使用`/`搜索字符串，按下n或N键表示向下或向上继续搜索。

使用less时，还多了一个搜索功能，使用`?`搜索字符串，同样，使用n或N键可以向上或向下继续搜索。

### 比较文件内容

```bash
shell> diff file1 file2
shell> vimdiff file1 file2
```

## 文件查找类命令

搜索文件的路径在何处以及文件的名称为何。

### which

显示命令或脚本的全路径，默认也会将命令的别名显示出来。

```bash
shell> which mv
alias mv='mv -i'
       /bin/mv
```

### whereis

找出二进制文件、源文件和man文档文件。
```bash
shell> whereis cd

cd: /usr/bin/cd /usr/share/man/man1/cd.1.gz /usr/share/man/man1p/cd.1p.gz /usr/share/man/mann/cd.n.gz
```

### whatis

列出给定命令(并非一定是命令)的man文档信息。

```bash
shell> whatis passwd

sslpasswd (1ssl) - compute password hashes
passwd (1) - update user's authentication tokens
passwd (5) - password file
```

根据上面的结果，执行：

```
man 1 passwd # 获取passwd命令的man文档
man 5 passwd # 获取password文件的man文档，文件类的man文档说明的是该文件中各配置项意义
man sslpasswd # 获取sslpasswd命令的man文档，实际上是openssl passwd的man文档
```

### locate

没什么好说的。

### find

内容太多，使用两篇单独的文章解释。

- [Linux find常用用法示例](https://www.junmajinlong.com/shell/find_usage)  
- [Linux find运行机制详解](https://www.junmajinlong.com/shell/find_intermediate)
