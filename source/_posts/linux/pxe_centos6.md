---
title: PXE+kickstart无人值守安装CentOS6
p: linux/pxe_centos6.md
date: 2019-07-10 12:29:41
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# PXE+kickstart无人值守安装CentOS6

PXE是Preboot Execution Environment的缩写，字面上的意思是开机前的执行环境。

要达成PXE必须要有两个环节：

- (1)客户端的网卡必须要支持PXE用户端功能，并且开机时选择从网卡启动，这样系统才会以网卡进入PXE客户端的程序；

- (2)PXE服务器必须要提供至少含有DHCP以及TFTP的服务！且其中：
  - DHCP服务必须要能够提供客户端的网络参数，还要告知客户端TFTP所在的位置；
  - TFTP则提供客户端的boot loader及kernel file下载路径。

还要加上NFS/FTP/HTTP(选择一样即可)等提供安装文件(安装镜像的解压文件)，才算是比较完整的PXE服务器。一般TFTP和DHCP服务都由同一台服务器提供，且大多数时候还提供NFS/FTP/HTTP服务，所以PXE服务器一般是提供3合一的服务。

## PXE流程

![](/img/linux/733013-20170225155539710-273351890.png)

- (1).**Client向PXE Server上的DHCP发送IP地址请求消息**，DHCP检测Client是否合法（主要是检测Client的网卡MAC地址），如果合法则返回Client的IP地址，同时将pxe环境下的Boot loader文件pxelinux.0的位置信息传送给Client。
- (2).**Client向PXE Server上的TFTP请求pxelinux.0**，TFTP接收到消息之后再向Client发送pxelinux.0大小信息，试探Client是否满意，当TFTP收到Client发回的同意大小信息之后，正式向Client发送pxelinux.0。
- (3).**Client执行接收到的pxelinux.0文件**。
- (4).**Client向TFTP请求pxelinux.cfg文件**(其实它是目录，里面放置的是是启动菜单，即grub的配置文件)，TFTP将配置文件发回Client，继而Client根据配置文件执行后续操作。
- (5).**Client向TFTP发送Linux内核请求信息**，TFTP接收到消息之后将内核文件发送给Client。
- (6).**Client向TFTP发送根文件请求信息**，TFTP接收到消息之后返回Linux根文件系统。
- (7).**Client加载Linux内核**（启动参数已经在4中的配置文件中设置好了）。
- (8).**Client通过nfs/ftp/http下载系统安装文件进行安装**。如果在(4)中的配置文件指定了kickstart路径，则会根据此文件自动应答安装系统。

## 部署环境说明

![](/img/linux/733013-20170811013326152-999090071.png)

## 部署DHCP

首先安装dhcp服务端程序。

```
yum -y install dhcp
```

DHCP主要是提供客户端网络参数与TFTP的位置，以及boot loader的文件名。同时，我们仅针对内网来告知TFTP的相关位置，所以可以编辑/etc/dhcp/dhcpd.conf在subnet的区块内加入两个参数即可。其中PXE上专门为PXE客户端下载的boot loader文件名称为pxelinux.0。

```
# vim /etc/dhcp/dhcpd.conf
ddns-update-style none;
default-lease-time 259200;
max-lease-time 518400;    
option routers 172.16.10.10;
option domain-name-servers 172.16.10.10;
subnet 172.16.10.0 netmask 255.255.255.0 {
        range 172.16.10.11 172.16.10.100;
        option subnet-mask 255.255.255.0;
        next-server 172.16.10.10; # 就是TFTP的位置
        filename "pxelinux.0";    # 告知得从TFTP根目录下载的boot loader文件名
}
```

重启dhcp。

```
service dhcpd restart
```

## 部署TFTP

从流程图中可以看出，boot loader文件pxelinux.0以及内核相关的配置文件(目录pxelinux.cfg下)主要都是由TFTP来提供的！

TFTP的安装很简单，直接使用yum即可。不过要告诉客户端TFTP的根目录在哪里，这样客户端才能找到相关文件。另外要注意，TFTP是由xinetd这个super daemon所管理的，因此设定好TFTP之后，要启动的是xinetd。

```
yum install tftp-server
```

默认TFTP服务的根目录是/var/lib/tftpboot/，为了少写些字母，将tftp的根目录修改为/tftpboot/。修改tftp的配置文件，主要是TFTP的根目录。

