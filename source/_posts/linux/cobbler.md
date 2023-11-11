---
title: cobbler无人值守批量安装Linux系统
p: linux/cobbler.md
date: 2019-07-10 12:29:48
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# cobbler无人值守批量安装Linux系统

在阅读本文之前，如果不懂pxe+kickstart，建议先学习下，对学习cobbler很有帮助。可以参考我所写的[pxe+kickstart无人值守安装CentOS6][1]文章。

## pxe安装系统

pxe的大致过程如下图。

![](/img/linux/733013-20170811191008054-1179900786.png)

其中pxelinux.0为bootloader。pxelinux.cfg目录下的文件(一般使用默认的default文件)定义了安装操作系统前的菜单项，如kernel和Initrd的路径，kickstart的路径等。

首先客户端请求pxe服务器上的dhcp，dhcp上指定了next-server和filename，它们分别是tftpd的地址和pxelinux.0的路径；然后客户端请求tftpd获取pxelinux.0，执行pxelinux.0后将引导进入安装界面，随后获取pxelinux.cfg目录下的文件并读取其中的配置，从中获取kernel和initrd的路径所在，如果有定义kickstart项则还会去获取kickstart文件并读取配置；再然后客户端请求获取kernel和initrd文件，以展开内核并进入到根文件系统；最后客户端获取完成系统安装所需的其他文件，这些文件可以是在pxe的本地，也可以是互联网上等能获取到的地方。

## cobbler基本介绍

cobbler可以看作是一个更多功能的pxe，它实现系统安装和pxe也差不多，需要的文件和过程大致都一样。

cobbler能自动管理dns/tftp/dhcp/rsync这四个服务(但似乎对tftp的管理有点bug，需要手动启动tftp)，且cobbler依赖于httpd(pxe支持http/nfs/ftp)。

基本的系统安装，cobbler只需生成一个distro和一个profile即可。

distro相当于一个镜像，它提供安装系统过程中所需的一切文件，如vmlinuz,initrd以及rpm包等。

profile的作用是为了自动修改pxelinux.cfg/default文件，每生成或修改一次profile，都会在default文件中修改或追加对应的label。

除了distro/profile之外，cobbler还管理system/images/repositories等，但是用的很少。

## 安装和配置cobbler

### 安装cobbler

cobbler在epel源中提供。由于还依赖于httpd、dhcp，所以httpd和dhcp也应该装上。

```
yum -y install cobbler cobbler-web pykickstart debmirror httpd dhcp
```

其中cobbler-web是提供web管理界面的，pykicstart是检查kicstart文件语法错误的，debmirror是维护debian源的工具，此处用不上但有依赖关系，所以装上。

安装后，在/etc/cobbler生成以下文件。

```
[root@xuexi ~]# cd /etc/cobbler/

[root@xuexi cobbler]# ls
auth.conf       distro_signatures.json  modules.conf    reporting
tftpd.template  zone_templates cheetah_macros  dnsmasq.template
mongodb.conf    rsync.exclude  users.conf  cobbler_bash
import_rsync_whitelist  named.template  rsync.template users.digest
completions     iso    power secondary.template  version
dhcp.template   ldap   pxe   settings            zone.template
```

![](/img/linux/733013-20170811202136335-1945255957.png)

先启动httpd，再启动cobblerd。

```
[root@xuexi cobbler]# systemctl start httpd.service
[root@xuexi cobbler]# systemctl start cobblerd.service
[root@xuexi cobbler]# netstat -tnlp
tcp   0   0 0.0.0.0:22        0.0.0.0:*    LISTEN  1298/sshd          
tcp   0   0 127.0.0.1:25      0.0.0.0:*    LISTEN  1402/master        
tcp   0   0 127.0.0.1:25151   0.0.0.0:*    LISTEN  14091/python2       
tcp   0   0 0.0.0.0:3306      0.0.0.0:*    LISTEN  2261/mysqld        
tcp   0   0 :::22             :::*         LISTEN  1298/sshd          
tcp   0   0 ::1:25            :::*         LISTEN  1402/master        
tcp   0   0 :::443            :::*         LISTEN  14037/httpd        
tcp   0   0 :::80             :::*         LISTEN  14037/httpd
```

启动之后，首先执行cobbler check检查配置是否正确。根据提示修改相关的配置项。

```
[root@xuexi cobbler]# cobbler check
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
4 : change 'disable' to 'no' in /etc/xinetd.d/rsync
5 : comment out 'dists' on /etc/debmirror.conf for proper debian support
6 : comment out 'arches' on /etc/debmirror.conf for proper debian support
7 : ksvalidator was not found, install pykickstart
8 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
9 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
```

第一和第二个问题：

```
[root@xuexi cobbler]# vim /etc/cobbler/settings

next_server: 172.16.10.10
server: 172.16.10.10
```

