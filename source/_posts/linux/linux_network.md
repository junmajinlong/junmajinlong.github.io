---
title: Linux网络管理
p: linux/linux_network.md
date: 2019-07-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux网络管理

在解释Linux网络管理相关的内容时，我觉得有必要先解释数据包转发功能以及因此涉及到的路由决策，这对厘清路由和未来的防火墙有重要帮助。然后再介绍网络配置命令和相关文件。

## Linux处理数据包过程 

**当向外界主机发送数据时，在它从网卡流入后需要对它做路由决策，根据其目标决定是流入本机数据还是转发给其他主机，如果是流入本机的数据，则数据会从内核空间进入用户空间(被应用程序接收、处理)。当用户空间响应(应用程序生成新的数据包)时，响应数据包是本机产生的新数据，在响应包流出之前，需要做路由决策，根据目标决定从哪个网卡流出。如果不是流入本机的，而是要转发给其他主机的，则必然涉及到另一个流出网卡，此时数据包必须从流入网卡完整地转发给流出网卡，这要求Linux主机能够完成这样的转发。但Linux主机默认未开启`ip_forward`功能，这使得数据包无法转发而被丢弃。Linux主机和路由器不同，路由器本身就是为了转发数据包，所以路由器内部默认就能在不同网卡间转发数据包，而Linux主机默认则不能转发**。

如下图：

![](/img/linux/1699272508044.png)

另外，IP地址是属于内核的(不仅如此，整个tcp/ip协议栈都属于内核，包括端口号)，只要能和其中一个地址通信，就能和另一个地址通信(这么说是不准确的，即使地址属于内核，但还存在一个检查数据包是否丢弃的问题，不过这不是本文内容)，而不管是否开启了数据包转发功能。例如某Linux主机有两网卡eth0:172.16.10.5和eth1:192.168.100.20，某192.168.100.22主机网关指向192.168.100.20，若它ping 172.16.10.5，结果将是通的，因为地址属于内核，从eth1进来的数据包被内核分析时，发现目标地址为本机地址，直接就回应192.168.100.22，回应数据包继续从eth1出去。

如果Linux主机有多块网卡，如果不开启数据包转发功能，则这些网卡之间是无法互通的。例如eth0是172.16.10.0/24网段，而eth1是192.168.100.0/24网段，到达该Linux主机的数据包无法从eth0交给eth1或者从eth1交给eth0，除非Linux主机开启了数据包转发功能。

在Linux上开启转发功能有多种方法：

```
shell> echo 1 > /proc/sys/net/ipv4/ip_forward
shell> sysctl -w net.ipv4.ip_forward=1
```

以上两种方法是临时生效的，若要永久生效，则应该写入配置文件。在CentOS 6中，将`/etc/sysctl.conf`文件中的`net.ipv4.ip\_forward`值改为1即可，但在CentOS 7中，systemd管理了太多的功能，sysctl的配置文件也分化为多个，包括`/etc/sysctl.conf、/etc/sysctl.d/\*.conf和/usr/lib/sysctl.d/\*.conf`，并且这些文件中默认都没有`net.ipv4.ip\_forward`项。当然，直接将此项写入到这些配置文件中也都是可以的，建议写在`/etc/sysctl.d/\*.conf`中，这是systemd提供自定义内核修改项的目录。例如：

```
shell> echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/ip_forward.conf
```

可以使用以下几种方式查看是否开启了转发功能。

```
[root@xuexi ~]# sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0

[root@xuexi ~]# cat /proc/sys/net/ipv4/ip_forward
0

[root@xuexi ~]# sysctl -a | grep ip_forward
net.ipv4.ip_forward = 0
net.ipv4.ip_forward_use_pmtu = 0
```

## 和网络相关的几个文件说明 

### 网卡配置文件ifcfg-\* 

在`/etc/sysconfig/network-scripts/`目录下有不少文件，绝大部分都是脚本类的文件，但有一类ifcfg开头的文件为网卡配置文件(interface config)，所有ifcfg开头的文件在启动网络服务的时候都会被加载读取，但具体的文件名ifcfg-XX的XX可以随意命名。

以下是一个(CentOS 7上)ifcfg-XX文件的内容示例。

```
[root@xuexi ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
# 显示的名称，必须/sys/class/net/目录下的某个网卡名相同
DEVICE="eth0"      
IPV6INIT="no" 
BOOTPROTO="dhcp"
ONBOOT=yes 
TYPE="Ethernet"
DEFROUTE="yes"

# 设置为yes时，此文件设置的DNS将覆盖/etc/resolv.conf，
# 若开启了DHCP，则默认为yes，所以dhcp的dns也会覆盖/etc/resolv.conf
PEERDNS="yes"      
                   
PEERROUTES="yes"
IPV4_FAILURE_FATAL="no"
NAME="System eth0"
DNS1=114.114.114.114
DNS2=8.8.8.8
DNS3=114.114.115.115
```

<a name="blog322"></a>

### DNS配置文件/etc/resolv.conf 

该文件用于设置DNS指向，以及解析顺序。该文件格式如下：

