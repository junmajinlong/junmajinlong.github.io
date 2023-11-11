---
title: PXE+kickstart无人值守安装CentOS 7
p: linux/pxe_centos7.md
date: 2019-07-10 12:29:45
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# PXE+kickstart无人值守安装CentOS 7

本文是[PXE+kickstart无人值守安装CentOS6][1]的续篇，主要是为了突出CentOS7和CentOS6配置kickstart时的不同点，例如`pxelinux.cfg/default`文件的变化，kickstart使用nfs提供时的bug等。为了文章的完整性和独立性，将很多CentOS6上直接复制搬到了本文。


## PXE说明

所谓的PXE是Preboot Execution Environment的缩写，字面上的意思是开机前的执行环境。

要达成PXE必须要有两个环节：

- (1)一个是客户端的网卡必须要支持PXE用户端功能，并且开机时选择从网卡启动，这样系统才会以网卡进入PXE客户端的程序；
- (2)一个是PXE服务器必须要提供至少含有DHCP以及TFTP的服务！且其中：
  - DHCP服务必须要能够提供客户端的网络参数，还要告知客户端TFTP所在的位置；
  - TFTP则提供客户端的boot loader及kernel file下载路径。

还要加上NFS/FTP/HTTP(选择一样即可)等提供安装文件(安装镜像的解压文件)，才算是比较完整的PXE服务器。一般TFTP和DHCP服务都由同一台服务器提供，且大多数时候还提供NFS/FTP/HTTP服务，所以PXE服务器一般是提供3合一的服务。

## PXE流程

如下图：图片来源于网络，虽不易理解，但细节描述的很好。

![](/img/linux/733013-20170811005619855-1661502854.png)

- (1).**Client向PXE Server上的DHCP发送IP地址请求消息**，DHCP检测Client是否合法（主要是检测Client的网卡MAC地址），如果合法则返回Client的IP地址，同时将pxe环境下的Boot loader文件pxelinux.0的位置信息传送给Client。
- (2).**Client向PXE Server上的TFTP请求pxelinux.0**，TFTP接收到消息之后再向Client发送pxelinux.0大小信息，试探Client是否满意，当TFTP收到Client发回的同意大小信息之后，正式向Client发送pxelinux.0。
- (3).**Client执行接收到的pxelinux.0文件**。
- (4).**Client向TFTP请求pxelinux.cfg文件**(其实它是目录，里面放置的是是启动菜单，即grub的配置文件)，TFTP将配置文件发回Client，继而Client根据配置文件执行后续操作。
- (5).**Client向TFTP发送Linux内核请求信息**，TFTP接收到消息之后将内核文件发送给Client。
- (6).**Client向TFTP发送根文件请求信息**，TFTP接收到消息之后返回Linux根文件系统。
- (7).**Client加载Linux内核**（启动参数已经在4中的配置文件中设置好了）。
- (8).**Client通过nfs/ftp/http下载系统安装文件进行安装**。如果在4中的配置文件指定了kickstart路径，则会根据此文件自动应答安装系统。

## 部署环境说明

如下图，172..16.10.10是PXE服务器，提供dhcp+tftp+nfs服务。其他该网段内的主机为待安装系统的主机群。

![](/img/linux/733013-20170811005506574-720002604.png)



## 部署DHCP服务

首先安装dhcp服务端程序。

```
yum -y install dhcp
```

DHCP主要是提供客户端网络参数与TFTP的位置，以及boot loader的文件名。同时，我们仅针对内网来告知TFTP的相关位置，所以可以编辑/etc/dhcp/dhcpd.conf在subnet的区块内加入两个参数即可。其中PXE上专门为PXE客户端下载的boot loader文件名称为pxelinux.0。

