---
title: Linux namespace之：network namespace
p: virtual/namespace/network_namespace.md
date: 2020-10-10 17:37:34
tags: Virtualization
categories: Virtualization
---

--------

**目录**：  
- **[Linux namespace概述](/virtual/namespace/ns_overview)**  
- **[Linux namespace之：uts namespace](/virtual/namespace/uts_namespace)**  
- **[Linux namespace之：mount namespace](/virtual/namespace/mount_namespace)**  
- **[Linux namespace之：pid namespace](/virtual/namespace/pid_namespace)**  
- **[Linux namespace之：network namespace](/virtual/namespace/network_namespace)**  
- **[Linux namespace之：user namespace](/virtual/namespace/user_namespace)**  

--------


# 理解network namespace

network namespace用来隔离网络环境，**在network namespace中，网络设备、端口、套接字、网络协议栈、路由表、防火墙规则等都是独立的**。

因network namespace中具有独立的网络协议栈，因此每个network namespace中都有一个lo接口，但lo接口默认未启动，需要手动启动起来。

```bash
# -n或--net选项用于创建network namespace
$ sudo unshare -n /bin/bash

# 默认未启动lo
root@longshuai-vm:/home/longshuai# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# 将之启动
root@longshuai-vm:/home/longshuai# ip link set lo up
root@longshuai-vm:/home/longshuai# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
```

让某个network namespace和root network namespace或其他network namespace之间保持通信是一个非常常见的需求，这一般通过veth虚拟设备实现。veth类型的虚拟设备由一对虚拟的eth网卡设备组成，像管道一样，一端写入的数据总会从另一端流出，从一端读取的数据一定来自另一端。

用户可以将veth的其中一端放在某个network namespace中，另一端保留在root network namespace中。这样就可以让用户创建的network namespace和宿主机通信。

例如：
```bash
##### 在第一个shell窗口
# 创建network namespace ns1
$ sudo unshare -n /bin/bash

# 查看该network namespace的进程号
root@longshuai-vm:/home/longshuai# echo $$
53091


##### 在第二个shell窗口
# 创建veth设备
$ sudo ip link add veth0 type veth peer name veth1

# 创建之后，就有了一对虚拟的eth设备
$ ip a | grep veth
3: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
4: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000

# veth0准备留在root network namespace中
# 为veth0设置IP地址并启动
$ sudo ip a add dev veth0 192.168.10.10/24
$ sudo ip link set veth0 up
$ ip a s veth0
4: veth0@veth1: <NO-CARRIER,BROADCAST,MULTICAST,UP,M-DOWN> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 36:30:54:ba:d3:aa brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.10/24 scope global veth0
       valid_lft forever preferred_lft forever

# 最后，将veth1移动到刚才新建的network namespace ns1中
# ip link set xxx netns [ PID|NETNS_NAME ]
$ sudo ip link set veth1 netns 53091

# 注：
# 并不是所有的网络设备都能在network namespace之间移动，
# ethtool -k <interface>输出的结果中，netns-local为on的表示不能移动
$ ethtool -k lo | grep netns
netns-local: on [fixed]
$ ethtool -k ens32 | grep netns      
netns-local: off [fixed]
$ ethtool -k veth1 | grep netns                         
netns-local: off [fixed]

##### 在ns1中启动veth1并设置IP地址
root@longshuai-vm:/home/longshuai# ip link set dev veth1 up

root@longshuai-vm:/home/longshuai# ip a add dev veth1 192.168.10.20/24

root@longshuai-vm:/home/longshuai# ip a s veth1
3: veth1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 22:a8:4b:5b:55:4d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.10.20/24 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::20a8:4bff:fe5b:554d/64 scope link 
       valid_lft forever preferred_lft forever 

# 互ping，测试两端是否能通信，例如在ns1测试
root@longshuai-vm:/home/longshuai# ping 192.168.10.10
PING 192.168.10.10 (192.168.10.10) 56(84) bytes of data.
64 bytes from 192.168.10.10: icmp_seq=1 ttl=64 time=0.076 ms
64 bytes from 192.168.10.10: icmp_seq=2 ttl=64 time=0.189 ms
```