```
# 声明本地域名，即解析时自动隐式补齐的域名
domain  domain_name
# 指定域名搜索顺序(最多6个)，和domain不能共存，若共存了，则后面的行生效
search  domain_name_list
# 设置DNS指向，最多3个
nameserver  IP1
nameserver  IP2
nameserver  IP3
# 指定解析超时时间(默认5秒)和解析次数(默认2次)
options timeout:n attempts:n
```

例如将`/etc/resolv.conf`设置为下所示，为了测试，暂且不设置nameserver。

```
domain malong.com
```

当解析不带点`.`的主机名时，如`www`，认为不是fqdn，将自动加上`.malong.com`变成解析`www.malong.com`。

```
[root@xuexi ~]# host -a www
Trying "www.malong.com"
;; connection timed out; trying next origin
Trying "www"
;; connection timed out; no servers could be reached
```

当解析的名称末尾不带点但中间带了点的，如`www.host`，认为是fqdn，将直接解析`www.host`，解析完这个后再解析加上`malong.com`的名称，即再解析`www.host.malong.com`。

```
[root@xuexi ~]# host -a www.host
Trying "www.host"
;; connection timed out; trying next origin
Trying "www.host.malong.com"
;; connection timed out; no servers could be reached
```

当解析末尾带点的名称时，如`www.host.`认为是完整的fqdn，将直接解析`www.host`,解析完后直接结束解析，不会再补齐本地域名再解析。

```
[root@xuexi ~]# host -a www.host.
Trying "www.host"
;; connection timed out; trying next origin
Trying "www.host"   # 默认解析两次
;; connection timed out; no servers could be reached
```

search关键字的作用和domain是一样的，只不过search同时还暗含域名搜索的顺序。例如设置search为如下内容：

```
search  malongshuai.com longshuai.com mashuai.com
```

此时若解析`www.host`，将依次解析`www.host`,`www.host.malongshuai.com`，`www.host.longshuai.com`，`www.host.mashuai.com`。

```
[root@xuexi ~]# host -a www.host
Trying "www.host"
;; connection timed out; trying next origin
Trying "www.host.malongshuai.com"
;; connection timed out; trying next origin
Trying "www.host.longshuai.com"
;; connection timed out; trying next origin
Trying "www.host.mashuai.com"
;; connection timed out; no servers could be reached
```

```
[root@xuexi ~]# host -a www
Trying "www.malongshuai.com"
;; connection timed out; trying next origin
Trying "www.longshuai.com"
;; connection timed out; trying next origin
Trying "www.mashuai.com"
;; connection timed out; trying next origin
Trying "www"
;; connection timed out; no servers could be reached
```

domain部分和search部分不能共存，如果共存了，则后出现的行有效。

### /etc/udev/rules.d/70-persistent-net.rules 

当插入新的网络设备时，内核首先识别到，随后在sysfs文件系统(一般挂载在/sys下)中生成该设备对应的信息文件。然后内核通知udev的后台守护进程udevd(若不知道它是什么东西，请认为它是Windows系统中的设备管理器，管理和监视硬件设备)，udevd将读取sysfs中对应设备的相关信息，并比对或生成udev的规则集，能匹配上的则做对应的操作。对于网卡来说，CentOS 6上它的的规则集文件默认为`/etc/udev/rules.d/70-persistent-net.rules`，匹配该规则集成功后，最后还在`/sys/class/net`目录中生成对应的设备子目录。

以下为两个网卡的规则集的内容：

```
[root@xuexi ~]# cat /etc/udev/rules.d/70-persistent-net.rules 

# PCI device 0x8086:0x100f (e1000)
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="00:0c:29:7f:cf:a4", ATTR{type}=="1", KERNEL=="eth*", NAME="eth0"

# PCI device 0x8086:0x100f (e1000)
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="00:0c:29:7f:cf:ae", ATTR{type}=="1", KERNEL=="eth*", NAME="eth1"
```

具体的udev规则集语法并非本文内容，所以此处仅简单解释下上面的两个规则集。规则集文件的写法都是key/value格式，但分为匹配key/vaule和行为key/value。上述例子中，从SUBSYSTEM直到KERNEL都是使用`==`号，表示匹配key/value，最后一个NAME使用单`=`号，表示赋值key/value。所以上述文件的意思是：当/sys中的某设备各信息都能匹配上述某条规则，则赋值该设备名称为eth0或eth1，/sys/class/net目录下也由此名称命名设备信息的目录，`ifcfg-*`配置文件中的DEVICE的值必须和它们相同。

注意，网络规则集文件会由内核检测到设备时自动生成或写入，因此清空它不会有任何影响。

克隆虚拟机时，总是会出现MAC地址冲突，这是因为规则集文件和ifcfg配置文件都被克隆了，而新克隆出来的机器中MAC地址又是新的，所以会生成新的规则集，但克隆过来的ifcfg配置文件中的DEVICE值和该规则对应不上，导致克隆主机的网络将启动不了。解决办法是清空该文件，然后重启克隆主机，这样内核将新生成对应新MAC地址的规则集文件。当然，在克隆前清空模板主机的规则集文件，然后再克隆也是可以的。

