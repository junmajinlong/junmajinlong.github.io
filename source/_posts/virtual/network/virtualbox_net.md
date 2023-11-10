---
title: 理解VirtualBox网络
p: virtual/network/virtualbox_net.md
date: 2020-10-14 17:37:30
tags: Virtualization
categories: Virtualization
---

--------

**[网络虚拟化系列文章](/virtual/index)**

--------

# 理解VirtualBox网络

## VBox网络连接类型

VBox有五种网络连接方式：  
- 桥接  
- NAT(网络地址转换)  
- NAT网络  
- Host-Only  
- 内部网络  

![](/img/virtual/1585132780552.png)

它们之间的通信关系如下：其中NATService即NAT网络方式。

![](/img/virtual/1585142594991.png)

## 桥接类型

VBox的桥接类型是创建一个看不见的虚拟交换机，虚拟机中设置为桥接模式的网卡和宿主机的物理网卡都连接在这个虚拟交换机上。

所以：

- 桥接网络内的虚拟机和物理网卡在同一个网段，各虚拟机及宿主机之间可以互相通信  
- 虚拟机桥接网络的网关默认和物理网卡的网关相同，所以物理网卡能上网，虚拟机就能上网  

它的模型大概如下：

![](/img/virtual/1602751688325.png)

桥接后，虚拟机网卡的IP地址和网关：

```
$ ip a s eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b3:36:6c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.9/24 brd 192.168.1.255 scope global noprefixroute dynamic eth0
       valid_lft 86271sec preferred_lft 86271sec

$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 eth0
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

物理网卡属性：

![](/img/virtual/1602753019923.png)

## 内部网络

内部网络是在宿主机上创建一个虚拟交换机，该虚拟交换机不和宿主机上的任何网卡相连，只有放在内部网络的各虚拟机网卡才会连接到此虚拟交换机。

所以，内部网络中的虚拟机之间可以互相通信，但它们不能和宿主机互相通信。

它的模型大概如下：

![](/img/virtual/1602753397338.png)



## Host-Only网络

每个Host-Only网络都会在宿主机上创建一个虚拟交换机，同时还会在宿主机创建一个连接到此虚拟交换机的虚拟网卡。Host-Only网络可以创建多个，即多个虚拟交换机+多个连接到各自虚拟交换机的虚拟网卡。

所以，Host-Only网络内的虚拟机之间以及宿主机之间可以互相通信。但因为虚拟交换机没有连接到物理网卡，所以Host-Only网络内的流量不能超出该网络，比如无法上外网，无法和其它Host-Only或其它类型的网络通信。

它的模型大概如下：

![](/img/virtual/1602753615936.png)

host-only网络内虚拟机的IP地址：

```
$ ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.101  netmask 255.255.255.0  broadcast 192.168.56.255
        ether 08:00:27:b3:36:6c  txqueuelen 1000  (Ethernet)
        RX packets 405  bytes 38712 (37.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 265  bytes 32810 (32.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

宿主机上为host-only创建的虚拟网卡：

![](/img/virtual/1602753940490.png)

### Host-Only上外网

由于Host-Only网络会在宿主机上创建对应的虚拟网卡，所以可以通过网络共享(即转发，ip_forward)的方式将物理网卡网络共享给某Host-Only网络内的宿主机虚拟网卡，然后设置该Host-Only网络内虚拟机的网关，便可以让该Host-Only网络内的流量通过物理网卡跨出该虚拟网络。

模型如下：

![](/img/virtual/1602754291544.png)

## NAT

VBox中的NAT借助了VBox内置的NAT引擎和DHCP服务，在VBox管理程序的网络图形界面下，除了将虚拟机放入NAT以及端口转发外，没有任何其它可操作的配置。换句话说，用户只需要将虚拟机放入NAT，VBox会按照默认方式自动设置好一切，用户也没法修改相关网络设置。

比如，VBox会自动将所有放入NAT的虚拟机放入10.0.2.0/24网段，并将它们的网关自动设置为10.0.2.2。

此外，宿主机为每个设置为NAT连接方式的虚拟机维护一个私有的虚拟NAT设备，每个虚拟NAT设备都以宿主机的物理网卡作为外部接口：每个虚拟NAT设备将虚拟机流向外部的流量做地址转换，即转换为物理网卡的地址，以便它们能通过物理网卡连接到外部网络。

它的模型如下图：

![](/img/virtual/1585145940847.png)

由于每个虚拟机都使用了自己私有的虚拟NAT设备将自己隔离开了，所以虚拟机之间不能互相通信。

注意，宿主机默认不能访问虚拟机，除非设置端口转发。其实很容易理解宿主机为什么不能访问虚拟机，想象一下，外网主机无法访问通过NAT隔离的内网主机一样。

NAT引擎为每个虚拟机都维护一个私有的虚拟NAT设备的另一方面，是所有**虚拟机的IP地址都是10.0.2.15**，相当于隔离了一个私网，只不过这个私网内只有一个地址且固定，它们并不会地址冲突。

![](/img/virtual/1585147089211.png)

## NAT网络

NAT网络连接模式和NAT连接模式类似，都提供网络地址转换和端口转发功能。但是NAT网络以NAT服务的方式存在，多个虚拟机全部连接在虚拟交换机上，虚拟交换机再连接在虚拟NAT上，并且使用一个DHCP服务来分配地址。

模型如下：

![](/img/virtual/1585147168758.png)

在NAT网络服务下，虚拟机之间可以互相通信，也可以和宿主机通信，但是宿主机不能和虚拟机通信，除非使用了端口转发功能。

用户可以在全局设定下创建一个或多个NAT网络并修改该NAT网络相关属性：

![](/img/virtual/1585148037319.png)

其实Vmware Workstation的NAT和Vbox的NAT网络是类似的，只不过VMware的NAT比VBox的NAT网络多了一个操作：还会在宿主机上创建一个连接到虚拟交换机的虚拟网卡，有了这个虚拟网卡，宿主机和虚拟机之间就可以互相通信。