```
vim /etc/dhcp/dhcpd.conf
ddns-update-style none;
default-lease-time 259200;
max-lease-time 518400;    
option routers 172.16.10.10;
option domain-name-servers 172.16.10.10;
subnet 172.16.10.0 netmask 255.255.255.0 {
        range 172.16.10.11 172.16.10.100;
        option subnet-mask 255.255.255.0;
        next-server 172.16.10.10;            # 就是TFTP的位置
        filename "pxelinux.0";               # 告知得从TFTP根目录下载的boot loader文件名
}
```

重启dhcp

```
systemctl start dhcpd
```

## 部署TFTP

从流程图中可以看出，boot loader文件pxelinux.0以及内核相关的配置文件(目录pxelinux.cfg下)主要都是由TFTP来提供的！

TFTP的安装很简单，直接使用yum即可。不过要告诉客户端TFTP的根目录在哪里，这样客户端才能找到相关文件。另外要注意，TFTP是由xinetd这个super daemon所管理的，因此设定好TFTP之后，要启动的是xinetd。

```
yum install tftp-server
yum -y install xinetd
```

默认TFTP服务的根目录是/var/lib/tftpboot/，为了少写些字母，将tftp的根目录修改为/tftpboot/。修改tftp的配置文件，主要是TFTP的根目录。

```
vim /etc/xinetd.d/tftp

service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        
        # 重点在这里！修改tftp的chroot根目录
        server_args             = -s /tftpboot
        disable                 = no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
```

创建tftp的根目录。

```
mkdir /tftpboot
```

启动TFTP并观察之：

```
systemctl start tftp

netstat -tulnp | grep xinetd
udp     0    0 0.0.0.0:69    0.0.0.0:*    28465/xinetd 
```

接下来的文件必须要放置于/tftpboot/目录下。

## 提供pxe的bootloader和相关配置文件

如果要使用PXE的开机引导的话，需要使用CentOS提供的syslinux包，从中copy两个文件到tftp的根目录/tftpboot下即可。整个过程如下：

```
yum -y install syslinux
cp -a /usr/share/syslinux/{menu.c32,vesamenu.c32,pxelinux.0}  /tftpboot/
mkdir /tftpboot/pxelinux.cfg
ls -l /tftpboot/
-rw-r--r-- 1 root root  61796 Oct 16  2014 menu.c32      # 提供图形化菜单功能
-rw-r--r-- 1 root root  26759 Oct 16  2014 pxelinux.0    # boot loader文件
drwxr-xr-x 2 root root   4096 Feb 24 20:02 pxelinux.cfg  # 开机的菜单设定在这里
-rw-r--r-- 1 root root 163728 Oct 16  2014 vesamenu.c32  # 也是提供图形化菜单功能，但界面和menu.c32不同
```

pxelinux.cfg是个目录，可以放置默认的开机选项，也可以针对不同的客户端主机提供不同的开机选项。一般来说，可以在pxelinux.cfg目录内建立一个名为default的文件来提供默认选项。

如果没有menu.c32或vesamenu.c32时，菜单会以纯文字模式一行一行显示。如果使用menu.c32或vesamenu.c32，就会有类似反白效果出现，此时可以使用上下键来选择选项，而不需要看着屏幕去输入数字键来选择开机选项。经过测试，使用vesamenu.c32比menu.c32更加好看些。

这部分设定完毕后，就是内核相关的设定了。

## 从安装镜像中获取Linux内核文件

要安装Linux系统，必须提供Linux内核文件和initrd文件，这里以64位版本的CentOS 7.2为例。

这里计划将内核相关文件放在/tftpboot/CentOS7.2/目录下。既然要从安装镜像中获取内核相关文件，首先得要挂载镜像。

```
mount /dev/cdrom /test
mkdir /tftpboot/CentOS7.2
cp /test/isolinux/{vmlinuz,initrd.img} /tftpboot/CentOS7.2
cp /test/isolinux/isolinux.cfg /tftpboot/pxelinux.cfg/default
```