值得一提的是，在CentOS 7中，systemd已经将udevd的功能整合在了一起，udev的规则集文件几乎都放到了/usr/lib/udev/rules.d/目录下，但却更方便了，克隆CentOS 7主机时，根本就不会出现MAC地址冲突的可能。不过，显式书写在/etc/udev/rules.d/目录下的规则集文件仍然是生效的。

### /etc/services 

该文件中记录的是端口和服务的对应关系。

```
[root@xuexi ~]# grep '^ftp\|^ssh' /etc/services 
ftp-data        20/tcp
ftp-data        20/udp
ftp             21/tcp
ftp             21/udp          fsp fspd
ssh             22/tcp                          # The Secure Shell (SSH) Protocol
ssh             22/udp                          # The Secure Shell (SSH) Protocol
ftp-data        20/sctp                 # FTP
ftp             21/sctp                 # FTP
ssh             22/sctp                 # SSH
ftp-agent       574/tcp                 # FTP Software Agent System
ftp-agent       574/udp                 # FTP Software Agent System
sshell          614/tcp                 # SSLshell
sshell          614/udp                 #       SSLshell
ftps-data       989/tcp                 # ftp protocol, data, over TLS/SSL
ftps-data       989/udp                 # ftp protocol, data, over TLS/SSL
ftps            990/tcp                 # ftp protocol, control, over TLS/SSL
ftps            990/udp                 # ftp protocol, control, over TLS/SSL
ssh-mgmt        17235/tcp               # SSH Tectia Manager
ssh-mgmt        17235/udp               # SSH Tectia Manager
```

## 网络接口配置和主机名 

### ifconfig 

该命令虽然在man文档中被说明已废弃，但大众显然无法忘记它。ifconfig命令是一个接口配置命令，但更多的被用来显示已激活的网络接口信息。

```
ifconfig [ interface | -a ]
ifconfig interface options

选项说明：
interface：指定被操作的网络接口名，如eth0
up       ：激活指定的网络接口，如果在命令行中为网络接口分配了IP地址，则默认会up
down     ：将指定的接口设置为down状态
[-]arp   ：启用或禁用该接口上使用ARP协议，如"ifconfig eth0 -arp"
mtu N    ：设置指定接口的最大传输单元(MTU)
netmask  ：设置该接口的IP netmask，默认会采用A/B/C类地址的掩码位数
address  ：要分配给该接口的IP地址
```

ifconfig示例：

```
# 添加IP地址
# 也可使用CIDR格式掩码
[root@xuexi ~]# ifconfig eth0:1 192.168.100.20 netmask 255.255.255.0 up
[root@xuexi ~]# ifconfig eth0:1 192.168.100.20/24 up

[root@xuexi ~]# ifconfig eth1 up       # 激活该网络接口
[root@xuexi ~]# ifconfig eth1 down     # 临时down掉eth1接口
[root@xuexi ~]# ifconfig eth1 -arp     # 抑制eth1上的arp
[root@xuexi ~]# ifconfig eth1 arp      # 启用eth1上的arp
```

需要注意的是，ifconfig所有的配置都是应用于内核的，所以只会临时生效，重启网络服务后会立即失效。

对于slave地址，即别名地址，若要永久生效，应该建立对应的别名接口配置文件，如`/ets/sysconfig/network-scripts/ifcfg-eth0:0`，然后在该文件中的DEVICE关键字上给定eth0:0名称，该DEVICE项必须配置正确。

### ifcfg

用法很简单。

```
ifcfg DEV [[add|del [ADDR[/LEN]] | stop]
       add - add new address
       del - delete address
       stop - completely disable IP
```

例如：

```
[root@xuexi ~]# ifcfg eth1:0 add 192.168.100.20/24   # 添加一个地址
[root@xuexi ~]# ifcfg eth1:0 del 192.168.100.20      # 删除一个地址
[root@xuexi ~]# ifcfg eth1 stop      # 临时禁用eth1
```

###  hostname命令 

用于设置主机名，但也有几个其它好用的功能。

```
hostname [-I] [-f] [-d] [-s] [hostname]

选项说明：
-I         ：获取该主机上所有非环回IP地址，该选项不依赖于主机名解析
-f,--fqdn  ：获取fqdn
-d,--domain：获取fqdn的域名部分，等价于命令dnsdomainname
-s,--short ：获取fqdn的主机名部分，严格地说是获取第一个"."前的部分，例如"www.baidu.com"将获取为"www"
```

使用-I选项可以直接获取该主机上的所有IP地址，包括别名地址，这在某些时候太方便了。

```
[root@xuexi ~]# hostname -I
192.168.100.54 172.16.10.10
```

hostname修改的主机名为临时生效，它修改的其实是/proc/sys/kernel/hostname文件。

```
[root@xuexi ~]# cat /proc/sys/kernel/hostname
xuexi.longshuai.com
```

虽然在man文档中说有个永久有效的选项(-b)，但测试时却毫无效果。要想永久生效，需要修改配置文件/etc/hostname(CentOS 7)或/etc/sysconfig/network(CentOS 6)。例如在CentOS 7上：

