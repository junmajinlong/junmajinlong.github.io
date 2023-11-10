---
title: grub2详解(翻译和整理官方手册)
p: linux/grub2.md
date: 2020-07-16 18:20:30
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# grub2详解(翻译和整理官方手册)

本文翻译了grub2[官方手册](https://www.gnu.org/software/grub/manual/html_node/index.html#SEC_Contents)的绝大部分内容，然后自己整理了一下。因为内容有点杂，所以章节安排上可能不是太合理，敬请谅解。

本文主要介绍的是grub2，在文末对传统grub进行了简述，但在grub2的内容部分中包含了很多grub2和传统grub的对比。

如果仅仅是想知道grub2中的boot.img/core.img/diskboot.img/kernel.img或者传统grub中stage1/stage1\_5/stage2文件的作用，请直接跳至[相关内容处](#blog122)阅读。

## 1.1 基础内容

### 1.1.1 grub2和grub的区别 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Changes-from-GRUB-Legacy.html#Changes-from-GRUB-Legacy>

只说明几个主要的区别：

1.配置文件的名称改变了。在grub中，配置文件为grub.conf或menu.lst(grub.conf的一个软链接)，在grub2中改名为grub.cfg。

2.grub2增添了许多语法，更接近于脚本语言了，例如支持变量、条件判断、循环。

3.grub2中，设备分区名称从1开始，而在grub中是从0开始的。

4.grub2使用img文件，不再使用grub中的stage1、stage1.5和stage2。

5.支持图形界面配置grub，但要安装grub-customizer包，epel源提供该包。

6.在已进入操作系统环境下，不再提供grub命令，也就是不能进入grub交互式界面，只有在开机时才能进入，算是一大缺憾。

7.在grub2中没有了好用的find命令，算是另一大缺憾。

### 1.1.2 命名习惯和文件路径表示方式 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Naming-convention.html#Naming-convention>。

```
(fd0)           ：表示第一块软盘
(hd0,msdos2)    ：表示第一块硬盘的第二个mbr分区。grub2中分区从1开始编号，传统的grub是从0开始编号的
(hd0,msdos5)    ：表示第一块硬盘的第一个逻辑分区
(hd0,gpt1)      ：表示第一块硬盘的第一个gpt分区
/boot/vmlinuz   ：相对路径，基于根目录，表示根目录下的boot目录下的vmlinuz，
                ：如果设置了根目录变量root为(hd0,msdos1)，则表示(hd0,msdos1)/boot/vmlinuz
(hd0,msdos1)/boot/vmlinuz：绝对路径，表示第一硬盘第一分区的boot目录下的vmlinuz文件
```

### 1.1.3 grub2引导操作系统的方式 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/General-boot-methods.html#General-boot-methods>。

grub2支持两种方式引导操作系统：

 *  直接引导：(direct-load)直接通过默认的grub2 boot loader来引导写在默认配置文件中的操作系统
 *  链式引导：(chain-load)使用默认grub2 boot loader链式引导另一个boot loader，该boot loader将引导对应的操作系统

一般只使用第一种方式，只有想引导grub默认不支持的操作系统时才会使用第二种方式。

### 1.1.4 grub2程序和传统grub程序安装后的文件分布 

在传统grub软件安装完后，在/usr/share/grub/RELEASE/目录下会生成一些stage文件。

```
[root@xuexi ~]# ls /usr/share/grub/x86_64-redhat/
e2fs_stage1_5  ffs_stage1_5  jfs_stage1_5  reiserfs_stage1_5
stage2 ufs2_stage1_5  xfs_stage1_5  fat_stage1_5  
iso9660_stage1_5  minix_stage1_5 stage1  stage2_eltorito  vstafs_stage1_5
```

在grub2软件安装完后，会在/usr/lib/grub/i386-pc/目录下生成很多模块文件和img文件，还包括一些lst列表文件。

```
[root@server7 ~]# ls /usr/lib/grub/i386-pc/*.mod | wc -l
257

[root@server7 ~]# ls -lh /usr/lib/grub/i386-pc/*.lst   
-rw-r--r--. 1 root root 3.7K Nov 24  2015 /usr/lib/grub/i386-pc/command.lst
-rw-r--r--. 1 root root  936 Nov 24  2015 /usr/lib/grub/i386-pc/crypto.lst
-rw-r--r--. 1 root root  214 Nov 24  2015 /usr/lib/grub/i386-pc/fs.lst
-rw-r--r--. 1 root root 5.1K Nov 24  2015 /usr/lib/grub/i386-pc/moddep.lst
-rw-r--r--. 1 root root  111 Nov 24  2015 /usr/lib/grub/i386-pc/partmap.lst
-rw-r--r--. 1 root root   17 Nov 24  2015 /usr/lib/grub/i386-pc/parttool.lst
-rw-r--r--. 1 root root  202 Nov 24  2015 /usr/lib/grub/i386-pc/terminal.lst
-rw-r--r--. 1 root root   33 Nov 24  2015 /usr/lib/grub/i386-pc/video.lst

[root@server7 ~]# ls -lh /usr/lib/grub/i386-pc/*.img
-rw-r--r--. 1 root root  512 Nov 24  2015 /usr/lib/grub/i386-pc/boot_hybrid.img
-rw-r--r--. 1 root root  512 Nov 24  2015 /usr/lib/grub/i386-pc/boot.img
-rw-r--r--. 1 root root 2.0K Nov 24  2015 /usr/lib/grub/i386-pc/cdboot.img
-rw-r--r--. 1 root root  512 Nov 24  2015 /usr/lib/grub/i386-pc/diskboot.img
-rw-r--r--. 1 root root  28K Nov 24  2015 /usr/lib/grub/i386-pc/kernel.img
-rw-r--r--. 1 root root 1.0K Nov 24  2015 /usr/lib/grub/i386-pc/lnxboot.img
-rw-r--r--. 1 root root 2.9K Nov 24  2015 /usr/lib/grub/i386-pc/lzma_decompress.img
-rw-r--r--. 1 root root 1.0K Nov 24  2015 /usr/lib/grub/i386-pc/pxeboot.img
```

### 1.1.5 boot loader和grub的关系 

当使用grub来管理启动菜单时，那么boot loader都是grub程序安装的。

传统的grub将stage1转换后的内容安装到MBR(VBR或EBR)中的boot loader部分，将stage1\_5转换后的内容安装在紧跟在MBR后的扇区中，将stage2转换后的内容安装在/boot分区中。

grub2将boot.img转换后的内容安装到MBR(VBR或EBR)中的boot loader部分，将diskboot.img和kernel.img结合成为core.img，同时还会嵌入一些模块或加载模块的代码到core.img中，然后将core.img转换后的内容安装到磁盘的指定位置处。

它们之间更具体的关系见下文。

### 1.1.6 grub2的安装位置 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/BIOS-installation.html#BIOS-installation>。

严格地说是core.img的安装位置，因为boot.img的位置是固定在MBR或VBR或EBR上的。

#### (1).MBR

MBR格式的分区表用于PC BIOS平台，这种格式允许四个主分区和额外的逻辑分区。使用这种格式的分区表，有两种方式安装GURB：

1.  嵌入到MBR和第一个分区中间的空间，这部分就是大众所称的"boot track","MBR gap"或"embedding area"，它们大致需要31kB的空间；
2.  将core.img安装到某个文件系统中，然后使用分区的第一个扇区(严格地说不是第一个扇区，而是第一个block)存储启动它的代码。

这两种方法有不同的问题。

使用嵌入的方式安装grub，就没有保留的空闲空间来保证安全性，例如有些专门的软件就是使用这段空间来实现许可限制的；另外分区的时候，虽然会在MBR和第一个分区中间留下空闲空间，但可能留下的空间会比这更小。

方法二安装grub到文件系统，但这样的grub是脆弱的。例如，文件系统的某些特性需要做尾部包装，甚至某些fsck检测，它们可能会移动这些block。

GRUB开发团队建议将GRUB嵌入到MBR和第一个分区之间，除非有特殊需求，但仍必须要保证第一个分区至少是从第31kB(第63个扇区)之后才开始创建的。

现在的磁盘设备，一般都会有分区边界对齐的性能优化提醒，所以第一个分区可能会自动从第1MB处开始创建。

#### (2).GPT

一些新的系统使用GUID分区表(GPT)格式，这种格式是EFI固件所指定的一部分。但如果操作系统支持的话，GPT也可以用于BIOS平台(即MBR风格结合GPT格式的磁盘)，使用这种格式，需要使用独立的BIOS boot分区来保存GRUB，GRUB被嵌入到此分区，不会有任何风险。

当在gpt磁盘上创建一个BIOS boot分区时，需要保证两件事：(1)它最小是31kB大小，但一般都会为此分区划分1MB的空间用于可扩展性；(2)必须要有合理的分区类型标识(flag type)。

例如使用gun parted工具时，可以设置为bios\_grub标识：

```
# parted /dev/sda toggle partition_num bios_grub
# parted /dev/sda set partiton_num bios_grub on
```

如果使用gdisk分区工具时，则分类类型设置为"EF02"。

如果使用其他的分区工具，可能需要指定guid，则可以指定其guid为"21686148-6449-6e6f-744e656564454649"。

下图是某个bios/gpt格式的bios boot分区信息，从中可见，它大小为1M，没有文件系统，分区表示为bios\_grub。

![](/img/linux/1699314170372.png)

下图为gpt磁盘在图形界面下安装操作系统时创建的Bios boot分区。

![](/img/linux/1699314182644.png)

### 1.1.7 进入grub命令行 

在传统的grub上，可以直接在bash下敲入grub命令进入命令交互模式，但grub2只能在系统启动前进入grub交互命令行。

按下e见可以编辑所选菜单对应的grub菜单配置项，按下c键可以进入grub命令行交互模式。

![](/img/linux/1699314241837.png)

## 1.2 安装grub2 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Installing-GRUB-using-grub_002dinstall.html#Installing-GRUB-using-grub_002dinstall>。

这里的安装指的不是安装grub程序，而是安装Boot loader，但一般都称之为安装grub，且后文都是这个意思。

### 1.2.1 grub安装命令 

安装方式非常简单，只需调用grub2-install，然后给定安装到的设备名即可。

```
shell> grub2-install /dev/sda
```

这样的安装方式，默认会将img文件放入到/boot目录下，如果想自定义放置位置，则使用`--boot-directory`选项指定，可用于测试练习grub的时候使用，但在真实的grub环境下不建议做任何改动。

```
shell> grub2-install --boot-director=/mnt/boot /dev/fd0
```

如果是EFI固件平台，则必须挂载好efi系统分区，一般会挂在/boot/efi下，这是默认的，此时可直接使用grub2-install安装。

```
shell> grub2-install
```

如果不是挂载在/boot/efi下，则使用`--efi-directory`指定efi系统分区路径。

```
shell> grub2-install --efi-directory=/mnt/efi
```

grub2-install实际上是一个shell脚本，用于调用其他工具，真正的功能都是其他工具去完成的，所以如果非常熟悉grub内部命令和机制，完全可以不用grub2-install。

对应传统的grub安装命令为grub-install，用法和grub2-install一样。

<a name="blog122"></a>

### 1.2.2 各种img和stage文件的说明 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Images.html#Images>。

img文件是grub2生成的，stage文件是传统grub生成的。下面是各种文件的说明。

#### grub2中的img文件 

grub2生成了好几个img文件，有些分布在/usr/lib/grub/i386-pc目录下，有些分布在/boot/grub2/i386-pc目录下，它们之间的关系，相信看了下文之后就会明白了。

![](/img/linux/1699314534252.png)

下图描述了各个img文件之间的关系。其中core.img是动态生成的，路径为/boot/grub2/i386-pc/core.img，而其他的img则存在于/usr/lib/grub/i386-pc目录下。当然，在安装grub时，boot.img会被拷贝到/boot/grub2/i386-pc目录下。

![](/img/linux/1699314553245.png)

(1)boot.img

在BIOS平台下，boot.img是grub启动的第一个img文件，它被写入到MBR中或分区的boot sector中，因为boot sector的大小是512字节，所以该img文件的大小也是512字节。

boot.img唯一的作用是读取属于core.img的第一个扇区并跳转到它身上，将控制权交给该扇区的img。由于体积大小的限制，boot.img无法理解文件系统的结构，因此grub2-install将会把core.img的位置硬编码到boot.img中，这样就一定能找到core.img的位置。

(2)core.img

core.img根据diskboot.img、kernel.img和一系列的模块被grub2-mkimage程序动态创建。core.img中嵌入了足够多的功能模块以保证grub能访问/boot/grub，并且可以加载相关的模块实现相关的功能，例如加载启动菜单、加载目标操作系统的信息等，由于grub2大量使用了动态功能模块，使得core.img体积变得足够小。

core.img中包含了多个img文件的内容，包括diskboot.img/kernel.img等。

core.img的安装位置随MBR磁盘和GPT磁盘而不同，这在上文中已经说明过了。

(3)diskboot.img

如果启动设备是硬盘，即从硬盘启动时，core.img中的第一个扇区的内容就是diskboot.img。diskboo.img的作用是读取core.img中剩余的部分到内存中，并将控制权交给kernel.img，由于此时还不识别文件系统，所以将core.img的全部位置以block列表的方式编码，使得diskboot.img能够找到剩余的内容。

该img文件因为占用一个扇区，所以体积为512字节。

(4)cdboot.img

如果启动设备是光驱(cd-rom)，即从光驱启动时，core.img中的第一个扇区的的内容就是cdboo.img。它的作用和diskboot.img是一样的。

(5)pexboot.img

如果是从网络的PXE环境启动，core.img中的第一个扇区的内容就是pxeboot.img。

(6)kernel.img

kernel.img文件包含了grub的基本运行时环境：设备框架、文件句柄、环境变量、救援模式下的命令行解析器等等。很少直接使用它，因为它们已经整个嵌入到了core.img中了。注意，kernel.img是grub的kernel，和操作系统的内核无关。

如果细心的话，会发现kernel.img本身就占用28KB空间，但嵌入到了core.img中后，core.img文件才只有26KB大小。这是因为core.img中的kernel.img是被压缩过的。

(7)lnxboot.img

该img文件放在core.img的最前部位，使得grub像是linux的内核一样，这样core.img就可以被LILO的`image=`识别。当然，这是配合LILO来使用的，但现在谁还适用LILO呢？

(8)\*.mod

各种功能模块，部分模块已经嵌入到core.img中，或者会被grub自动加载，但有时也需要使用insmod命令手动加载。

#### 传统grub中的stage文件 

grub2的设计方式和传统grub大不相同，因此和stage之间的对比关系其实没那么标准，但是将它们拿来比较也有助于理解img和stage文件的作用。

stage文件也分布在两个地方：/usr/share/grub/RELEASE目录下和/boot/grub目录下，/boot/grub目录下的stage文件是安装grub时从/usr/share/grub/RELEASE目录下拷贝过来的。

![](/img/linux/1699314666374.png)

(1)stage1

stage1文件在功能上等价于boot.img文件。目的是跳转到stage1\_5或stage2的第一个扇区上。

(2)\*\_stage1\_5

\*stage1\_5文件包含了各种识别文件系统的代码，使得grub可以从文件系统中读取体积更大功能更复杂的stage2文件。从这一方面考虑，它类似于core.img中加载对应文件系统模块的代码部分，但是core.img的功能远比stage1\_5多。

stage1\_5一般安装在MBR后、第一个分区前的那段空闲空间中，也就是MBR gap空间，它的作用是跳转到stage2的第一个扇区。

其实传统的grub在某些环境下是可以不用stage1\_5文件就能正常运行的，但是grub2则不能缺少core.img。

(3)stage2

stage2的作用是加载各种环境和加载内核，在grub2中没有完全与之相对应的img文件，但是core.img中包含了stage2的所有功能。

当跳转到stage2的第一个扇区后，该扇区的代码负责加载stage2剩余的内容。

注意，stage2是存放在磁盘上的，并没有像core.img一样嵌入到磁盘上。

(4)stage2\_eltorito

功能上等价于grub2中的core.img中的cdboot.img部分。一般在制作救援模式的grub时才会使用到cd-rom相关文件。

(5)pxegrub

功能上等价于grub2中的core.img中的pxeboot.img部分。

### 1.2.3 安装grub涉及的过程 

安装grub2的过程大体分两步：一是根据/usr/lib/grub/i386-pc/目录下的文件生成core.img，并拷贝boot.img和core.img涉及的某些模块文件到/boot/grub2/i386-pc/目录下；二是根据/boot/grub2/i386-pc目录下的文件向磁盘上写boot loader。

当然，到底是先拷贝，还是先写boot loader，没必要去搞清楚，只要/boot/grub2/i386-pc下的img文件一定能通过grub2相关程序再次生成boot loader。所以，既可以认为/boot/grub2/i386-pc目录下的img文件是boot loader的特殊备份文件，也可以认为是boot loader的源文件。

不过，img文件和boot loader的内容是不一致的，因为img文件还要通过grub2相关程序来转换才是真正的boot loader。

对于传统的grub而言，拷贝的不是img文件，而是stage文件。

以下是安装传统grub时，grub做的工作。很不幸，grub2上没有该命令，也没有与之等价的命令。

```
grub> setup (hd0)
 Checking if "/boot/grub/stage1" exists... yes
 Checking if "/boot/grub/stage2" exists... yes
 Checking if "/boot/grub/e2fs_stage1_5" exists... yes
 Running "embed /boot/grub/e2fs_stage1_5 (hd0)"...  15 sectors are embedded.
succeeded
 Running "install /boot/grub/stage1 (hd0) (hd0)1+15 p (hd0,0)/boot/grub/stage2 /boot/grub/menu.lst"... succeeded
Done.
```

首先检测各stage文件是否存在于/boot/grub目录下，随后嵌入stage1\_5到磁盘上，该文件系统类型的stage1\_5占用了15个扇区，最后安装stage1，并告知stage1 stage1\_5的位置是第1到第15个扇区，之所以先嵌入stage1\_5再嵌入stage1就是为了让stage1知道stage1\_5的位置，最后还告知了stage1 stage2和配置文件menu.lst的路径。

## 1.3 grub2配置文件 

grub2的默认配置文件为/boot/grub2/grub.cfg，该配置文件的写法弹性非常大，但绝大多数需要修改该配置文件时，都只需修改其中一小部分内容就可以达成目标。

grub2-mkconfig程序可用来生成符合绝大多数情况的grub.cfg文件，默认它会自动尝试探测有效的操作系统内核，并生成对应的操作系统菜单项。使用方法非常简单，只需一个选项"-o"指定输出文件即可。

```
shell> grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 1.3.1 通过/etc/default/grub文件生成grub.cfg 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Simple-configuration.html#Simple-configuration>

grub2-mkconfig是根据/etc/default/grub文件来创建配置文件的。该文件中定义的是grub的全局宏，修改内置的宏可以快速生成grub配置文件。实际上在/etc/grub.d/目录下还有一些grub配置脚本，这些shell脚本读取一些脚本配置文件(如/etc/default/grub)，根据指定的逻辑生成grub配置文件。若有兴趣，不放读一读`/etc/grub.d/10_linux`文件，它指导了创建grub.cfg的细节，例如如何生成启动菜单。

```
[root@xuexi ~]# ls /etc/grub.d/
00_header  00_tuned  01_users  10_linux  20_linux_xen  20_ppc_terminfo  30_os-prober  40_custom  41_custom  README
```

在/etc/default/grub中，使用`key=vaule`的格式，key全部为大小字母，如果vaule部分包含了空格或其他特殊字符，则需要使用引号包围。

例如，下面是一个/etc/default/grub文件的示例：

```
[root@xuexi ~]# cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto biosdevname=0 net.ifnames=0 rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

虽然可用的宏较多，但可能用的上的就几个：`GRUB_DEFAULT、GRUB_TIMEOUT、GRUB_CMDLINE_LINUX和GRUB_CMDLINE_LINUX_DEFAULT`。

以下列出了部分key。

(1).GRUB\_DEFAULT

默认的菜单项，默认值为0。其值可为数值N，表示从0开始计算的第N项是默认菜单，也可以指定对应的title表示该项为默认的菜单项。使用数值比较好，因为使用的title可能包含了容易改变的设备名。例如有如下菜单项：

```
menuentry 'Example GNU/Linux distribution' --class gnu-linux --id example-gnu-linux {
    ...
}
```

如果想将此菜单设为默认菜单，则可设置`GRUB_DEFAULT=example-gnu-linux`。

如果`GRUB_DEFAULT`的值设置为"saved"，则表示默认的菜单项是`GRUB_SAVEDEFAULT`或"grub-set-default"所指定的菜单项。

(2).GRUB\_SAVEDEFAULT

默认该key的值未设置。如果该key的值设置为true时，如果选定了某菜单项，则该菜单项将被认为是新的默认菜单项。该key只有在设置了`GRUB_DEFAULT=saved`时才有效。

不建议使用该key，因为`GRUB_DEFAULT`配合grub-set-default更方便。

(3).GRUB\_TIMEOUT

在开机选择菜单项的超时时间，超过该时间将使用默认的菜单项来引导对应的操作系统。默认值为5秒。等待过程中，按下任意按键都可以中断等待。

设置为0时，将不列出菜单直接使用默认的菜单项引导与之对应的操作系统，设置为"-1"时将永久等待选择。

是否显示菜单，和`GRUB_TIMEOUT_STYLE`的设置有关。

(4).GRUB\_TIMEOUT\_STYLE

如果该key未设置值或者设置的值为"menu"，则列出启动菜单项，并等待`GRUB_TIMEOUT`指定的超时时间。

如果设置为"countdown"和"hidden"，则不显示启动菜单项，而是直接等待`GRUB_TIMEOUT`指定的超时时间，如果超时了则启动默认菜单项并引导对应的操作系统。在等待过程中，按下"ESC"键可以列出启动菜单。设置为countdown和hidden的区别是countdown会显示超时时间的剩余时间，而hidden则完全隐藏超时时间。

(5).GRUB\_DISTRIBUTOR

设置发行版的标识名称，一般该名称用来作为菜单的一部分，以便区分不同的操作系统。

(6).GRUB\_CMDLINE\_LINUX

添加到菜单中的内核启动参数。例如：

```
GRUB_CMDLINE_LINUX="crashkernel=ro root=/dev/sda3 biosdevname=0 net.ifnames=0 rhgb quiet"
```

(7).GRUB\_CMDLINE\_LINUX\_DEFAULT

除非`GRUB_DISABLE_RECOVERY`设置为"true"，否则该key指定的默认内核启动参数将生成两份，一份是用于默认启动参数，一份用于恢复模式(recovery mode)的启动参数。

该key生成的默认内核启动参数将添加在`GRUB_CMDLINE_LINUX`所指定的启动参数之后。

(8).GRUB\_DISABLE\_RECOVERY

该项设置为true时，将不会生成恢复模式的菜单项。

(9).GRUB\_DISABLE\_LINUX\_UUID

默认情况下，grub2-mkconfig在生产菜单项的时候将使用uuid来标识Linux 内核的根文件系统，即`root=UUID=...`。

例如，下面是/boot/grub2/grub.cfg中某菜单项的部分内容。
```
menuentry 'CentOS Linux (3.10.0-327.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-327.el7.x86_64-advanced-b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8' {
  ......
  
  linux16 /vmlinuz-3.10.0-327.el7.x86_64root=UUID=b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8 ro crashkernel=auto biosdevname=0 net.ifnames=0 quiet LANG=en_US.UTF-8
  initrd16 /initramfs-3.10.0-327.el7.x86_64.img
}
```
虽然使用UUID的方式更可靠，但有时候不太方便，所以可以设置该key为true来禁用。

(10).GRUB\_BACKGROUND

设置背景图片，背景图片必须是grub可读的，图片文件名后缀必须是".png"、".tga"、".jpg"、".jpeg"，在需要的时候，grub会按比例缩小图片的大小以适配屏幕大小。

(11).GRUB\_THEME

设置grub菜单的主题。

(12).GRUB\_GFXPAYLOAD\_LINUX

设置为"text"时，将强制使用文本模式启动Linux。在某些情况下，可能不支持图形模式。

(13).GRUB\_DISABLE\_OS\_PROBER

默认情况下，grub2-mkconfig会尝试使用os-prober程序(如果已经安装的话，默认应该都装了)探测其他可用的操作系统内核，并为其生成对应的启动菜单项。设置为"true"将禁用自动探测功能。

(14).GRUB\_DISABLE\_SUBMENU

默认情况下，grub2-mkconfig如果发现有多个同版本的或低版本的内核时，将只为最高版本的内核生成顶级菜单，其他所有的低版本内核菜单都放入子菜单中，设置为"y"将全部生成为顶级菜单。

(15).GRUB\_HIDDEN\_TIMEOUT(已废弃，但为了向后兼容，仍有效)

使用`GRUB_TIMEOUT_STYLE={countdown|hidden}`替代该项

(16).GRUB\_HIDDEN\_TIMEOUT\_QUIET(已废弃，但为了向后兼容，仍有效)

配合`GRUB_HIDDEN_TIMEOUT`使用，可以使用`GRUB_TIMEOUT_STYLE=countdown`来替代这两项。

### 1.3.2 脚本方式直接编写grub.cfg文件 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Shell_002dlike-scripting.html#Shell_002dlike-scripting>

- 注释符：从`#`开始的字符都被认为是注释，所以grub支持行中注释
- 连接操作符：`{ } | & $ ; < >`
- 保留关键字和符号：`! [[ ]] { } case do done elif else esac fi for function if in menuentry select then time until while`。并非所有的关键字都有用，只是为了日后的功能扩展而提前提供的。
- 引号和转义符

  对于特殊的字符需要转义。有三种方式转义：使用反斜线、使用单引号、使用双引号。

  反斜线转义方式和shell一样。

  单引号中的所有字符串都是字面意思，没有任何特殊意义，即使单引号中的转义符也被认为是纯粹的字符。所以`'\''`是无法保留单引号的。单引号需要使用双引号来转移，所以应该写`"'"`。

  双引号和单引号作用一样，但它不能转义某几个特殊字符，包括`$`和`\`。对于双引号中的`$`符号，它任何时候都保留本意。对于`\`，只有反斜线后的字符是`$、'"'、'\'`时才表示转义的意思，另外 ，某行若以反斜线结尾，则表示续行，但官方不建议在grub.cfg中使用续行符。

 - 变量扩展

   使用`$`符号引用变量，也可以使用`${var}`的方式引用var变量。

   支持位置变量，例如$1引用的是第一个参数。

   还支持特殊的变量，如`$?`表示上一次命令的退出状态码。如果使用了位置变量，则还支持`$* $@ $#`，`$*`代表的所有参数整体，各参数之间是不可分割的，`$@`也代表所有变量，但`$@`的各参数是可以被分割的，`$#`表示参数的个数。

 - 简单的命令

   可以在grub.cfg中使用简单的命令。各命令之间使用换行符或分号表示该命令结束。

   如果在命令前使用了"!"，则表示逻辑取反。

 - 循环结构：for name in word …; do list; done
 - 循环结构：while cond; do list; done
 - 循环结构：until cond; do list; done
 - 条件判断结构：if list; then list; \[elif list; then list;\] … \[else list;\] fi
 - 函数结构：function name \{ command; … \}
 - 菜单项命令：menuentry title \[--class=class …\] \[--users=users\] \[--unrestricted\] \[--hotkey=key\] \[--id=id\] \{ command; … \}

这是grub.cfg中最重要的项，官方原文：<https://www.gnu.org/software/grub/manual/html_node/menuentry.html#menuentry>

该命令定义了一个名为title的grub菜单项。当开机时选中该菜单项时，grub会将chosen环境变量的值赋给`--id`(如果给定了`--id`的话)，执行大括号中的命令列表，如果直到最后一个命令都全部执行成功，且成功加载了对应的内核后，将执行boot命令。随后grub就将控制权交给了操作系统内核。
```
--class：该选项用于将菜单分组，从而使得grub可以通过主题样式为不同组的菜单显示不同的样式风格。一个menuentry中，可以使用多次class表示将该菜单分到多个组中去。
--users：该选项限定只有此处列出的用户才能访问该菜单项，不指定该选项时将表示所有用户都能访问该菜单。
--unrestricted：该选项表示所有用户都有权访问该菜单项。
--hotkey：该选项为该菜单项关联一个热键，也就是快捷键，关联热键后只要按下该键就会选中该菜单。热键只能是字母键、backspace键、tab键或del键。
--id：该选项为该菜单关联一个唯一的数值。id的值可以由ASCII字母、数字//下划线组成，且不得以数字开头。
```

所有其他的参数包括title都被当作位置参数传递给大括号中的命令，但title总是$1，除title外的其余参数，位置值从前向后类推。

```
break [n]：强制退出for/while/until循环
continue [n]：跳到下一次迭代，即进入下一次循环
return [n]：指定返回状态码
setparams [arg] …：从$1开始替换位置参数
shift [n]：踢掉前n个参数，使得第n+1个参数变为$1，但和shell中不一样的是，踢掉了前n个参数后，从$#-n+1到$#这些参数的位置不变
```

具体如何编写grub.cfg文件，继续看下文的命令和变量。

## 1.4 命令行和菜单项中的命令 

官方手册原文：<https://www.gnu.org/software/grub/manual/html_node/Commands.html#Commands>

grub2支持很多命令，有些命令只能在交互式命令行下使用，有些命令可用在配置文件中。在救援模式下，只有insmod、ls、set和unset命令可用。

无需掌握所有的命令，掌握用的上的几个命令即可。

### 1.4.1 help命令 

```
help [pattern]
```

显示能匹配到pattern的所有命令的说明信息和usage信息，如果不指定patttern，将显示所有命令的简短信息。

例如`help cmos`。

![](/img/linux/1699315067527.png)

### 1.4.2 boot命令 

用于启动已加载的操作系统。

只在交互式命令行下可用。其实在menuentry命令的结尾就隐含了boot命令。

### 1.4.3 set和unset命令 

```
set [envvar=value]
unset envvar
```

前者设置环境变量envvar的值，如果不给定参数，则列出当前环境变量。

后者释放环境变量envvar。

### 1.4.4 lsmod命令和insmod命令 

分别用于列出已加载的模块和调用指定的模块。

注意，若要导入支持ext文件系统的模块时，只需导入ext2.mod即可，实际上也没有ext3和ext4对应的模块。

### 1.4.5 linux和linux16命令 

```
linux file [kernel_args]
linux16 file [kernel_args]
```

都表示装载指定的内核文件，并传递内核启动参数。linux16表示以传统的16位启动协议启动内核，linux表示以32位启动协议启动内核，但linux命令比linux16有一些限制。但绝大多数时候，它们是可以通用的。

在linux或linux16命令之后，必须紧跟着使用init或init16命令装载init ramdisk文件。

一般为/boot分区下的`vmlinuz-RELEASE_NUM`文件。

![](/img/linux/1699315087423.png)

但在grub环境下，boot分区被当作root分区，即根分区，假如boot分区为第一块磁盘的第一个分区，则应该写成：
```
linux (hd0,msdos1)/vmlinuz-XXX
```
或者相对路径的：
```
set root='hd0,msdos1'

linux /vmlinuz-XXX
```
在grub阶段可以传递内核的启动参数(内核的参数包括3类：编译内核时参数，启动时参数和运行时参数)，可以传递的启动参数非常非常多，完整的启动参数列表见：<http://redsymbol.net/linux-kernel-boot-parameters>。这里只列出几个常用的：

```
init=   ：指定Linux启动的第一个进程init的替代程序。
root=   ：指定根文件系统所在分区，在grub中，该选项必须给定。
ro,rw   ：启动时，根分区以只读还是可读写方式挂载。不指定时默认为ro。
initrd  ：指定init ramdisk的路径。在grub中因为使用了initrd或initrd16命令，所以不需要指定该启动参数。
rhgb    ：以图形界面方式启动系统。
quiet   ：以文本方式启动系统，且禁止输出大多数的log message。
net.ifnames=0：用于CentOS 7，禁止网络设备使用一致性命名方式。
biosdevname=0：用于CentOS 7，也是禁止网络设备采用一致性命名方式。
             ：只有net.ifnames和biosdevname同时设置为0时，才能完全禁止一致性命名，得到eth0-N的设备名。
```

例如：

```
linux16 /vmlinuz-3.10.0-327.el7.x86\_64 root=UUID=edb1bf15-9590-4195-aa11-6dac45c7f6f3 ro rhgb quiet LANG=en\_US.UTF-8
```

另外，root启动参数有多种定义方式，可以使用UUID的方式指定，也可以直接指定根文件系统所在分区，如`root=/dev/sda2`，

### 1.4.6 initrd和initrd16命令 

```
initrd file
```

只能紧跟在linux或linux16命令之后使用，用于为即将启动的内核传递init ramdisk路径。

同样，基于根分区，可以使用绝对路径，也可以使用相对路径。路径的表示方法和linux或linux16命令相同。例如：

```
linux16 /vmlinuz-0-rescue-d13bce5e247540a5b5886f2bf8aabb35 root=UUID=b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8 ro crashkernel=auto quiet

initrd16 /initramfs-0-rescue-d13bce5e247540a5b5886f2bf8aabb35.img
```
### 1.4.7 search命令 

```
search [--file|--label|--fs-uuid] [--set [var]] [--no-floppy] [--hint args] name
```

通过文件`[--file]`、卷标`[--label]`、文件系统UUID`[--fs-uuid]`来搜索设备。

如果使用了`--set`选项，则会将第一个找到的设备设置为环境变量"var"的值，默认的变量"var"为'root'。

搜索时可使用`--no-floppy`选项来禁止搜索软盘，因为软盘速度非常慢，已经被淘汰了。

有时候还会指定`--hint=XXX`，表示优先选择满足提示条件的设备，若指定了多个hint条件，则优先匹配第一个hint，然后匹配第二个，依次类推。

例如：
```
if [ x$feature_platform_search_hint = xy ]; then
    search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1 --hint='hd0,msdos1' 367d6a77-033b-4037-bbcb-416705ead095
else
    search --no-floppy --fs-uuid --set=root 367d6a77-033b-4037-bbcb-416705ead095
fi

linux16 /vmlinuz-3.10.0-327.el7.x86_64 root=UUID=b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8 ro crashkernel=auto quiet LANG=en_US.UTF-8

initrd16 /initramfs-3.10.0-327.el7.x86_64.img
```
上述if语句中的第一个search中搜索uuid为"367d6a77-033b-4037-bbcb-416705ead095"的设备，但使用了多个hint选项，表示先匹配bios平台下/boot分区为(hd0,msdos1)的设备，之后还指定了几个hint，但因为search使用的是uuid搜索方式，所以这些hint选项是多余的，因为单磁盘上分区的uuid是唯一的。

再举个例子，如果某启动设备上有两个boot分区(如多系统共存时)，分别是(hd0,msdos1)和(hd0,msdos5)，如果此时不使用uuid搜索，而是使用label方式搜索:

```
search --no-floppy --fs-label=boot --set=root --hint=hd0,msdos5
```

则此时将会选中`(hd0,msdos5)`这个boot分区，若不使用hint，将选中(hd0,msdos1)这个boot分区。

### 1.4.8 true和false命令 

直接返回true或false布尔值。

### 1.4.9 test expression和\[ expression \] 

用法等同于test命令，用于计算"expression"的结果是否为真，为真时返回0，否则返回非0，主要用于if、while或until结构中。

### 1.4.10 cat命令 

读取文件内容，借此可以帮助判断哪个是boot分区，哪个是根分区。

交互式命令行下使用。

### 1.4.11 clear命令 

清屏。

### 1.4.12 configfile命令 

立即装载一个指定的文件作为grub的配置文件。但注意，导入的文件中的环境变量不在当前生效。

在grub.cfg丢失时，该命令将排上用场。

### 1.4.13 echo命令 

```
echo [-n] [-e] string
```

"-n"和"-e"用法同shell中echo。如果要引用变量，使用`${var}`的方式。

### 1.4.14 export命令 

导出环境变量，若在configfile的file中导出环境变量，将会在当前环境也生效。

### 1.4.15 halt和reboot命令 

关机或重启

### 1.4.16 ls命令 

```
ls [args]
```

如果不给定任何参数，则列出grub可见的设备。

如果给定的参数是一个分区，则显示该分区的文件系统信息。

如果给定的参数是一个绝对路径表示的目录，则显示该目录下的所有文件。

例如：

![](/img/linux/1699315174927.png)

### 1.4.17 probe命令 

```
probe [--set var] --partmap|--fs|--fs-uuid|--label device
```

探测分区或磁盘的属性信息。如果未指定`--set`，则显示指定设备对应的信息。如果指定了`--set`，则将对应信息的值赋给变量var。

```
--partmap：显示是gpt还是mbr格式的磁盘。
--fs：显示分区的文件系统。
--fs-uuid：显示分区的uuid值。
--label：显示分区的label值。
```

### 1.4.18 save\_env和list\_env命令 

将环境变量保存到环境变量块中，以及列出当前的环境变量块中的变量。

![](/img/linux/1699315257153.png)

### 1.4.19 loopback命令 

```
loopback [-d] device file
```

将file映射为回环设备。使用`-d`选项则是删除映射。

例如：

```
loopback loop0 /path/to/image
ls (loop0)/
```

### 1.4.20 normal和normal\_exit命令 

进入和退出normal模式，normal是相对于救援模式而言的，只要不是在救援模式下，就是在normal模式下。

救援模式下，只能使用非常少的命令，而normal模式下则可以使用非常多的命令。

### 1.4.21 password和password\_pbkdf2命令 

```
password user clear-password
password_pbkdf2 user hashed-password
```

前者使用明文密码定义一个名为user的用户。不建议使用此命令。

后者使用哈希加密后的密码定义一个名为user的用户，加密的密码通过`grub-mkpasswd-pbkdf2`工具生成。建议使用该命令。

## 1.5 几个常设置的内置变量 

### 1.5.1 chosen变量 

当开机时选中某个菜单项启动时，该菜单的title将被赋值给chosen变量。该变量一般只用于引用，而不用于修改。

### 1.5.2 cmdpath变量 

grub2加载的core.img的目录路径，是绝对路径，即包括了设备名的路径，如(hd0,gpt1)/boot/grub2/。该变量值不应该修改。

### 1.5.3 default变量 

指定默认的菜单项，一般其后都会跟随timeout变量。

default指定默认菜单时，可使用菜单的title，也可以使用菜单的id，或者数值顺序，当使用数值顺序指定default时，从0开始计算。

### 1.5.4 timeout变量 

设置菜单等待超时时间，设置为0时将直接启动默认菜单项而不显示菜单，设置为"-1"时将永久等待手动选择。

### 1.5.5 fallback变量 

当默认菜单项启动失败，则使用该变量指定的菜单项启动，指定方式同default，可使用数值(从0开始计算)、title或id指定。

### 1.5.6 grub\_platform变量 

指定该平台是"pc"还是"efi"，pc表示的就是传统的bios平台。

该变量不应该被修改，而应该被引用，例如用于if判断语句中。

### 1.5.7 prefix变量 

在grub启动的时候，grub自动将/boot/grub2目录的绝对路径赋值给该变量，使得以后可以直接从该变量所代表的目录下加载各文件或模块。

例如，可能自动设置为：

```
set prefix = (hd0,gpt1)/boot/grub2/
```

所以可以使用`$prefix/grubN.cfg`来引用/boot/grub2/grubN.cfg文件。

该变量不应该修改，且若手动设置，则必须设置正确，否则牵一发而动全身。

### 1.5.8 root变量 

该变量指定根设备的名称，使得后续使用从"/"开始的相对路径引用文件时将从该root变量指定的路径开始。一般该变量是grub启动的时候由grub根据prefix变量设置而来的。

例如`prefix=(hd0,gpt1)/boot/grub2`，则`root=(hd0,gpt1)`，后续就可以使用相对路径`/vmlinuz-XXX`表示`(hd0,gpt1)/vmlinuz-XXX`文件。

注意：在Linux中，从根"/"开始的路径表示绝对路径，如/etc/fstab。但grub中，从"/"开始的表示相对路径，其相对的基准是root变量设置的值，而使用`(dev\_name)/`开始的路径才表示绝对路径。

一般root变量都表示/boot所在的分区，但这不是绝对的，如果设置为根文件系统所在分区，如`root=(hd0,gpt2)`，则后续可以使用/etc/fstab来引用`(hd0,gpt2)/etc/fstab`文件。

该变量在grub2中一般不用修改，但若修改则必须指定正确。

另外，root变量还应该于linux或linux16命令所指定的内核启动参数`root=`区分开来，内核启动参数中的`root=`的意义是固定的，其指定的是根文件系统所在分区。例如：
```
set root='hd0,msdos1'

linux16 /vmlinuz-3.10.0-327.el7.x86_64 root=UUID=b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8 ro crashkernel=auto quiet LANG=en_US.UTF-8

initrd16 /initramfs-3.10.0-327.el7.x86_64.img
```
一般情况下，/boot都会单独分区，所以root变量指定的根设备和root启动参数所指定的根分区不是同一个分区，除非/boot不是单独的分区，而是在根分区下的一个目录。

## 1.6 grub配置和安装示例 

首先写一个grub.cfg。例如此处，在msdos磁盘上安装了两个操作系统，CentOS 7和CentOS 6。

```
# 设置一些全局环境变量
set default=0
set fallback=1
set timeout=3

# 将可能使用到的模块一次性装载完
# 支持msdos的模块
insmod part_msdos
# 支持各种文件系统的模块
insmod exfat
insmod ext2
insmod xfs
insmod fat
insmod iso9660

# 定义菜单
menuentry 'CentOS 7' --unrestricted {
        search --no-floppy --fs-uuid --set=root 367d6a77-033b-4037-bbcb-416705ead095
        linux16 /vmlinuz-3.10.0-327.el7.x86_64 root=UUID=b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8 ro biosdevname=0 net.ifnames=0 quiet
        initrd16 /initramfs-3.10.0-327.el7.x86_64.img
}
menuentry 'CentOS 6' --unrestricted {
        search --no-floppy --fs-uuid --set=root f5d8939c-4a04-4f47-a1bc-1b8cbabc4d32
        linux16 /vmlinuz-2.6.32-504.el6.x86_64 root=UUID=edb1bf15-9590-4195-aa11-6dac45c7f6f3 ro quiet
        initrd16 /initramfs-2.6.32-504.el6.x86_64.img
}
```

然后执行grub安装操作。

```
shell> grub2-install /dev/sda
```

## 1.7 传统grub简述 

因为本文主要介绍grub2，所以传统的grub只简单介绍下，其实前面已经提及了很多传统grub和grub2的比较了。另外，传统grub已足够强大，足够应付一般的需求。

### 1.7.1 grub安装 

例如安装到/dev/sda上。

```
shell> grub-install /dev/sda
```

### 1.7.2 grub.conf配置 

```
default=0  # 默认启动第一个系统
timeout=5  # 等待超时时间5秒
splashimage=(hd0,0)/grub/splash.xpm.gz  # 背景图片
hiddenmenu  # 隐藏菜单，若要显式，在启动时按下ESC
title Red Hat Enterprise Linux AS (2.6.18-92.el5)  # 定义操作系统的说明信息
    root (hd0,0) 
    kernel /vmlinuz-2.6.18-92.el5 ro root＝/dev/sda2 rhgb quiet
    initrd /initrd-2.6.18-92.el5.img
```

在说明配置方法之前，需要说明一个关键点，boot是否是一个独立的分区，它影响后面路径的配置。

在一个正常的操作系统中查看/boot/grub/grub.conf文件，可以在NOTICE段看到提示，说你是否拥有一个独立的boot分区？如果有则意味着kernel和initrd的路径是从/开始的而不是/boot开始的，如/vmlinuz-xxx，如果没有独立的boot分区，则kernel和initrd的路径中需要指明boot路径，例如Boot没有分区而是在/文件系统下的一个目录，则/boot/vmlinuz-xxx。

root (hd0,0)定义grub识别的根。一般定义的都是boot所在的分区，grub只能识别hd，所以这里只能使用hd，hd0表示在第一块磁盘上，hd0,0的第二个0表示boot在第一个分区上，grub2在分区的计算上是从1开始的，这是传统grub和grub2不同的地方。

kernel定义内核文件的路径和启动参数，等价于grub2的linux命令或linux16命令。首先说明参数，ro表示只读，`root=/dev/sda[N]`或者`root=UUID="device_uuid_num"`指定根文件系统所在的分区，这是必须的参数。rhgb表示在操作系统启动过程中使用图形界面输出一些信息，将其省略可以加快启动速度，quiet表示启动操作系统时静默输出信息。再说明路径，如果是boot是独立分区的，则kernel的路径定义方式为/vmlinuz-xxx，如果没有独立分区，则指明其绝对路径，一般都是在根文件系统下的目录，所以一般为/boot/vmlinuz-xxx。

initrd定义init ramdisk的路径，路径的定义方式同kernel。除了路径之外没有任何参数。

![](/img/linux/1699315389669.png)

或者使用下图的UUID的方式。

![](/img/linux/1699315399401.png)

如果没有指定`root=`的选项，将报错`no or empty root …… dracut…kernel panic`的错误。如下图。

![](/img/linux/1699315411714.png)