但现在ns1还不能和公网通信。解决这个问题也很简单，在root network namespace中开启转发并设置SNAT，在ns1中添加默认路由即可。
```bash
##### 在root network namespace中执行
$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo iptables -t nat -A POSTROUTING -o ens32 -j MASQUERADE

##### 在network namespace ns1中执行
$ route add default gw 192.168.10.10
$ ping www.baidu.com
PING www.baidu.com (36.152.44.95) 56(84) bytes of data.
64 bytes from 36.152.44.95: icmp_seq=1 ttl=127 time=53.4 ms
64 bytes from 36.152.44.95: icmp_seq=2 ttl=127 time=51.0 ms
```


## 持久化的network namespace

正常情况下，当namespace中的所有进程都退出后，namespace也会随之销毁。但有时候需要让namespace即使没有进程在其中运行也依然有效，即namespace的持久化。例如，创建了network namespace后，想要让network namespace中的网络配置一直生效。

实际上，无论是network namespace还是其他类型的namespace，**当通过mount bind为某个namespace的namespace文件(`/proc/$$/ns/xxx`)进行了bind挂载，这个namespace将成为持久化的namespace，即使namespace中的第一个进程或所有进程都退出了，namespace也不会立即销毁，之后还可以通过nsenter重新进入该namespace**。

其实方式很简单，直接在unshare创建namespace时的对应长选项上指定一个已存在的文件即可(注：不允许在短选项上指定文件名)。
```
unshare: 
 -m, --mount[=<file>]   unshare mounts namespace
 -u, --uts[=<file>]     unshare UTS namespace (hostname etc)
 -i, --ipc[=<file>]     unshare System V IPC namespace
 -n, --net[=<file>]     unshare network namespace
 -p, --pid[=<file>]     unshare pid namespace
 -U, --user[=<file>]    unshare user namespace
 -C, --cgroup[=<file>]  unshare cgroup namespace
```

例如，想要让network namespace持久化：
```bash
# 对于network namespace持久化，通常指定/var/run/netns/NAME作为持久化文件
$ sudo mkdir -p /var/run/netns
$ sudo touch /var/run/netns/ns1

# 也可以不在选项--net上指定文件，而是在network namespace内部
# 将/var/run/netns/ns1通过mount bind挂载到/proc/$$/ns/net上
$ sudo unshare --net=/var/run/netns/ns1 /bin/bash

# 现在/var/run/netns/ns1和/proc/$$/ns/net是同一个network namespace
root@longshuai-vm:/home/longshuai# ls -i /var/run/netns/ns1 
4026532587 /var/run/netns/ns1

root@longshuai-vm:/home/longshuai# readlink /proc/$$/ns/net
net:[4026532587]

# 退出ns1
root@longshuai-vm:/home/longshuai# exit
exit

# /var/run/netns/ns1仍然指向net inode
$ ls -i /var/run/netns/ns1 
4026532587 /var/run/netns/ns1

# nsenter重新进入ns1
$ sudo nsenter --net=/var/run/netns/ns1 /bin/bash
root@longshuai-vm:/home/longshuai# hostname -I
192.168.10.20
```

## ip netns

`ip netns`命令用于管理network namespace。

ip netns将network namespace与/var/run/netns/NAME相关联，将/var/run/netns下的每一个NAME作为其所管理的每一个network namespace的名称。

ip netns将/etc/netns/NAME/作为对应network namespace的全局网络配置文件的目录，查找它之后才会查找/etc/目录。

例如，如果想要为netns名为ns1的network namespace单独设置DNS，可创建/etc/netns/ns1/resolv.conf，并将DNS相关配置写入该文件，当该文件不存在时才查找/etc/resolv.conf。

ip netns创建network namespace时，同时会创建mount namespace，以便将网络相关配置文件/etc/netns/NAME/xxx挂载到对应的/etc/xxx。

1. **ip netns add NAME**  
   创建名为NAME的network namespace，同时会关联/var/run/netns/NAME文件，如果文件不存在，则会自动创建

   ```bash
   $ sudo ip netns add ns2
   $ ls -i /var/run/netns/ns2
   4026532759 /var/run/netns/ns2
   ```

2. **ip netns list**  
   列出/var/run/netns下的所有network namespace

   ```bash
   # 带有id的，表示正在运行(即有进程尚未退出)的network namespace以及它的ID号
   # ID会自动分配，从0开始，后面通过ip netns命令也可以自己设置ID号
   $ ip netns list
   ns2
   ns1 (id: 0)
   ```