```
[root@xuexi ~]# echo "ma.longshuai.com" >/etc/hostname
```

## 网关/路由 

Linux上分为3种路由：

 *  主机路由：直接指明到某台具体的主机怎么走，主机路由也就是所谓的静态路由
 *  网络路由：指明某类网络怎么走
 *  默认路由：不走主机路由的和网络路由的就走默认路由。操作系统上设置的默认路由一般也称为网关。

若Linux上到某主机有多条路由可以选择，这时候会挑选优先级高的路由。在Linux中，路由条目的优先级确定方式是先匹配掩码位长度，再比较管理距离(比如metric)。也就是说，掩码位长的路由条目优先级一定比掩码位短的优先级高，所以主机路由的优先级最高，然后是直连网络(即同网段)的路由(也算是网络路由)次之，再是网络路由，最后才是默认路由。若路由条目的掩码长度相同，则比较节点之间的管理距离，管理距离短的生效。

例如下面的路由表中，若ping 192.168.5.20，则先比对192.168.100.78发现无法匹配，然后比对192.168.100.0，发现也无法匹配，接着再匹配192.168.0.0这条网络路由条目，发现能匹配，所以选择该路由条目。

```
[root@xuexi ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.100.2   0.0.0.0         UG    100    0        0 eth0
172.16.10.0     0.0.0.0         255.255.255.0   U     100    0        0 eth1
192.168.0.0     192.168.100.70  255.255.0.0     UG    0      0        0 eth0
192.168.100.0   0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.100.78  0.0.0.0         255.255.255.255 UH    0      0        0 eth0
```

再比如下面的路由表。由于两块网卡eth0和eth1都是192.168.100.0/24网段地址，所以它们的路由条目在掩码长度的匹配上是相同的，但是和eth0直连的网段主机通信时，肯定会选择eth0这条路由条目，因为eth1和该网段主机隔了一个eth0，距离增加了1。

```
[root@xuexi ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.100.2   0.0.0.0         UG    100    0        0 eth0
192.168.100.0   0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.100.0   0.0.0.0         255.255.255.0   U     101    0        0 eth1
```

### route命令 

route命令用于显示和管理路由表。当使用了add或del选项时，route命令将设置路由条目，否则route命令将显示路由表。

要显示路由表信息，只需简单的`route -n`即可，其中`-n`选项表示不解析主机名。

例如：

```
[root@xuexi ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.100.2   0.0.0.0         UG    100    0        0 eth0
172.16.10.0     0.0.0.0         255.255.255.0   U     100    0        0 eth1
192.168.0.0     192.168.100.70  255.255.0.0     UG    0      0        0 eth0
192.168.100.0   0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.100.78  0.0.0.0         255.255.255.255 UH    0      0        0 eth0
```

对于CentOS 6以上的系统，请忽略Metric和Ref两列，它们已经不被内核使用，只是有些路由软件可能会用上。

对于Flags列，如果没有安装路由软件，则只可能出现下面的3种值：

```
U (route is up)
H (target is a host)
G (use gateway，也即是设置了下一跳的路由条目)
```

若要管理路由表，则使用add或del选项。

```
route [add/del] [-host/-net/default] [address[/mask]] [netmask] [gw] [dev]

选项说明：
add/del：增加或删除路由条目
-net：增加或删除的是一条网络路由
-host：增加或删除的是一条主机路由
default：增加或删除的是一条默认路由
netmask：明确使用netmask关键字指定掩码，要可以不使用该选项直接在地址上使用cidr格式的掩码，即IP/MASK。
gw：指定下一跳的地址。要求下一跳地址必须是能到达的，且一般是和本网段直连的接口。
dev：强制将路由条目关联到指定的接口上。一般内核会自动判断路由条目应该关联到哪个网络接口。
```

例如：

(1).添加和删除默认路由

```
shell> route add default gw 192.168.100.10
shell> route del default
# 若有多条默认路由，则再加上gw即可唯一删除指定条目
shell> route del default gw 192.168.100.10   
```

因为默认路由的目的地是0.0.0.0，所以操作默认路由也可以使用0.0.0.0替代default关键字，但这样就麻烦的多了。

(2).添加和删除网络路由

```
shell> route add -net 172.16.10.0/24 gw 192.168.100.70
shell> route add -net 172.16.10.0 netmask 255.255.255.0 gw 192.168.100.70
```

若实在不知道下一跳给谁，那么指定本机接口也是可以的。

```
shell> route add -net 172.16.10.0/24 dev eth0
```

删除路由可以直接在增加路由的语句上将add改为del关键字。如

```
shell> route del -net 172.16.10.0/24 gw 192.168.100.70
shell> route del -net 172.16.10.0 netmask 255.255.255.0 gw 192.168.100.70
shell> route del -net 172.16.10.0/24 dev eth0
```

但大多数时候，可以偷懒，只要能唯一确定删除的是哪条路由即可。如：

```
shell> route del -net 172.16.10.0/24
```

(3)添加和删除主机路由

```
shell> route add -host 172.16.10.55 gw 192.168.10.20
shell> route del -host 172.16.100.55
```

### 配置永久路由 