第三个问题：获取pxelinux.0和menu.c32文件(对于centos来说只需这两个文件)，可以像pxe一样从syslinux包中手动复制到/var/lib/cobbler/loaders目录下，也可以执行cobbler get-loaders自动下载，但要求联网。

```
[root@xuexi cobbler]# cobbler get-loaders
```

第四个问题：有可能该问题不是如此的，而是说要将rsyncd.service使用给start且enable，只需`systemctl enable rsyncd`，`systemctl start rsyncd`。

```
[root@xuexi cobbler]# vim /etc/xinetd.d/rsync
disable=no
[root@xuexi cobbler]# service xinetd start
```

第5、6个问题，注释掉/etc/debmirror.conf中相关项即可。

第7个问题：因为之前安装的时候写成了pykicstart，所以出错了这里。

```
[root@xuexi cobbler]# yum -y install pykickstart
```

第8个问题：

```
[root@xuexi cobbler]# openssl passwd -1 -salt `openssl rand -hex 8` '123456'
$1$77e1022c$D9rxuxUWdc0NN46gzj9XT.
[root@xuexi cobbler]# vim /etc/cobbler/settings
default_password_crypted: "$1$77e1022c$D9rxuxUWdc0NN46gzj9XT."
```

第九个问题和电源管理有关，不用管了。直接重启cobbler，然后cobbler sync。

```
[root@xuexi cobbler]# service cobblerd restart

[root@xuexi cobbler]# cobbler check
The following are potential configuration items that you may want to fix:
1 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
[root@xuexi cobbler]# cobbler sync
```

 cobbler sync命令用于将tftpboot目录和/var/www/cobbler保持最新，当/var/lib/cobbler或者kickstart文件发生了变化，应该执行一次cobbler sync或者直接重启cobbler服务。

### 配置dhcp和tftp

如果在/etc/cobbler/setting中设置了manage_dhcp:1，表示由cobbler管理dhcp(默认为0即人为手动管理)，则cobbler管理的dhcp的配置模板/etc/cobbler/dhcp.template会覆盖/etc/dhcp/dhcpd.conf中配置，所以应该修改dhcp.template。

此处采用默认的不由cobbler管理dhcp。

```
[root@xuexi cobbler]# yum-y install dhcp
[root@xuexi cobbler]# vim /etc/dhcp/dhcpd.conf
ddns-update-style none;
default-lease-time 259200;
max-lease-time 518400;
subnet 172.16.10.0 netmask 255.255.255.0 {
        range 172.16.10.20 172.16.10.50;
        option subnet-mask 255.255.255.0;
        next-server 172.16.10.10;          # tftp的地址
        filename "pxelinux.0";             # pxelinux.0的路径，此为tftp根目录(/var/lib/tftpboot)的相对路径
}
[root@xuexi cobbler]# service dhcpd restart
```

关于tftp，在/etc/cobbler/settings中默认启用了由cobbler管理tftp，所以此处无需配置它。只要知道它的根目录为/var/lib/tftpboot即可。但是如果后面装系统的时候如果找不到tftp(应该是cobbler管理tftp的bug)，则手动启动tftp即可。

## cobbler从本地光盘安装系统

### 生成distro

生成distro的方法有多种，可以从本地镜像导入生成，也可以根据网络上的资源生成。显然，从本地生成的效率是最好的。

从本地导入的过程实际上是将系统镜像中的文件复制到/var/www/cobbler/目录(默认)下。

```
mkdir /mnt
mount /dev/cdrom /mnt
cobbler import --name=CentOS7.2 --path=/mnt
```

等待导入完成，则表示distro生成完成。

```
[root@xuexi cobbler]# ls -l /var/www/cobbler/images/CentOS7.2-x86_64/
total 38056
-r--r--r-- 3 root root 34815427 Oct 24  2014 initrd.img
-r-xr-xr-x 3 root root  4152336 Oct 24  2014 vmlinuz

# 此目录完全来源于镜像
[root@xuexi cobbler]# ls -l /var/www/cobbler/ks_mirror/CentOS7.2/
total 340
-r--r--r-- 1 root root     14 Oct 24  2014 CentOS_BuildTag
dr-xr-xr-x 3 root root   4096 Oct 24  2014 EFI
-r--r--r-- 1 root root    212 Nov 28  2013 EULA
-r--r--r-- 1 root root  18009 Nov 28  2013 GPL
dr-xr-xr-x 3 root root   4096 Oct 24  2014 images
dr-xr-xr-x 2 root root   4096 Oct 24  2014 isolinux
dr-xr-xr-x 2 root root 278528 Oct 24  2014 Packages
-r--r--r-- 1 root root   1354 Oct 20  2014 RELEASE-NOTES-en-US.html
dr-xr-xr-x 2 root root   4096 Oct 24  2014 repodata
-r--r--r-- 1 root root   1706 Nov 28  2013 RPM-GPG-KEY-CentOS-6
-r--r--r-- 1 root root   1730 Nov 28  2013 RPM-GPG-KEY-CentOS-Debug-6
-r--r--r-- 1 root root   1730 Nov 28  2013 RPM-GPG-KEY-CentOS-Security-6
-r--r--r-- 1 root root   1734 Nov 28  2013 RPM-GPG-KEY-CentOS-Testing-6
-r--r--r-- 1 root root   3380 Oct 24  2014 TRANS.TBL
```