```
# vim /etc/xinetd.d/tftp
service tftp
{
socket_type = dgram
protocol = udp
wait = yes
user = root
server = /usr/sbin/in.tftpd
server_args = -s /tftpboot #重点在这里！修改tftp的根目录
disable = no
per_source = 11
cps = 100 2
flags = IPv4
}
```

创建tftp的根目录。
```
mkdir /tftpboot
```
启动TFTP并观察之：
```
# /etc/init.d/xinetd restart
# chkconfig xinetd on
# chkconfig tftp on

# netstat -tulnp | grep xinetd
Proto Recv-Q Send-Q Local Address Foreign Address State PID/Program name
udp 0 0 0.0.0.0: 69 0.0.0.0:* 2238/ xinetd
```

接下来的文件必须要放置于/tftpboot/目录下。

## 提供pxe的bootloader和相关配置文件

如果要使用PXE的开机引导的话，需要使用CentOS提供的syslinux包，从中copy两个文件到tftp的根目录/tftpboot下即可。整个过程如下：

```
$ yum -y install syslinux 
$ cp -a /usr/share/syslinux/{menu.c32,vesamenu.c32,pxelinux.0}  /tftpboot/
$ mkdir /tftpboot/pxelinux.cfg 

$ ls -l /tftpboot/
# 提供图形化菜单功能
-rw-r--r-- 1 root root  61796 Oct 16  2014 menu.c32      
# boot loader文件
-rw-r--r-- 1 root root  26759 Oct 16  2014 pxelinux.0    
# 开机的菜单设定在这里
drwxr-xr-x 2 root root   4096 Feb 24 20:02 pxelinux.cfg  
# 也是提供图形化菜单功能，但界面和menu.c32不同
-rw-r--r-- 1 root root 163728 Oct 16  2014 vesamenu.c32  
```

pxelinux.cfg是个目录，可以放置默认的开机选项，也可以针对不同的客户端主机提供不同的开机选项。一般来说，可以在pxelinux.cfg目录内建立一个名为default的文件来提供默认选项。

如果没有menu.c32或vesamenu.c32时，菜单会以纯文字模式一行一行显示。如果使用menu.c32或vesamenu.c32，就会有类似反白效果出现，此时可以使用上下键来选择选项，而不需要看着屏幕去输入数字键来选择开机选项。经过测试，使用vesamenu.c32比menu.c32更加好看些。

这部分设定完毕后，就是内核相关的设定了。


## 从安装镜像获取Linux内核文件

要安装Linux系统，必须提供内核文件，这里以64位版本的CentOS 6.6为例。

这里计划将内核相关文件放在/tftpboot/centos6.6/目录下。

既然要从安装镜像中获取内核相关文件，首先得要挂载镜像。

```
mount /dev/cdrom /test
mkdir /tftpboot/CentOS6.6
cp /test/isolinux/{vmlinuz,initrd.img} /tftpboot/CentOS6.6 
cp /test/isolinux/isolinux.cfg /tftpboot/pxelinux.cfg/default
```

其实仅需要vmlinuz和initrd.img两个文件即可，不过这里还将isolinux.cfg这个文件拷贝出来了，这个文件里提供了开机选项，可以以它作为修改开机选项和菜单的模板，这样修改起来比较容易，也更便捷！


## 选项设置

修改开机配置文件isolinux.cfg。由于拷贝它的时候重命名为default，所以修改default即可。修改的地方标红色了。
```
vim /tftpboot/default

default vesamenu.c32  # 这是必须项，或者改为menu.c32
#prompt 1
timeout 10
display ./centos6.6/boot.msg
#这是为选项提供一些说明的文件
menu background splash.jpg
menu title Welcome to CentOS 6.6!
menu color border 0 #ffffffff #00000000
menu color sel 7 #ffffffff #ff000000
menu color title 0 #ffffffff #00000000
menu color tabmsg 0 #ffffffff #00000000
menu color unsel 0 #ffffffff #00000000
menu color hotsel 0 #ff000000 #ffffffff
menu color hotkey 7 #ffffffff #ff000000
menu color scrollbar 0 #ffffffff #00000000
label linux
menu label ^Install your Linux
 menu default #设置默认的光标停留在此label上
 kernel ./centos6.6/vmlinuz
#设置内核文件，注意相对路径是从tftp的根路径/tftpboot开始的
#设置init ramdom disk文件，并设置启动时文本方式启动
append initrd=./centos6.6/initrd.img quiet 
label vesa
menu label Install system with ^basic video driver
kernel vmlinuz
append initrd=initrd.img xdriver=vesa nomodeset
label rescue
menu label ^Rescue installed system
kernel vmlinuz
append initrd=initrd.img rescue
label local
menu label Boot from ^local drive
localboot 0xffff
label memtest86
menu label ^Memory test
kernel memtest
append -
```