根据接口创建路由配置文件/etc/syconfig/network-scripts/route-ethX，要从那个接口出去X就是几。

路由配置文件的配置格式非常简单，每一行一个路由条目，先是要到达的目标，然后是via关键字，最后是下一跳地址。要求下一跳必须能到达，且一般都和ethX同网段。

> DEST via nexthop

例如eth0网卡的IP地址是192.168.10.123，要通过网卡eth0出去到达10.0.0.10，那么下一跳的地址要和eth0的地址在同网段，如192.168.10.222。

```java
10.0.0.0 via 192.168.10.222
```

添加主机路由、默认路由、网段路由示例如下，其中dev是可以省略的，因为没有任何用处，配置在哪个eth文件中就会从哪个接口出去。

```java
#默认路由
default     via 192.168.100.1
0.0.0.0/0   via 192.168.100.1

#网段路由
192.168.10.0/24   via 192.168.100.1

#主机路由
192.168.100.52/32 via 192.168.100.33 dev eth1
```

配置完后，重启network服务即可立即生效。

route-ethX文件的还有另外一种永久路由的配置写法，但上面的方法更简单快捷，所以此处就不多说了。

配置永久路由时，需要注意几点：

- (1).route-ethX的对应网卡配置文件ifcfg-ethX必须存在，否则路由无效。(对于虚拟机，通常新添加的网卡都没有对应的ifcfg-ethX文件，但ifconfig却能找到该网卡)

- (2).如果在文件中配置永久默认路由，则必须保证所有使用了DHCP服务的网卡配置文件ifcfg-ethX中的DEFROUTE指令设置为"no"，表示DHCP不设置默认路由。

- (3).如果在route-ethX文件中配置永久路由，且该网卡使用了DHCP服务分配地址，则必须保证该网卡的ifcfg-ethX文件中的PEERROUTES指令设置为"no"，表示DHCP设置的路由允许被覆盖。

## arp和arping命令 

维护或查看系统arp缓存，该命令已废弃，使用ip neigh代替。

arp为地址解析协议，将给定的ipv4地址在网络中查找其对应的MAC地址。

一般会使用arp协议获取局域网内的主机MAC，所以局域网主机之间也互称为网络邻居。

### arp命令 

arp命令语法：

```
arp -n -v -i           # 查看arp缓存
arp -i -d hostname     # 删除arp缓存条目

选项说明：
-n：不解析ip地址为名称
-v：详细信息
-i：指定操作的接口
-d：删除一个arp条目
```

hostname：操作该主机的arp条目，除了删除还有其他动作，如手动添加主机的arp条目，此处就不解释该用法了

例如：

```
[root@xuexi ~]# arp -n
Address            HWtype  HWaddress           Flags Mask  Iface
192.168.100.1      ether   00:50:56:c0:00:08   C           eth1
192.168.100.254    ether   00:50:56:e7:e1:d4   C           eth0
192.168.100.70     ether   00:0c:29:71:81:64   C           eth0
192.168.100.1      ether   00:50:56:c0:00:08   C           eth0
192.168.100.2      ether   00:50:56:e2:16:04   C           eth1
192.168.100.254    ether   00:50:56:e7:e1:d4   C           eth1
192.168.100.2      ether   00:50:56:e2:16:04   C           eth0
```

其实查看的信息是/proc/net/arp文件中的内容。

```
[root@xuexi ~]# cat /proc/net/arp
IP address       HW type   Flags  HW address            Mask   Device
192.168.100.1    0x1       0x2    00:50:56:c0:00:08     *      eth1
192.168.100.254  0x1       0x2    00:50:56:e7:e1:d4     *      eth0
192.168.100.70   0x1       0x2    00:0c:29:71:81:64     *      eth0
192.168.100.1    0x1       0x2    00:50:56:c0:00:08     *      eth0
192.168.100.2    0x1       0x2    00:50:56:e2:16:04     *      eth1
192.168.100.254  0x1       0x2    00:50:56:e7:e1:d4     *      eth1
192.168.100.2    0x1       0x2    00:50:56:e2:16:04     *      eth0
```

```
[root@xuexi ~]# arp -d 192.168.100.70 -i eth0   # 删除arp缓存条目
```

arp命令一次只能删除一条arp条目，要批量删除或清空整个arp条目，使用`ip neigh flush`命令。如：

```
[root@xuexi ~]# ip neigh flush all       # 清空所有
[root@xuexi ~]# ip neigh flush dev eth0  # 删除eth0上缓存的arp条目
```

### arping命令 

arping用于发送arp请求报文，解析并获取目标地址的MAC。默认将先发送广播报文，收到回复后再发送单播报文，局域网内所有主机都能收到广播报文，但只有目标主机才会回复自己的MAC地址。

注意：发送arp请求报文实际上是另类的ping，所以可以探测目标是否存活，也需要和目标通信，通信时目标主机上也会缓存本主机(即源地址)的arp条目。

语法：

