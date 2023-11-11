---
title: Linux DHCP服务
p: linux/dhcp.md
date: 2019-08-07 11:29:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux DHCP服务

DHCP前身是BOOTP，在Linux的网卡配置中也能看到显示的是BOOTP，DHCP引进一个bootp没有的概念：租约。bootp分配的地址是永久的，而dhcp分配的地址是可以有期限的。

```
[root@xuexi vsftpd]# grep -i bootproto /etc/sysconfig/network-scripts/ifcfg-eth0
BOOTPROTO=dhcp
```

DHCP可以自动分配IP、子网掩码、网关、DNS。

DHCP客户端使用的端口68，服务端使用端口67，使用的UDP应用层的协议。

DHCP一般不为服务器分配IP，因为他们要使用固定IP，所以DHCP一般只为办公环境的主机分配IP。

DHCP服务器和客户端需要在一个局域网内，在为客户端分配IP的时候需要进行多次广播。但DHCP也可以为其他网段内主机分配IP，只要连接两个网段中间的路由器能转发DHCP配置请求即可，但这要求路由器配置中继功能。

![](/img/linux/733013-20170809104133527-1990385690.png)

![](/img/linux/733013-20170809104140667-128016091.png)

## DHCP客户端请求过程（4步请求过程）

- (1).搜索阶段：客户端广播方式发送报文，搜索DHCP服务器。此时网段内所有机器都收到报文，只有DHCP服务器返回消息。
- (2).提供阶段：众多DHCP服务器返回报文信息，并从地址池找一个IP提供给客户端。因为此时客户端还没有IP，所以返回信息也是以广播的方式返回的。
- (3).选择阶段：选择一个DHCP服务器，使用它提供的IP。然后发送广播包，告诉众多DHCP服务器，其已经选好DHCP服务器以及IP地址。此后没有入选的DHCP就可以将原本想分配的IP分配给其他主机。
  客户端选择第一个接收到的IP。谁的IP先到客户端的速度是不可控的。但是如果在配置文件里开启了authoritative选项则表示该服务器是权威服务器，其他DHCP服务器将失效，如果多台服务器都配置了这个权威选项，则还是竞争机制；通过MAC地址给客户端配置固定IP也会优先于普通的动态DHCP分配。另外Windows的DHCP服务端回应Windows客户端比Linux更快。
- (4).确认阶段：DHCP服务器收到回应，向客户端发送一个包含IP的数据包，确认租约，并指定租约时长。

如果DHCP服务器要跨网段提供服务，一样是四步请求，只不过是每一步中间都多了一个路由器和DHCP服务器之间的单播通信。

- (1).客户端广播方式发送报文，搜索DHCP服务器。所有机器包括路由器都收到报文，路由器配置了中继，知道搜索消息后单播给DHCP服务器；
- (2).DHCP服务器单播返回信息给路由器，路由器再广播给客户端；
- (3).客户端选择DHCP服务器提供的IP，并广播信息告诉它我选好了，路由器单播给DHCP服务器；
- (4).DHCP服务器收到信息将确认信息单播给路由器，路由器单播给客户端。

所以DHCP的4步请求：

```
Client--> DHCPDISCOVER             # 广播：客户端发现DHCP服务器
          DHCPOFFER <-- Server     # 广播：服务端提供IP给客户端

Client--> DCHPREQUEST              # 广播：客户端请求使用提供的IP
          DCHPACK <-- Server       # 单播：服务端进行确认，订立租约等信息
```

续租的过程：

```
Client--> DHCPREQUEST              # 单播：继续请求使用提供的IP
          DHCPACK <-- Server       # 单播：确认续租
```

DHCP服务器**不跨网段提供服务时，**它自己的IP地址必须要和地址池中全部IP在同一网络中。

DHCP服务器**跨网段提供服务时，**它自己的IP地址必须要和地址池中的一部分IP在同一网络中，另一部分提供给其他网段。因为如果自己的IP完全不在自己的网络中而只提供其他网段的IP，更好的做法是将DHCP服务器设在那个需要DHCP服务的网络中。

当计算机从一个子网移到另一个子网，找的DHCP服务器不同，因为旧的租约还存在，会先续租，新的DHCP服务器肯定拒绝它的续租请求，这时将重新开始四步请求。

有些机器希望一直使用一个固定的IP，也就是静态IP，除了手动进行配置，DHCP服务器也可以实现这个功能。DHCP服务器可以根据MAC地址来分配这台机器固定IP地址（保留地址），即使重启或重装了系统也不会改变根据MAC地址分配的地址。