其实仅需要vmlinuz和initrd.img两个文件即可，不过这里还将isolinux.cfg这个文件拷贝出来了，这个文件里提供了开机选项，可以以它作为修改开机选项和菜单的模板，这样修改起来比较容易，也更便捷！

## 设置开机菜单并提供系统安装文件

以下是CentOS 7.2中syslinux包中提供的isolinux.cfg中提供的默认内容。

```
[root@xuexi ~]# cat /tftpboot/pxelinux.cfg/default
default vesamenu.c32   # 这是必须项，或者使用menu.c32
timeout 600  # 超时等待时间，60秒内不操作将自动选择默认的菜单来加载

display boot.msg  # 这是为选项提供一些说明的文件

# Clear the screen when exiting the menu, instead of leaving the menu displayed.
# For vesamenu, this means the graphical background is still displayed without
# the menu itself for as long as the screen remains in graphics mode.
menu clear
menu background splash.png   # 背景图片
menu title CentOS 7          # 大标题
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

# Border Area
menu color border * #00000000 #00000000 none

# Selected item
menu color sel 0 #ffffffff #00000000 none

# Title bar
menu color title 0 #ff7ba3d0 #00000000 none

# Press [Tab] message
menu color tabmsg 0 #ff3a6496 #00000000 none

# Unselected menu item
menu color unsel 0 #84b8ffff #00000000 none

# Selected hotkey
menu color hotsel 0 #84b8ffff #00000000 none

# Unselected hotkey
menu color hotkey 0 #ffffffff #00000000 none

# Help text
menu color help 0 #ffffffff #00000000 none

# A scrollbar of some type? Not sure.
menu color scrollbar 0 #ffffffff #ff355594 none

# Timeout msg
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none

# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.

menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Install CentOS 7   # 菜单文字
  
  # 内核文件路径，注意相对路径是从tftp的根路径/tftpboot开始的，
  # 所以要改为"./CentOS7.2/vmlinuz"
  kernel vmlinuz
  
  # 内核启动选项，其中包括initrd的路径，同样要改为"./CentOS7.2/initrd.img"
  # stage2文件的搜索路径，搜索的文件一般是".treeinfo"，找不到该文件则找LiveOS/squashfs.img
  # 一般pxe环境下此路径直接指向系统安装文件的路径，具体做法见下文示例
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 quiet  

label check
  menu label Test this ^media & install CentOS 7
  menu default  # menu default表示开机时光标一开始默认停留在此label上
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet

menu separator # insert an empty line

# utilities submenu          # 子菜单项的设置方法
menu begin ^Troubleshooting
  menu title Troubleshooting

label vesa
  menu indent count 5
  menu label Install CentOS 7 in ^basic graphics mode
  text help
        Try this option out if you're having trouble installing
        CentOS 7.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 xdriver=vesa nomodeset quiet

label rescue
  menu indent count 5
  menu label ^Rescue a CentOS system
  text help
        If the system will not boot, this lets you access files
        and edit config files to try to get it booting again.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rescue quiet

label memtest
  menu label Run a ^memory test
  text help
        If your system is having issues, a problem with your
        system's memory may be the cause. Use this utility to
        see if the memory is working correctly.
  endtext
  kernel memtest

menu separator # insert an empty line

label local
  menu label Boot from ^local drive
  localboot 0xffff

menu separator # insert an empty line
menu separator # insert an empty line

label returntomain
  menu label Return to ^main menu
  menu exit

menu end
```

所以，将其稍作修改，使其适合做pxe的菜单配置文件。