```
arping [-fqbDU] [-c count] [-w timeout] [-I device] [-s source] destination
-f : 收到第一个reply就立即退出
-q : 安静模式，什么都不输出
-b : 只发送广播，不发送单播
-D : 地址冲突检测
-U : 主动更新邻居的arp缓存(Unsolicited ARP mode)
-c count : 发送多少个arp请求包后退出
-w timeout : 等待reply的超时时间
-I device : 使用哪个接口发送请求包。发送arp请求包接口的MAC地址将缓存在目标主机上
-s source : 指定arp请求报文中源地址，若发送的接口和源地址不同，则目标主机将缓存该地址和接口的MAC地址，而非该源地址所在接口的MAC地址
 destination : 向谁发送arp请求报文，即要获取该IP或主机名的MAC地址
```

例如：

(1).请求解析192.168.100.70主机的MAC地址

```
[root@xuexi ~]# arping -f 192.168.100.70
```

这将会发送广播报文，直到收到192.168.100.70的回复才退出。

同时，192.168.100.70也会缓存本机的IP和MAC对应条目，由于此处没有指定请求报文的发送接口和源地址，所以发送报文时是根据路由表来选择接口和对应该接口地址的。

(2).指定发送一个请求报文给192.168.100.70就退出，发送报文的接口为eth1，并指定请求报文中的源地址为本机eth0接口上的地址192.168.100.54

```
[root@xuexi ~]# arping -c 1 -I eth1 -s 192.168.100.54 192.168.100.70
```

发送这样的arp请求包，将会使得目标主机192.168.100.70缓存本机的arp条目为"192.168.100.54 MAC\_eth1"，但实际上，192.168.100.54所在接口的MAC地址为MAC\_eth0。

arping命令仅能实现这种简单的arp欺骗，更多的arp欺骗方法可以使用专门的工具。

(3).探测对方主机是否存活

例如发送4个探测报文，有回复就说明对方存活

```
[root@xuexi ~]# arping -c 4 -I eth0 192.168.100.2  
ARPING 192.168.100.2 from 192.168.100.54 eth0
Unicast reply from 192.168.100.2 [00:50:56:E2:16:04]  0.593ms
Unicast reply from 192.168.100.2 [00:50:56:E2:16:04]  0.930ms
Unicast reply from 192.168.100.2 [00:50:56:E2:16:04]  0.868ms
Unicast reply from 192.168.100.2 [00:50:56:E2:16:04]  0.844ms
Sent 4 probes (1 broadcast(s))
Received 4 response(s)
```

可见发送了4个探测报文，其中第一个报文是广播报文，并且收到了4个回复。

## ip命令 

这是一个极其强大的命令，前面所有的网络信息显示和管理的命令，都可以由ip命令来替代完成。它是一个严格模式化的命令。

### 获取ip命令的帮助 

先简单说明下ip命令的基础和获取帮助的方法。

```
[root@xuexi ~]# ip -h

Usage: ip [ OPTIONS ] OBJECT { COMMAND | help }
       ip [ -force ] -batch filename
where  OBJECT := { link | addr | addrlabel | route | rule | neigh | ntable |
                   tunnel | tuntap | maddr | mroute | mrule | monitor | xfrm |
                   netns | l2tp | tcp_metrics | token }
       OPTIONS := { -V[ersion] | -s[tatistics] | -d[etails] | -r[esolve] |
                    -h[uman-readable] | -iec |
                    -f[amily] { inet | inet6 | ipx | dnet | bridge | link } |
                    -4 | -6 | -I | -D | -B | -0 |
                    -l[oops] { maximum-addr-flush-attempts } |
                    -o[neline] | -t[imestamp] | -b[atch] [filename] |
                    -rc[vbuf] [size] | -n[etns] name | -a[ll] }
```

可见命令非常复杂，有很多options，还有很多object，每个Object又对应不同的命令。但其实能用到的就几个object：addr/route/neigh/link。

使用ip object help可以获取到该object的语法帮助。例如：

```
[root@xuexi ~]# ip addr help
```

在ip命令行下，任何object都可以写其全名，也可以写其缩写名，例如address这个object，可以简写为addr，也可以简写为一个字母a。

```
# 等价于ip address help和ip addr help
[root@xuexi ~]# ip a help
```

尽管还有一个a开头的object为addrlabel。这时因为ip会从上述语法给出的object顺序从前向后匹配，例如"ip m"将匹配到"ip maddr"，如果想匹配别的，如addrlabel，则写长一点即可"ip addrl"。

对于CentOS 6，man ip时会输出整个ip的帮助文档，包括每个object的命令和说明。在CentOS 7中，则要对每个object独立进行man，例如addr这个object。

```
[root@xuexi ~]# man ip-address
```

以下是所有Object的man列表。

```
[root@xuexi ~]# rpm -ql iproute | grep "man8/ip-"
/usr/share/man/man8/ip-address.8.gz
/usr/share/man/man8/ip-addrlabel.8.gz
/usr/share/man/man8/ip-l2tp.8.gz
/usr/share/man/man8/ip-link.8.gz
/usr/share/man/man8/ip-maddress.8.gz
/usr/share/man/man8/ip-monitor.8.gz
/usr/share/man/man8/ip-mroute.8.gz
/usr/share/man/man8/ip-neighbour.8.gz
/usr/share/man/man8/ip-netconf.8.gz
/usr/share/man/man8/ip-netns.8.gz
/usr/share/man/man8/ip-ntable.8.gz
/usr/share/man/man8/ip-route.8.gz
/usr/share/man/man8/ip-rule.8.gz
/usr/share/man/man8/ip-tcp_metrics.8.gz
/usr/share/man/man8/ip-token.8.gz
/usr/share/man/man8/ip-tunnel.8.gz
/usr/share/man/man8/ip-xfrm.8.gz
```