假如在一个正常联网有DHCP服务器的网段内因为做实验练习的缘故新建立了一台DHCP服务器，但是这台DHCP服务器不能上网，会导致什么后果？使用DHCP分配地址的客户端至少会有续租的请求，如果没有续租成功，或者有新的计算机加入这个网络，那么进行四步请求，有可能会请求到这个不能连网的DHCP服务器上，那么他也就不能上网了。特别是Windows的DHCP服务端回应Windows客户端速度比Linux回应快。

## 安装和配置DHCP服务

```
[root@xuexi ~]# yum -y install dhcp

[root@xuexi ~]# rpm -ql dhcp
/etc/dhcp/dhcpd.conf    # DHCP配置文件
/etc/sysconfig/dhcpd
/usr/sbin/dhcpd         # DHCP服务程序
/usr/sbin/dhcrelay      # 中继命令程序，用于跨网段提供DHCP服务
/var/lib/dhcpd/dhcpd.leases    # 存放租借信息（如IP）和租约信息（如租约时长）
/usr/share/doc/dhcp-4.1.1/dhcpd.conf.sample # 配置文件的范例文件
```

可以将dhcpd.conf.sample复制到/etc/。

```
cp /usr/share/dhcp-4.1.1/dhcpd.conf.sample /etc/dhcpd.conf 
```

以下是dhcpd.conf中部分配置项。

```
# 每行分号结束
ddns-update-style none;      # 动态dns相关，几乎不开启它。也就是不管它。
ignore client-updates;       # 和上面的相关，也不管它
authoritative                # 声明为权威服务器
next-server marvin.redhat.com;    # PXE环境下指定的提供引导程序的文件服务器

# DHCP配置文件里必须配置一个地址池，其和DHCP服务器自身IP在同一网段
subnet 10.5.5.0 netmask 255.255.255.224 {
  # 地址池
  range 10.5.5.26 10.5.5.30;
  # 为客户端指明DNS服务器地址，可以是多个，最多三个
  option domain-name-servers ns1.internal.example.org;
  # 为客户端指明DNS名字，定义了它会覆盖客户端/etc/resolv.conf里的配置
  option domain-name "internal.example.org";
  # 默认路由，其实就是网关
  option routers 10.5.5.1;
  # 广播地址，不设置时默认会根据A/B/C类地址自动计算
  option broadcast-address 10.5.5.31;
  # 默认租约时长
  default-lease-time 600;
  # 最大租约时长
  max-lease-time 7200;
}

#下面的是绑定MAC地址设置保留地址，保留地址不能是地址池中的地址
host fantasia {   # 固定地址的配置，host后面的是标识符，没意义
hardware ethernet 08:00:07:26:c0:a5;
  fixed-address 192.168.100.3;  # 根据MAC地址分配的固定IP 
}
```

如果不让dhcp修改/etc/resolv.conf里的内容，就在网卡配置文件/etc/sysconfig/network-scripts/ifcfg-ethX里添加一行选项：`PEERDNS=no`。

在客户端如何获取动态分配的地址呢？

方法一：`service network restart`，但是每次重启网络很麻烦，可以使用客户端命令dhclient。

方法二：直接执行dhclient命令。这种方法会显示4步请求中需要显示的步骤信息，以及最终分配的地址，所以是一个很好的理解dhcp工作的工具。但是这种方法只能使用一次，第二次执行命令会提示该进程已经在执行，因为dhclient是一个进程。可以kill掉该进程再执行dhclient，或者使用`dhclient -d`选项。

方法三：`dhclient -d`

## 如何重新获取IP地址

每次重启网卡默认都获取的同一个ip，有时候想换个ip都很麻烦。在/var/lib/dhclient/目录下有".leases"文件，将它们清空或者删除这些文件中对应网卡的部分，再重启网络就可以获取新的动态ip。

```
[root@xuexi ~]# cat /var/lib/dhclient/dhclient-eth0.leases 
lease {
  interface "eth0";
  fixed-address 192.168.100.16;
  option subnet-mask 255.255.255.0;
  option routers 192.168.100.2;
  option dhcp-lease-time 1800;
  option dhcp-message-type 5;
  option domain-name-servers 192.168.100.2;
  option dhcp-server-identifier 192.168.100.254;
  option broadcast-address 192.168.100.255;
  option domain-name "localdomain";
  renew 3 2017/02/15 12:28:27;
  rebind 3 2017/02/15 12:42:39;
  expire 3 2017/02/15 12:46:24;
}
```

或者，在/etc/sysconfig/network-scripts/ifcfg-eth0加入`DHCPRELEASE=yes`。

当运行`ifdown eth0`的时候就会发出dhcprelase报文，查看/etc/sysconfig/network-scripts/ifdown-eth脚本中实际上是调用`dhclient`命令，用下面这个命令应该也可以。

```
/sbin/dhclient -r eth0
```