确保url路径`http://172.16.10.10/cobbler/ks_mirror/CentOS7.2/`是有效的。

![](/img/linux/733013-20170811202440132-1268760937.png)

### 提供kickstart文件

以下是CentOS7的Kickstart内容。如果要改为适合CentOS6的内容，只需将keyboard项设置为"keyboard us"，并修改下分区方式(如有必要的话)以及%post脚本段的内容即可。

```
[root@xuexi ~]# vim /var/lib/cobbler/kickstarts/CentOS7.2.ks
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Install OS instead of upgrade
install
# Use network installation
url --url=$tree
# Use text mode install
text
# Firewall configuration
firewall --disabled
firstboot --disable
# 此项是CentOS7默认的项，但cobbler编译ks文件时不支持此语法，所以必须将此项注释掉
# ignoredisk --only-use=sda
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
rootpw --iscrypted $6$KIPkwGVYqtjHln80$密码
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
cat >>/etc/yum.repos.d/my.repo<<eof
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
sed -i "/UUID/d" /etc/sysconfig/network-scripts/ifcfg-eth0
echo "DNS1=114.114.114.114" >> /etc/sysconfig/network-scripts/ifcfg-eth0
echo "UseDNS no" >> /etc/ssh/sshd_config
sed -i "s/GSSAPIAuthentication yes/GSSAPIAuthentication no/" /etc/ssh/ssh_config
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

上面的url也可以写成`url --url="http://172.16.10.10/cobbler/ks_mirror/CentOS7.2/"`。

### 提供profile

在导入镜像生成distro的过程中，会自动生成一个profile。

```
[root@xuexi cobbler]# cobbler profile list
   CentOS7.2-x86_64
```

该profile默认使用的kickstart是/var/lib/cobbler/kickstarts/sample_end.ks，所以需要修改此项。

```
[root@xuexi cobbler]# cobbler profile report --name=CentOS7.2-x86_64
Name                           : CentOS7.2-x86_64
TFTP Boot Files                : {}
Comment                        : 
DHCP Tag                       : default
Distribution                   : CentOS7.2-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/sample_end.ks
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 : 
Internal proxy                 : 
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      : 
Virt RAM (MB)                  : 512
Virt Type                      : kvm
[root@xuexi cobbler]# cobbler profile edit --name=CentOS7.2-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS7.2.ks

[root@xuexi cobbler]# cobbler profile report --name=CentOS7.2-x86_64 | grep -i kickstart
Kickstart                      : /var/lib/cobbler/kickstarts/CentOS7.2.ks
Kickstart Metadata             : {}
```

对于centos7系列，则加上内核启动参数net.ifnames和biosdevname使得网卡名使用ethN系列而不使用enoXXXXXXX这样的随机名称。

```
[root@xuexi cobbler]# cobbler profile edit --name=CentOS7.2-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS7.2.ks --kopts="net.ifnames=0 biosdevname=0"
[root@xuexi cobbler]# cobbler profile report --name=CentOS7.2-x86_64 | grep -Ei 'kernel|kickstart'                                                  
Kernel Options                 : {'biosdevname': '0', 'net.ifnames': '0'}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/CentOS7.2.ks
Kickstart Metadata             : {}
```

当然，不使用自生成的profile，自己添加一个profile也可以，同时还可以设置profile选项，如`--kickstart`项。如下：其中`--distro`指定该profile是添加到哪个distro下的。

```
[root@xuexi cobbler]# cobbler profile add --name=CentOS7.2.1-x86_64 --distro=CentOS7.2-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS7.2.ks
```

实际上，每添加一个profile都是在向/var/lib/tftpboot/pxelinux.cfg/default中添加一个label。

```
[root@xuexi cobbler]# cat /var/lib/tftpboot/pxelinux.cfg/default   
DEFAULT menu
PROMPT 0
MENU TITLE Cobbler | http://cobbler.github.io/
TIMEOUT 200
TOTALTIMEOUT 6000
ONTIMEOUT local

LABEL local
        MENU LABEL (local)
        MENU DEFAULT
        LOCALBOOT -1