### ip addr 

ip addr用于管理网络设备上的ip地址，也可以查看ip地址的属性信息。在老版本的Linux中，一块网卡上设置多个IP，这些IP称为别名IP，但是从CentOS 6开始，这些IP称为secondary IP或slave IP，因为这些IP自身也可以附带属性。

(1).`ip addr add/del`

```
ip address { add | del } IFADDR dev STRING
IFADDR := PREFIX [ broadcast ADDR ] [ anycast ADDR ] [ label STRING ]

以add为例：
dev NAME：指定要设置IP地址的网卡
local ADDRESS (default)：接口的IP地址。IP地址的格式依赖于是ipv4还是ipv6。对于ipv4而言，给定地址，可能还需要给定cidr的掩码位长度
broadcast ADDRESS：接口的广播地址
label NAME：为该接口的IP地址设置label名，label名称必须以网络接口名开头后接冒号，如eth0:X

del和add的参数相同，且dev是必须要给定的，其余的参数可选，因为del的时候是通配del，如果删除时有多个满足条件的条目，则删除第一个条目。
```

例如：

```java
[root@xuexi ~]# ip addr add 192.168.100.45 dev eth0
[root@xuexi ~]# ip addr add 192.168.100.35/24 dev eth1
```

此方式添加的地址不会在ifconfg命令中显示，ifconfg能捕捉到的是别名，所以可以为地址加上label，以让secondary被ifconfig查看到。例如：

```
[root@xuexi ~]# ip addr add 192.168.100.45 dev eth0 label eth0:0
```

要删除ip，则简单的多，但必须指定dev，且最好也指定cidr的掩码长度。

```
[root@xuexi ~]# ip addr del 192.168.100.45 dev eth0
[root@xuexi ~]# ip addr del 192.168.100.35/24 dev eth1
```

(2).`ip addr show`

虽然也有几个选项，但是感觉没什么用，直接ip addr show就够了。因为ip命令可以缩写，所以可以写为

```
[root@xuexi ~]# ip a show
[root@xuexi ~]# ip a s
[root@xuexi ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:71:81:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.70/24 brd 192.168.100.255 scope global eth0
    inet6 fe80::20c:29ff:fe71:8164/64 scope link
       valid_lft forever preferred_lft forever
```

(3).`ip addr flush`

用于批量删除地址，该命令其实非常危险，一个不小心就会误伤无辜，所幸的是flush的时候不给定任何参数或者没有任何条目可以匹配上的时候将不执行flush动作，总之该命令要小心使用。同样也必须给定dev参数。

例如删除eth1上所有地址。

```
[root@xuexi ~]# ip a flush dev eth1
```

删除eth1上所有的secondary地址。

```
[root@xuexi ~]# ip a f secondary dev eth1
```

### ip route 

该命令维护和查看内核中的路由表。

(1).`ip route add/del/change/append/replace`

语法格式为：

```
ip route { add | del | change | append | replace } dest[/cidr_mask] [ via ADDRESS ] [ dev STRING ]
```

其中dest为目标地址，可以是主机地址、网段地址，一般在地址后都会带上cidr格式的掩码长度，不带时默认为32位长度。如果dest为"0/0"或者写为"default"，则表示默认路由。

例如添加/修改/替换普通路由：

```
[root@xuexi ~]# ip route add/change/replace 172.16.10.0/24 via 192.168.10.20
```

添加/修改/替换默认路由：

```
[root@xuexi ~]# ip route add/change/replace default via 192.168.10.20
[root@xuexi ~]# ip route add/change/replace 0/0 via 192.168.100.2
```

删除某路由：

```
[root@xuexi ~]# ip route del 172.16.10.0/24
[root@xuexi ~]# ip route del default   # 删除默认路由
```

(2).`ip route show`

列出路由表。

语法格式为：

```
ip route show [to [ root | match | exact ] ADDR_pattern ] [ via ADDR ]
```

其中to关键字是默认关键字，用来匹配路由的目标地址。其后可以跟上修饰符root/match/exact，exact为默认修饰符，表示精确匹配掩码位长度，root修饰符表示匹配的掩码位长度大于或等于ADDR\_pattern给定的掩码位长度，match修饰符匹配短于或等于ADDR\_pattern掩码位长度。例如"to match 16.0/16"将能匹配到"16.0/16"、"16/8"和"0/0"，但却无法匹配"16.1/16"和"16.0/24"以及"16.0.1/24"，而"to root 16.0/16"将能匹配"16.0/24"和"16.0.1/24"。

via是根据下一跳的方式来列出路由条目。

例如：