```
default vesamenu.c32  
timeout 600           

display boot.msg      

menu clear
menu background splash.png
menu title CentOS 7 menu
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

menu color border * #00000000 #00000000 none
menu color sel 0 #ffffffff #00000000 none
menu color title 0 #ff7ba3d0 #00000000 none
menu color tabmsg 0 #ff3a6496 #00000000 none
menu color unsel 0 #84b8ffff #00000000 none
menu color hotsel 0 #84b8ffff #00000000 none
menu color hotkey 0 #ffffffff #00000000 none
menu color help 0 #ffffffff #00000000 none
menu color scrollbar 0 #ffffffff #ff355594 none
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

label linux
  menu label ^Install CentOS 7.2 through pxe
  menu default
  kernel "./CentOS7.2/vmlinuz"
  append initrd="./CentOS7.2/initrd.img" inst.stage2=ftp://172.16.10.10 quiet net.ifnames=0 biosdevname=0
```

其中`net.ifnames=0 biosdevname=0`这两个内核启动参数是为了让网卡名称为ethN，而不是默认的eno16777728这样的随机名称。

注意示例中stage2的路径是放在ftp的路径下(vsftpd根目录/var/ftp/)，所以先将镜像文件中的系统安装文件提取出来放到/var/ftp/下。当然，除了ftp，还支持nfs/http。但是，CentOS7.2在pxe+kickstart时对NFS的支持出现了bug，所以不建议使用nfs，当使用nfs出现各种疑难杂症时请换回ftp或http。

```
yum -y install vsftpd
cp -a /test/* /var/ftp/
systemctl start vsftpd
```



## 开机测试

新开一个虚拟机，进入bios界面设置从网卡启动。将首先搜索DHCP服务器，找到DHCP后搜索bootloader文件，启动菜单设置文件等，然后进入启动菜单等待选择要启动的项。如下：

 ![](/img/linux/733013-20170811010346839-1647400781.png)

![](/img/linux/733013-20170811010350964-1563400162.png)

因为只设置了一个启动项，所以菜单中只有一项。启动它，将加载一系列文件，直到出现安装操作界面。

![](/img/linux/733013-20170811010401120-1548449224.png)

然后就可以直接操作安装系统了。但这样毕竟是手动操作，无法实现批量系统安装，所以要提供一个自动应答文件，每一次的手动操作步骤都由自动应答文件中给定的项来应答，这样就能实现自动安装操作系统，也就能实现批量系统安装。

## 通过pxe+kickstart实现无人值守批量安装操作系统

所谓的无人值守，就是自动应答，当安装过程中需要人机交互提供某些选项的答案时（如如何分区），自动应答文件可以根据对应项自动提供答案。但是，无人值守并不完全是无人值守，至少设置bios从网卡启动是必须人为设置的，且安装完系统后设置不从网卡启动也是需要人为设置的。除此之外，其他的基本上都可以实现无人值守安装。

要配置无人值守的系统安装环境，需要提供安装过程中需要的各种答案，这些答案在kickstart的配置文件中设置，一般正常安装完Linux系统在root用户的家目录下有一个anaconda-ks.cfg，该文件的选项说明见[kickstart文件详细说明][2]。

以下是修改后该文件中的内容，将用来做kickstart应答文件。并设置由ftp服务来提供该文件，所以将kickstart文件保存到ftp的pub目录中。

