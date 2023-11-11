---
title: kickstart文件详解
p: linux/kickstart_config.md
date: 2019-07-10 12:29:43
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# kickstart文件详解

kickstart自动应答文件选项非常多，以下只说明CentOS 6下几个常用的可能用到的选项。另外，CentOS 6和CentOS 7的选项有不小区别，所以请注意使用，可以查看官方安装文档。

[CentOS6的Installation向导](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/pdf/Installation_Guide/Red_Hat_Enterprise_Linux-6-Installation_Guide-en-US.pdf)

[CentOS7的Installation向导](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/pdf/Installation_Guide/Red_Hat_Enterprise_Linux-7-Installation_Guide-en-US.pdf)

以下是CentOS 6上kickstart选项说明：在最后还给出了一个kickstart文件的示例。

文件由三部分组成

- 一是选项指令段，用于自动应答图形界面安装时除包选择外的所有手动操作

- 二是package选择段，使用`%packages`引导该功能

- 三是脚本段，该段可有可无，分为两种：
  - (1)`%pre`预安装脚本段，在安装系统之前就执行的脚本，该段很少使用，因为可用的命令太少
  - (2)`%post`后安装脚本段，在系统安装完成后执行的脚本

kickstart选项指令段的说明。

【必须的选项】：
```
1.auth或者authconfig: 验证选项

--useshadow或者--enableshadow启用shadow文件来验证
--passalgo=sha512使用sha512算法
    
2.bootloader: 指定如何安装引导程序，要求必须已选择分区、已选择引导程序、已选择软件包，
              如果没选择将会停止而不会询问
--location=mbr 指定引导程序的位置，默认为mbr，还可以指定none
               或者包含bootloader的引导块所在分区
--driveorder=sda 指定grub安装在哪个分区以及指定寻找顺序，
                 例如 --driverorder=sda sdc sdb
--append="crashkernel=auto rhgb quiet" 指定内核参数

3.keyboard：指定键盘类型，一般使用美式键盘"keyboard us"，
            新版的kickstart的格式有所变化，但也支持"keyboard us"这样的老格式

4.lang：指定语言，如"lang en_US.UTF-8"5.rootpw：设置root用户的密码

--iscrypted:使用加密密码，可以使用MD5,SHA-256,sha-512等。
            如：rootpw  --iscrypted $6$kxEBpy0HqHiY2Tsx$xxxxxxxxxx.
            其中SHA-512位的加密密码先尝试使用openssl来生成，
            如果openssl命令版本较老不支持，那么：
            - 在CentOS 6上可以使用"grub-crypt --sha-512"生成，
            - CentOS7上可以使用python等工具来生成，如下：
            python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
```