3. **ip netns attach NAME PID**  
   将PID对应的network namespace关联到/var/run/netns/NAME(不存在时会自动创建)，使得该network namespace就像是被ip netns创建一样，之后它将受ip netns管理

   ```bash
   # 在第一个窗口中执行
   # 使用unshare而非ip netns创建一个network namespace
   $ sudo unshare -n /bin/bash
   root@longshuai-vm:/home/longshuai# echo $$
   5094

   # 在第二个窗口中执行
   # 现在这个network namespace就像是由ip netns创建一样
   $ sudo ip netns attach ns3 5094
   ```

4. **ip [-all] netns delete [ NAME ]**  
   删除/vaer/run/netns/下指定的network namespace，如果指定了--all，则删除/var/run/netns下所有的network namespace。注意，它同时会卸载mount bind的挂载点/var/run/netns/NAME并删除该文件。

   ```bash
   $ ip netns list
   ns3
   ns2
   ns1 (id: 0)

   $ sudo ip netns del ns3
   $ ls /var/run/netns
   ns1  ns2

   $ sudo ip --all netns del
   $ ls /var/run/netns
   ```

5. **ip netns set NAME NETNSID**  
   为/var/run/netns/NAME对应的network namespace设置ID号。  

   ```bash
   $ sudo ip netns add ns1
   $ ip netns list
   ns1
   
   $ sudo ip netns set ns1 11
   $ ip netns list
   ns1 (id: 11)
   ```

6. **ip netns identify [PID]**  
   根据PID，输出该PID所在的network namespace的netns name。

   ```bash
   $ ip netns list
   ns1 (id: 11)
   $ sudo nsenter --net=/var/run/netns/ns1 /bin/bash
   root@longshuai-vm:/home/longshuai# echo $$
   5298
   
   # 在另一个窗口中查询进程PID=5298在哪一个network namespace中
   $ sudo ip netns identify 5298
   ns1
   ```

7. **ip netns pids NAME**  
   输出networ namespace中当前正在运行的所有进程PID  

   ```bash
   $ sudo ip netns pids ns1
   5298
   ```

8. **ip [-all] netns exec [ NAME ] cmd ...**  
   在指定的network namespace中执行命令CMD。如果指定了--all选项，则CMD命令将在/var/run/netns/下的所有network namespace中都执行。
   
   ```bash
   $ sudo ip netns exec ns1 ip link set lo up
   $ sudo ip netns exec ns1 ping www.baidu.com
   ```

注：如果/etc/netns/NAME下有配置文件，执行ip netns exec命令时会自动将其bind到/etc/下对应的配置文件上。此时还需注意，systemd管理的/etc/resolv.conf是一个软链接，ip netns直接bind时会失败，将其移除后再创建普通文件类型的/etc/resolv.conf才可bind成功。

```bash
$ sudo mkdir -p /etc/netns/ns1
$ echo 'nameserver 8.8.8.8' | sudo tee /etc/netns/ns1/resolv.conf


$ sudo ip netns exec ns1 ping www.baidu.com
Bind /etc/netns/ns1/resolv.conf -> /etc/resolv.conf failed: No such file or directory
PING www.baidu.com (36.152.44.96) 56(84) bytes of data.
64 bytes from 36.152.44.96 (36.152.44.96): icmp_seq=1 ttl=127 time=36.2 ms
64 bytes from 36.152.44.96 (36.152.44.96): icmp_seq=2 ttl=127 time=37.5 ms

# 创建普通文件类型的/etc/resolv.conf
$ readlink /etc/resolv.conf
../run/systemd/resolve/stub-resolv.conf
$ sudo mv /etc/resolv.conf{,.bak}
$ sudo touch /etc/resolv.conf
$ sudo ip netns exec ns1 dig -t A www.baidu.com
......
;www.baidu.com.                 IN      A

;; ANSWER SECTION:
www.baidu.com.          581     IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       51      IN      CNAME   www.wshifen.com.
www.wshifen.com.        70      IN      A       104.193.88.77
www.wshifen.com.        70      IN      A       104.193.88.123

;; Query time: 215 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)      # 已经成功使用8.8.8.8作为nameserver
;; WHEN: Sun Oct 11 16:58:51 CST 2020
;; MSG SIZE  rcvd: 127m
```