![](/img/linux/733013-20170225155545382-1132548593.png)

## 从网卡安装系统——开机测试

设置Bios从网卡启动。

![](/img/linux/733013-20170225155547179-1316072777.png)

![](/img/linux/733013-20170225155549273-2141647057.png)

![](/img/linux/733013-20170225155551116-1129749169.png)

![](/img/linux/733013-20170225155553523-2024111969.png)

由于到这里我还没有提供Linux的安装文件，所以选择URL从互联网来获取系统安装。

![](/img/linux/733013-20170225155556648-139650012.png)

由于要从互联网上获取系统安装文件，所以需要设置IP等网络参数，但要注意，这里的网络参数和前面设置的PXE网络参数是无关的，这里设置的IP仅是为了联上互联网。由于已经配置了DHCP，所以这里选择DHCP。

![](/img/linux/733013-20170225155559538-1694775741.png)

设置一个获取Linux系统的站点。上图设置的是163的站点。

如果没什么问题，到这里就开始进行安装直到完成了。以下是进度图片。

![](/img/linux/733013-20170225155601070-440360449.png)


## 通过http/ftp/nfs来提供系统安装文件

现在在本地服务器上安装http或ftp或nfs来作为系统文件的来源。

首先挂载Linux的镜像光盘（前文已经挂载过了），假设挂载到/mnt目录上。
```
mount /dev/cdrom /mnt
```
注意，要提供的是镜像中的所有文件，而不是简单的提供一个镜像。所以将/mnt中的所有文件复制出来，假设复制到目录/install目录下。
```
mkdir /install
cp -a /mnt/* /install
```
其实也可以不用复制出来的，只需要将镜像挂载到某个目录下，只要nfs/http/ftp能够找到它就行了。

(1). 使用NFS提供安装文件
```
yum -y install rpcbind nfs-utils
```
启动rpcbind和nfs。
```
service rpcbind start
service nfs start
```
然后导出/install目录给需要安装系统的客户端，这里导出给整个网段。
```
$ exportfs -o ro,async,no_root_squash 192.168.100.0/24:/install
$ showmount -e

Export list for node1.longshuai.com:
/install 192.168.0.0/24
```

(2). 使用http提供安装文件

安装httpd。
```
yum -y install httpd
service httpd start
```
由于http的`DocumentRoot "/var/www/html"`，所以系统的安装文件需要在此目录下或其子目录才能找到，假设在/var/www/html/centos6.6目录下，只需要简单的将镜像挂载到此目录即可。
```
mkdir /var/www/html/centos6.6
mount /dev/cdrom /var/www/html/centos6.6
```

(3). 使用vsftpd来提供安装文件
```
yum -y install vsftpd
```
由于这里仅用来提供下系统的安装文件，所以就没必要对vsftpd多多配置了，使用它最简单的匿名用户模式即可，但是匿名用户的根目录为/var/ftp，所以要将镜像挂载到此目录或此目录下的子目录下，假设放在/var/ftp/centos6.6。
```
mkdir /var/ftp/centos6.6
mount /dev/cdrom /var/ftp/centos6.6
```

(4). 测试并填写安装文件的路径地址

到此，就可以启动虚拟机来测试了。和前面的一样，直到下面这里。

![](/img/linux/733013-20170225155603601-1710848770.png)

对于ftp和http，直接填写即可。
```
ftp://192.168.100.100/centos6.6
http://192.168.100.100/centos6.6
```

对于NFS写这样的路径，因为在上面NFS的设定上是导出了/install目录，安装文件也是复制到此文件中的。

![](/img/linux/733013-20170225155606320-1889163293.png)

然后就会进入安装画面，但是这样还是有些地方需要手动指定的。无法实现非交互时无人值守的方式安装。

所以下文就介绍kickstart实现无人值守的方式。

## kickstart+PXE无人值守大量部署Linux