```
[root@xuexi ~]# ip route show | column -t                    
default           via  192.168.100.2  dev    eth0    proto  static  metric  100
172.16.10.0/24    dev  eth1   proto  kernel  scope  link  src  172.16.10.20    metric  100
172.168.10.0/24   dev  eth0   proto  kernel  scope  link  src  172.168.10.20
172.168.10.0/24   dev  eth0   proto  kernel  scope  link  src  172.168.10.20   metric  100
192.168.10.0/24   dev  eth0   proto  kernel  scope  link  src  192.168.10.20
192.168.10.0/24   dev  eth0   proto  kernel  scope  link  src  192.168.10.20   metric  100
192.168.100.0/24  dev  eth0   proto  kernel  scope  link  src  192.168.100.54  metric  100
192.168.100.0/24  dev  eth1   proto  kernel  scope  link  src  192.168.100.74  metric  101

[root@xuexi ~]# ip route show to match 192.168.10/24 | column -t
default          via  192.168.100.2  dev    eth0    proto  static  metric  100
192.168.10.0/24  dev  eth0  proto  kernel  scope  link  src  192.168.10.20
192.168.10.0/24  dev  eth0  proto  kernel  scope  link  src  192.168.10.20  metric  100

[root@xuexi ~]# ip route show to match 192.168/24 | column -t      
default  via  192.168.100.2  dev  eth0  proto  static  metric  100

[root@xuexi ~]# ip route show to root 192.168/16 | column -t 
192.168.10.0/24   dev  eth0  proto  kernel  scope  link  src  192.168.10.20
192.168.10.0/24   dev  eth0  proto  kernel  scope  link  src  192.168.10.20   metric  100
192.168.100.0/24  dev  eth0  proto  kernel  scope  link  src  192.168.100.54  metric  100
192.168.100.0/24  dev  eth1  proto  kernel  scope  link  src  192.168.100.74  metric  101
```

其实无需那么花哨，简简单单的`ip r`多方便。

(3).`ip route flush`

批量删除路由表条目。参数和`ip route show`的参数一样。

例如删除由eth1出去的路由条目。

```java
[root@xuexi ~]# ip route flush eth1
```

删除下一跳为192.168.100.70的路由条目。

```java
[root@xuexi ~]# ip r flush via 192.168.100.70
```

删除目标为192.168.0.0/16网段的路由

```java
[root@xuexi ~]# ip route flush 192.168/16
```

(4).`ip route save/restore`

用于保存当前的路由表以及恢复路由表。保存路由表时，路由表将以二进制裸数据的格式输出，也就是看不懂的二进制文件。恢复路由表时，要求设备的设置和保存路由表时是一样的，恢复时已存在于路由表中的路由条目将被忽略。

例如当前路由表信息如下：

```
[root@xuexi ~]# ip r
default via 192.168.100.2 dev eth0  proto static  metric 100
192.168.10.0/24 dev eth0  proto kernel  scope link  src 192.168.10.20
192.168.10.0/24 dev eth0  proto kernel  scope link  src 192.168.10.20  metric 100
192.168.100.0/24 dev eth0  proto kernel  scope link  src 192.168.100.54  metric 100
192.168.100.0/24 dev eth1  proto kernel  scope link  src 192.168.100.74  metric 101
```

保存当前路由表。

```
[root@xuexi ~]# ip route save > /tmp/route.txt
```

删除几条路由表。

```
[root@xuexi ~]# ip route flush dev eth0

[root@xuexi ~]# ip r
192.168.100.0/24 dev eth1  proto kernel  scope link  src 192.168.100.74  metric 101
```

恢复路由表。

```
[root@xuexi ~]# ip route restore < /tmp/route.txt
```

### ip link 

link表示link layer的意思，即链路层。该命令用于管理和查看网络接口，甚至可以添加虚拟网络接口，将网络接口分组进行管理。

(1).`ip link set`

```
ip link set DEVICE  { up | down | arp { on | off } | name NEWNAME | address LLADDR }  

选项说明：
dev DEVICE：指定要操作的设备名
up and down：启动或停用该设备
arp on or arp off：启用或禁用该设备的arp协议
name NAME：修改指定设备的名称，建议不要在该接口处于运行状态或已分配IP地址时重命名
address LLADDRESS：设置指定接口的MAC地址
```

例如，禁用eth1网卡。

```
[root@xuexi ~]# ip link set eth1 down
```

其实等价于:

```
[root@xuexi ~]# ifconfig eth1 down
```

修改网卡eth1的MAC地址。

```
[root@xuexi ~]# ip link set eth1 address 00:0c:29:f3:33:77
```

(2).`ip link show`

语法格式：

```
ip [ -s | -h ] link show [dev DEV]  

选项说明：
-s：将显示各网络接口上的流量统计信息
-h：以人类可读的方式显式，即单位转换。注："-h"在CentOS 7上才支持。
```

例如：

```
[root@xuexi ~]# ip -s -h link show  dev eth0  
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 00:0c:29:f7:43:77 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast  
    13.7M      20.0k    0       0       0       0      
    TX: bytes  packets  errors  dropped carrier collsns
    1.59M      9.97k    0       0       0       0
```