```
[root@xuexi ~]# cp -a ~/anaconda-ks.cfg /var/ftp/pub/ks.cfg
[root@xuexi ~]# chmod +r /var/ftp/pub/ks.cfg     # 必须要保证ks.cfg是全局可读的
[root@xuexi ~]# cat anaconda-ks.cfg
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Install OS instead of upgrade
install
# Use network installation
url --url="ftp://172.16.10.10"
#url --url="http://192.168.100.53/cblr/links/CentOS7.2-x86_64"
#nfs --server=172.16.10.10 --dir=/install
# Use text mode install
text
# Firewall configuration
firewall --disabled
firstboot --disable
ignoredisk --only-use=sda
# Keyboard layouts
# old format: keyboard us
# new format:
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --onboot=yes --bootproto=dhcp --device=eth0 --noipv6
network  --hostname=node1.xuexi.com
# Reboot after installation
reboot
# Root password
rootpw --iscrypted $6$KIPkwGVYqtjHln80$quxmkE5MKKA2LyzLOAc/s3FWH/jX76sObq6hqwOsEBoeMc/wIrzGG4xm72lkXwLeOfRLS/sl5vdajY9j34D4J. 
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx
# System timezone
timezone Asia/Shanghai
# System bootloader configuration
bootloader --append="quiet crashkernel=auto" --location=mbr --boot-drive=sda
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
part /boot --asprimary --fstype="xfs" --size=250
part swap --fstype="swap" --size=2000
part / --asprimary --fstype="xfs" --grow --size=5000

# 如果是要LVM分区，则考虑以下分区
# part /boot --fstype ext4 --size=100
# part swap --fstype=swap --size=2048
# part pv26 --size=100 --grow
# volgroup VG00 --pesize=32768 pv26
# logvol / --fstype ext4 --name=LVroot --vgname=VG00 --size=29984
# logvol /data --fstype ext4 --name=LVdata --vgname=VG00 --size=100 --grow

%post
rm -f /etc/yum.repos.d/*
cat >>/etc/yum.repos.d/base.repo<<eof
[base]
name=sohu
baseurl=http://mirrors.sohu.com/centos/7/os/x86_64/
gpgcheck=0
enable=1
[epel]
name=epel
baseurl=http://mirrors.aliyun.com/epel/7Server/x86_64/
enable=1
gpgcheck=0
eof

sed -i "s/rhgb //" /boot/grub2/grub.cfg
sed -i "s/ONBOOT.*$/ONBOOT=yes/" /etc/sysconfig/network-scripts/ifcfg-eth0
sed -i "/UUID/d" /etc/sysconfig/network-scripts/ifcfg-eth0
echo "DNS1=114.114.114.114" >> /etc/sysconfig/network-scripts/ifcfg-eth0
echo "UseDNS no" >> /etc/ssh/sshd_config
sed -i "s/^SELINUX=.*$/SELINUX=disabled/" /etc/sysconfig/selinux
systemctl disable firewalld
%end

%packages
@base
@core
@development
@platform-devel
kexec-tools
lftp
tree
lrzsz

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end
```

设置后，修改`/tftpboot/pxelinux.cfg/default`文件，在其中的内核启动参数上加上一项kickstart文件的寻找路径。

```
vim /tftpboot/pxelinux.cfg/default
label linux
  menu label ^Install CentOS 7.2 through pxe
  menu default
  kernel "./CentOS7.2/vmlinuz"
  append initrd="./CentOS7.2/initrd.img" inst.stage2=ftp://172.16.10.10 ks=ftp://172.16.10.10/pub/ks.cfg quiet net.ifnames=0 biosdevname=0

# 如果使用nfs提供安装文件和kickstart文件，则ks参数必须使用nfs4协议，
# 即使使用了nfs4，仍然无法实现无人值守，这是bug
  append initrd="./CentOS7.2/initrd.img" inst.stage2=nfs:172.16.10.10:/install ks=nfs4:172.16.10.10:/install/ks.cfg quiet net.ifnames=0 biosdevname=0
```

注意注释行中使用nfs4而不是nfs，否则在安装系统时将报错，如下。不知道为什么到CentOS7.2还需要明确指定nfs4，算是bug吧，在redhat的bug提交区已经有用户提交相关问题。

![](/img/linux/733013-20170811010657480-1064846370.png)

但即使使用nfs4协议，虽然能够读取kickstart文件，但却无法生效，即无法实现自动应答，仍然需要手动操作。

所以，建议使用ftp或者http，暂时不要使用NFS。但这个bug只针对CentOS 7，CentOS 6是没有任何问题的。

回归正题，现在已经设置好`/tftpboot/pxelinux.cfg/default`和`/var/ftp/pub/ks.cfg`，所以可以进行无人值守安装Linux了。



[1]: /linux/pxe_centos6	"PXE+Kickstart安装CentOS6"
[2]: /linux/kickstart_config "kickstart配置文件"