LABEL CentOS7.2-x86_64
        kernel /images/CentOS7.2-x86_64/vmlinuz
        MENU LABEL CentOS7.2-x86_64
        append initrd=/images/CentOS7.2-x86_64/initrd.img ksdevice=bootif lang=  text net.ifnames=0 biosdevname=0 kssendmac  ks=http://172.16.10.10/cblr/svc/op/ks/profile/CentOS7.2-x86_64
        ipappend 2

LABEL CentOS7.2.1-x86_64
        kernel /images/CentOS7.2-x86_64/vmlinuz
        MENU LABEL CentOS7.2.1-x86_64
        append initrd=/images/CentOS7.2-x86_64/initrd.img ksdevice=bootif lang=  kssendmac text  ks=http://172.16.10.10/cblr/svc/op/ks/profile/CentOS7.2.1-x86_64
        ipappend 2

MENU end
```

也就是说，其实可以不用生成profile，自己手动编辑label也可以。

默认使用的菜单背景图片是menu.c32，此处我改为vesamenu.c32，该背景图片是从syslinux包中提取的，背景图片而已，看个人喜好了。另外默认菜单等待时间是2秒，在自动安装的环境中，可以将其设置的短些。并且进入菜单默认停留在local，即从本地启动系统，但是此时系统还没装，所以要实现自动化，建议修改此项。

以下是修改后的项。

```
DEFAULT vemamenu
DEFAULT menu
PROMPT 0
MENU TITLE Cobbler | http://cobbler.github.io/
TIMEOUT 20
TOTALTIMEOUT 6000
ONTIMEOUT CentOS7.2-x86_64

LABEL local
        MENU LABEL (local)
        LOCALBOOT -1

LABEL CentOS7.2-x86_64
        kernel /images/CentOS7.2-x86_64/vmlinuz
        MENU DEFAULT
        MENU LABEL CentOS7.2-x86_64
        append initrd=/images/CentOS7.2-x86_64/initrd.img ksdevice=bootif lang=  text net.ifnames=0 biosdevname=0 kssendmac  ks=http://172.16.10.10/cblr/svc/op/ks/profile/CentOS7.2-x86_64
        ipappend 2

LABEL CentOS7.2.1-x86_64
        kernel /images/CentOS7.2-x86_64/vmlinuz
        MENU LABEL CentOS7.2.1-x86_64
        append initrd=/images/CentOS7.2-x86_64/initrd.img ksdevice=bootif lang=  kssendmac text  ks=http://172.16.10.10/cblr/svc/op/ks/profile/CentOS7.2.1-x86_64
        ipappend 2

MENU end
```

在开始安装之前，要确保该ks路径是有效的且kickstart内容是正确的。有时候提供的Kickstart内容错误了，在制作成profile的时候不会报错，但实际上浏览器访问该ks路径的内容提示错误。例如，访问CentOS7.2.1-x86_64这个LABEL的kickstart文件，将其ks文件url地址`http://172.16.10.10/cblr/svc/op/ks/profile/CentOS7.2.1-x86_64`输入浏览器中。如果得到如下结果，则表示出错了，很大的可能是cobbler不支持kickstart中的某指令，这个需要慢慢检查。

```
# This kickstart had errors that prevented it from being rendered correctly.
# The cobbler.log should have information relating to this failure.
```

修改kickstart文件后，需要重新编译profile加载新的kickstart文件。只需使用`cobbler profile edit --name=XXXXX --kickstart=YYYYY`即可重新编译XXXXX这个profile，或者执行cobbler sync命令。直到浏览器中能获取到kickstart的内容时才算成功。

或者，使用`cobbler profile getks --name=XXXXX`命令获取名为XXXXX的profile的ks内容。

总之，必须要保证能正确获取到ks内容。

### 开始安装

准备一个新的机器开机就会自动进入菜单，2-3秒超时后自动进行安装，安装完成后自动重启，重启时自动从本地启动。

所以，除了对新机器进行开机，其他的一切完完全全是全自动的。

建议在真正开始安装前，将dhcpd/rsyncd/tftp/cobbler等给重启一遍，防止中间改过哪些地方忘记重启而导致装机时出错。

## cobbler比pxe+kickstart好的地方

仅就cobbler基本功能而言，它跟pxe的能力基本是一样的，只是提供了更多花哨的功能。

但cobbler能够使用变量，能够通过几个命令自动完成文件复制，修改等繁琐的动作，另外它提供了api接口，常用的是它的图形界面。在这一点上，它还是不错的。

## 让新机器自动执行脚本

有些时候新机器上要进行很多配置，在kickstart的`%post`段也可以配置，但是这里能进行的配置是有限的。

可以在cobbler服务端写好要执行的脚本，然后在新机器上将脚本使用scp复制过去，但是scp复制需要确认和输入密码，所以需要在kickstart的选包部分指定安装expect包，然后使用expect进行非交互scp。

最后在`%post`段直接执行此脚本即可。



[1]: /linux/pxe_centos6