所谓的无人值守，就是自动应答，当安装过程中需要人机交互提供某些选项的答案时（如如何分区），自动应答文件可以根据对应项自动提供答案。但是，无人值守并不完全是无人值守，在设置bios从网卡启动是必须人为设置的，且安装完系统后设置不从网卡启动也是需要人为设置的。此处之外，其他的都可以无人值守。

要配置无人值守的系统安装，需要提供安装过程中需要的各种选择，这些选择在kickstart的配置文件中，一般正常安装完Linux系统在root用户的家目录下有一个anaconda-ks.cfg，该文件的配置说明见[kickstart文件详解][1]。以下是该文件中的部分内容。

![](/img/linux/733013-20170225155608554-1345052732.png)

不难发现，装系统时很多选项在这里面都记录了。

那么，要使用kickstart来批量部署操作系统，就需要提供该文件。以下是我提供的配置文件/install/ks.cfg(因为我是使用NFS作为文件提供源的，所以我将其放在nfs的导出目录中，让客户端能够找到)。其中rootpw的加密密码要使用grub-crypt生成。
```
$ vim /install/ks.cfg

install
text
nfs --server=192.168.100.100 --dir=/install

#url --url=http://192.168.100.100/centos6.6
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto quiet"
lang en_US.UTF-8
keyboard us
network --onboot no --device eth0 --bootproto dhcp --noipv6
rootpw --iscrypted 
$6$x4u9sIfSQsO7ddk5$/.0Xe6tFBY0uUmFFtyvAeY9YVPtcn8zl21fFNgmAoYtepQHRYDthQ4T1ZE12kDfAT6O3oXfRb7uv214t3Bb3K1
firewall --service=ssh
authconfig --enableshadow --passalgo=sha512
selinux --disabled
timezone Asia/Shanghai
reboot #安装结束后重启

#make partitions
zerombr
clearpart --all --initlabel
part /boot --fstype=ext4 --asprimary --size=250
part / --fstype=ext4 --asprimary --grow --size=2000
part swap --fstype=swap --size=2000

%packages
@base
@core
@debugging
@development
@dial-up
@hardware-monitoring
@performance
@server-policy
@workstation-policy
sgpio
device-mapper-persistent-data
systemtap-client

%post #结束后做的事

cat >>etc/yum.repos.d/base.repo<<eof
[base]
name=163repo
baseurl=http://mirrors.163.com/centos/6/os/x86_64/
gpgcheck=0
enable=1
eof

#设置网卡为启动
sed "s/ONBOOT.*$/ONBOOT=yes/" /etc/sysconfig/network-scripts/ifcfg-eth0      
#设置启动系统时不使用图形进度条方式
sed "s/rhgb //" /boot/grub/grub.conf   
#设置主机名
sed "s/HOSTNAME=.*$/HOSTNAME=xuexi.longshuai.com/" /etc/sysconfig/network    
%end
```

然后修改defalut文件，让客户端能够找到ks.cfg文件。



```
$ vim /tftpboot/pxelinux.cfg/default
label linux
menu label ^Install your Linux
menu default
kernel ./centos6.6/vmlinuz
append initrd=./centos6.6/initrd.img ks=nfs:192.168.100.100:/install/ks.cfg quiet
```

如果要使用LVM的分区方式，参考如下：

```
part /boot --fstype ext4 --size=100
part swap --fstype=swap --size=2048
part pv26 --size=100 --grow
volgroup VG00 --pesize=32768 pv26
logvol / --fstype ext4 --name=LVroot --vgname=VG00 --size=29984
logvol /data --fstype ext4 --name=LVdata --vgname=VG00 --size=100 --grow
```

如果觉得使用样本的方式手工写配置文件比较麻烦，也可以使用图形化工具来制作ks.cfg文件。在linux中用yum安装system-config-kickstart就行了（图形化依赖于x-window），选项也有些限制（比如分区不能使用lvm）。

然后找台机器从网卡启动就进入安装模式了。

因为在ks.cfg中设置了安装完成后reboot，所以要手动去修改bios不要再从网卡启动，否则重启后又再次从网卡启动然后又去自动应答装系统了。当然，可以将reboot换成shutdown或者poweroff，这样装完就只是关机了，等开机前人为设置不从网卡启动。



[1]: /linux/kickstart_config	"kickstart配置文件详解"