可选的选项：
```
1.selinux：设置selinux，值为enforcing,permissice,disable之一
2.autostep: 交互式，和interactive类似
3.interactive: 使用kickstart文件指定的参数交互式安装，但仍会给出每一步的选择项，如果直接下一步就使用kickstart参数
4.cmdline：在完全非交互的命令行模式下进行安装
5.driverdisk：指定驱动程序所在位置
    drvierdisk --source=
6.firewall：设置firewall
    --disable禁用防火墙
7.firstboot：
    --disable：安装后第一次启动默认会给出很多需要手动配置的界面，禁用它
8.graphical：在图形模式下根据kickstart执行安装，默认该选项
9.text：文本模式下根据kickstart执行安装
        既然使用kickstart了，当然建议选择使用纯文本模式而不是图形模式了
10.halt/reboot：安装完成后关机还是reboot，默认是halt
11.ignoredisk：指定忽略的磁盘
12.install/upgrade：指定是安装还是升级系统
    对于install，还必须指定下面几种安装方式之一：
        cdrom：指定从第一个光盘驱动器安装
        harddrive：指定从本地硬盘安装，要求硬盘必须是vfat或者ext2文件系统格式
            --biospart：指定从bios类型的分区来安装，如82文件系统类型号的分区
            --partition：从某个分区安装
            --dir：指定从包含install-tree（安装树）的目录安装
                例如：harddrive --partition=hdb2 --dir=/tmp/install-tree
        nfs：指定从nfs路径安装
            --server:指定nfs服务器主机名或IP
            --dir:指定包含install-tree的目录
            --opts:指定挂载NFS的mount选项
            如：nfs --server=172.16.10.10 --dir=/export_path
        url：指定从ftp、http、https安装
             例如：url --url ftp://172.16.10.10
13.loggin：指定安装过程中的错误日志位置
    --host:指定日志将发送到那台主机上
    --port:如果远程主机的rsyslog使用非默认端口，则应该指定该端口选项
    --levle:指定日志级别
14.network：为系统配置网络信息，并在安装过程中激活该网络设备。
           可多次使用network指令，例如既设置网络，又设置主机名
    --bootproto:dhcp或static；对于static则必须指定IP地址、子网掩码、网关和DNS
    --device：网卡名，可以使用eth0类似的名称来指定
    --hostname:指定主机名
    --onboot：是否在引导系统时启用指定的设备
        如：
        network --bootproto=static --ip=192.168.100.2 --netmask=255.255.255.0 --gateway=192.168.100.254 --nameserver=8.8.8.8
        network --bootproto=dhcp --device=eth0 --noipv6
        network --hostname=node1.xuexi.com
15.autopart: 自动创建几个分区：大于1G的根分区，250M的boot分区和swap分区
16.zerombr：清除磁盘的mbr
  
17.clearpart: 在安装系统前清除分区，如果指定该选项则必须指定正确
    --all:清除所有分区
    --Linux：清除Linux分区
    --none：不清除分区
    --initlabel：创建标签，对于没有MBR或者GPT的新硬盘，该选项是必须的
    --drivers=sdb：清除指定的分区
    所以，clearpart --all --initlabel是常见的方式
18.part：创建分区
--asprimary:强制指定为主分区
--grow：使用所有可用空间，即为其分配所有剩余空间。
        对于根分区至少需要3G空间（即使是--grow，也还是需要指定--size）
--ondisk：指定在哪块磁盘上创建分区。如果有多块磁盘，则需要指定
          在哪块磁盘上创建哪个分区，只有一块硬盘时可以省略该选项
          如：
            #boot分区200-250M足以，但应多给一些留给未来可能的内核升级
            #part /boot --fstype=ext4 --asprimary --size=600 
            #part swap --fstype=swap --asprimary --size=2048
            #part / --fstype=ext4 --grow --asprimary --size=2000
LVM的分区方法：
part /boot --fstype ext4 --size=600
part swap --fstype=swap --size=2048
part pv26 --size=100 --grow
volgroup VG00 --pesize=32768 pv26
logvol / --fstype ext4 --name=LVroot --vgname=VG00 --size=29984
logvol /data --fstype ext4 --name=LVdata --vgname=VG00 --size=100 --grow        
            
19.repo：指定除自带的yum源外的其他yum源，可以指定多行yum源
        （既然是第一次装系统，基本都不会去加这项）
         如：repo --name="CentOS" --baseurl=cdrom:sr0 --cost=100
20.services：设置默认运行级别下开机自启动的服务
    --disable
    --enable
        disable先处理enable后处理
        如services --disable auditd,cups,atd
21.timezone：指定时区
    如：Asia/Shanghai
22.user：在系统中生成一个新用户
    --name：指定用户名
    --groups：指定辅助组，非默认组
    --homedir：用户家目录，如果不指定则默认为/home/<username>
    --password：该用户的密码，如果不指定或省略则创建后该用户处于锁定状态
    --shell：用户的shell，不指定则默认
    --uid：用户UID，不指定则自动分配一个非系统用户的UID
23.key：输入序列号，只在redhat中有，CentOS系统没有该项
    --skip  跳过key选项
```

kickstart软件包或包组选项：使用`%packages`表示该段内容,\@表示选择的包组，最前面使用横杠表示取反，即不选择的包或包组。\@base和\@core两个包组总是被默认选择，所以不必在`%packages`中指定它们

```
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
lrzsz
tree

%end
```

 以下是CentOS 6.6下的ks文件示例。

```
install
text
nfs --server=192.168.100.100 --dir=/install
#url --url=http://192.168.100.100/centos6.6
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto quiet"
lang en_US.UTF-8
keyboard us
network --onboot=yes --device=eth0 --bootproto=dhcp --noipv6
rootpw  --iscrypted $6$x4u9sIfSQsO7ddk5$/密码数据
firewall --service=ssh
authconfig --enableshadow --passalgo=sha512
selinux --disabled
timezone Asia/Shanghai
reboot       #安装结束后重启

#make partitions
zerombr
clearpart --all --initlabel
part    /boot   --fstype=ext4   --asprimary     --size=250
part    /       --fstype=ext4   --asprimary     --grow     --size=2000
part    swap    --fstype=swap   --size=2000

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
%end

%post     #结束后做的事
cat >>/etc/yum.repos.d/base.repo<<eof
[base]
name=sohu
baseurl=http://mirrors.sohu.com/centos/$releasever/os/$basearch/
gpgcheck=0
enable=1
[epel]
name=epel
baseurl=http://mirrors.sohu.com/fedora-epel/6Server/x86_64/
enable=1
gpgcheck=0
eof

#设置网卡为启动
sed -i "s/ONBOOT.*$/ONBOOT=yes/" /etc/sysconfig/network-scripts/ifcfg-eth0
# 设置启动系统时不使用图形进度条方式
sed -i "s/rhgb //" /boot/grub/grub.conf
#设置主机名
sed -i "s/HOSTNAME=.*$/HOSTNAME=xuexi.longshuai.com/" /etc/sysconfig/network

%end
```