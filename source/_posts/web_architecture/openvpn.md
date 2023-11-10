---
title: 部署openvpn
p: web_architecture/openvpn.md
date: 2019-07-06 17:37:29
tags: Web_architecture
categories: Web_architecture
---

> 注：本文内容不多，且是很早以前写的，只够openvpn初学尝鲜，若不能满足需求，下面两本书随选一本下载：
>
> - [Mastering OpenVPN](/files/Mastering_OpenVPN.pdf)  
> - [OpenVPN Cookbook 2nd](/files/openvpn-cookbook-2nd.pdf)  

# vpn概述

VPN（virtual Private network）即虚拟专用网络，是通过建立一条专用通道进行通信的技术，可以理解为通过建立一条独有的隧道让两边通信。

## vpn应用场景

假设台北总公司有一个上海分公司，两个办公地点都设置了防火墙，内部网络也都使用的私有地址。由于总公司老板经常外出出差，他能直接正常的访问到台北总公司内网的服务器吗？不能。同理上海分公司也不能正常访问到台北总公司内网的服务器。

![](/img/web_architecture/733013-20180317202244410-1376290225.png) 

使用VPN就是为了解决这样问题的。在以前要实现这样的需求，需要找运营商申请帧中继的服务，帧中继就像是找他们拉一条专有的网线，然后两边通过这条专有的线进行通信。但是帧中继是不是万能的，它有优点和缺点。优点是申请多少带宽的服务通信时就一定有多少带宽，不会受到公网的网络阻塞；缺点是很贵，且不方便，试想如果到外国各地出差，如何解决它的流动性以及距离的问题。

VPN就像是把整个因特网虚拟成一个路由器，无论出差的老板身处何处，都能通过这个“路由器”来访问内网的机器。就像是随身带了一根无长度限制、无地理位置限制的网线，随时连接到内网中。而且，现在网速不断增速，加密技术也不断进步，使用VPN已经能够很好的实现上面的需求了。

从上面的描述中，可以总结出VPN的几个用途。  
- (1)client-to-site，个人对企业，就像上面的老板访问企业内部服务器。是客户端对vpn服务器的隧道传输。  
- (2)site-to-site，企业对企业，就像上面上海分公司访问台北总公司的内部服务器。是两边vpn服务器之间的隧道传输。  

即如下图中标注的两种用途。

![](/img/web_architecture/733013-20180317202311571-1768949937.png)

## 隧道协议

有三种隧道协议：PPTP、L2TP、IPSec。

L2TP是第2层隧道协议，是一种工业标准Internet隧道协议。L2TP和PPTP都使用PPP协议对数据进行封装，然后添加独特的包头，之后在在网络上进行传输。PPTP只能在两端点之间建立单一隧道，L2TP支持两端之间使用多隧道，用户可以针对不同的服务质量创建不同的隧道。L2TP可以提供隧道验证，而PPTP不可以。当L2TP或PPTP结合IPSEC共同使用的时候，可以由IPSEC提供隧道验证。

IPSEC（IP Security）是一套协议包而不是一个单独的协议。IPSEC隧道模式是封装、路由与解封的过程。隧道将原始数据包封装在新的数据包内部，新的数据包可能会有新的寻址和路由信息，从而使其能够通过网络进行传输。封装后的数据包到达目的地后，会解除封装然后获得其中的数据内容。

## SSL VPN和IPSEC VPN

ssl vpn是使用SSL数字证书来实现数据加密的VPN。最典型的就是openvpn。它能用于client-to-site，也能用于site-to-site。由于openvpn是C/S架构的工具，所以要求使用vpn服务的客户端安装openvpn客户端程序。

IPSEC VPN主要用于site-to-site，即公司对公司的这种场景，两边公司都布置vpn服务器，两个vpn服务器之间进行隧道传输。常见的ipsec vpn的开源软件是openswan。

对于site-to-site的互联：  
- (1)使用openvpn的实现是在两边都安装openvpn服务和openvpn的客户端，要和对端通信时使用本地的客户端拨号过去;  
- (2)使用openswan的实现只需要两边安装openswan服务就可以，两边的通信是服务与服务的通信。

# openvpn搭建client-to-site的vpn

openvpn的所有数据通信都基于一个单一的端口(默认是1194)，默认使用UDP协议，也可以使用且建议使用TCP协议。

openvpn的核心是虚拟网卡。安装openvpn后会在主机上多出一个网卡，可以像其他的网卡一样进行配置。这个虚拟网卡可以接收和发送数据。

openvpn提供了两种虚拟网络接口：**Tun和Tap**。通过它们分别可以建立三层IP隧道和虚拟2层以太网，即分别**在虚拟接口上分别实现数据包(tun)格式的传输和数据帧(tap)格式的传输。**

传输的数据可以通过压缩算法进行数据压缩后传输。

openvpn2.0以后的版本都能实现一个进程管理多个并发的隧道。

openvpn的身份验证方式支持预享密钥、第三方证书、用户名密码三种方式。

以下图为例，将防火墙当作vpnserver，其中eth0是对外的网卡，eth1是内网地址。

![](/img/web_architecture/733013-20180317203449239-1281643505.png)

由于10.0.10.6处于内网目前无法和外网直接通信，可以通过防火墙的NAT或者将10.6的默认路由指向防火墙使其和外界通信。但此处使用VPN来实现和外界某些主机通信，所以不做任何NAT也不设置默认路由。

现在的环境下，vpn sever能ping通win主机，能ping通10.0.10.6，win不能ping通vpn server的内网地址，10.6也不能ping通win主机。

## 安装lzo和openvpn

由于vpn很可能要在公网上传输数据，不压缩的话传输的数据量就太大，导致传输速度较慢。所以建议安装一种压缩插件。可以使用lzo和lz4。

2.4版本的openvpn已经支持lz4压缩算法，这种压缩算法性能远强于lzo，而且lzo已经停止更新了。以下是几种压缩算法压力测试的结果。

![](/img/web_architecture/733013-20180317220011863-644635737.png)

但是此处还是先使用lzo的算法来演示示例。

```
tar xf lzo-2.09.tar.gz  
cd lzo-2.09
./configure
make && make install
```

安装openvpn。
```
yum -y install openssl-devel pam-devel

wget https://openvpn.net/index.php/open-source/downloads.html
tar xf openvpn-2.4.0.tar.gz
cd openvpn-2.4.0
./configure --enable-lzo
make && make install
```
一定要使用./configure --help查看下安装方法，因为不同版本的openvpn有点不同。默认情况下lzo是被禁用的，所以要启用。

## 创建CA和SSL证书

**(1).自建CA(为了方便，在服务端建立)**

```
(umask 077;openssl genrsa -out /etc/pki/CA/private/cakey.pem 2048)
openssl req -new -x509 -key /etc/pki/CA/private/cakey.pem -out /etc/pki/CA/cacert.pem
```
**(2).为vpn服务端颁发证书(服务端持有)**

为openvpn提供一个目录/etc/openvpn，路径随意，因为在启动openvpn时会使用`--config`选项指定配置文件路径，以后密钥、证书文件以及配置文件都放在此目录下。当然这是非必须的，只是为了方便以及统一化。
```
mkdir -p /etc/openvpn/certs
```

由于VPN server端要使用ca的证书，所以复制ca证书到此目录中，并重命名为ca.crt。

```
cp -a /etc/pki/CA/cacert.pem /etc/openvpn/certs/ca.crt
```

**(3).vpn服务端需要生成证书申请，并使用ca为其颁发证书**

```
(umask 077;openssl genrsa -out /etc/openvpn/certs/server.key 2048)
openssl req -new -key /etc/openvpn/certs/server.key -out /etc/openvpn/certs/server.csr
openssl ca -in /etc/openvpn/certs/server.csr -out /etc/openvpn/certs/server.crt -days 365
```

**(4).为客户端颁发证书(客户端持有)**

```
(umask 077;openssl genrsa -out /etc/openvpn/certs/boss.key 2048)
openssl req -new -key /etc/openvpn/certs/boss.key -out /etc/openvpn/certs/boss.csr
openssl ca -in /etc/openvpn/certs/boss.csr -out /etc/openvpn/certs/boss.crt -days 365
```

**(5).生成密钥交换协议文件(服务端持有)**

```
openssl dhparam -out /etc/openvpn/certs/dh2048.pem 2048
```
## 配置服务端

**(1).为vpn提供配置文件**

openvpn的样例配置文件在openvpn解压目录下的sample目录下的sample-config-files目录下。
```
cd ~/openvpn-2.4.0/sample/sample-config-files/
cp -a server.conf client.conf /etc/openvpn/
```
配置文件参数说明：
- `local 192.168.100.5`：表示openvpn服务端的监听地址  
- `port 1194`：监听的端口，默认是1194  
- `proto tcp`：使用的协议，有udp和tcp。建议选择tcp  
- `dev tun`：使用三层路由IP隧道(tun)还是二层以太网隧道(tap)。一般都使用tun  
- `ca ca.crt`  
- `cert server.crt`  
- `key server.key`  
- `dh dh2048.pem`：ca证书、服务端证书、服务端密钥和密钥交换文件。如果它们和server.conf在同一个目录下则可以不写绝对路径，否则需要写绝对路径调用  
- `server 10.8.0.0 255.255.255.0`：vpn服务端为自己和客户端分配IP的地址池。服务端自己获取网段的第一个地址(此处为10.8.0.1)，后为客户端分配其他的可用地址。以后客户端就可以和10.8.0.1进行通信。注意：该网段地址池不要和已有网段冲突或重复。其实一般来说是不用改的。除非当前内网使用了10.8.0.0/24的网段。  
- `ifconfig-pool-persist ipp.txt`：使用一个文件记录已分配虚拟IP的客户端和虚拟IP的对应关系，以后openvpn重启时，将可以按照此文件继续为对应的客户端分配此前相同的IP。也就是自动续借IP的意思。  
- `server-bridge XXXXXX`：使用tap模式的时候考虑此选项。  
- `push "route 10.0.10.0 255.255.255.0"`：vpn服务端向客户端推送一条客户端到vpn内网网段的路由配置，以便让客户端能够找到内网。多条路由就写多个Push指令  
- `client-to-client`：让vpn客户端之间可以互相看见对方，即能互相通信。默认情况客户端只能看到服务端一个人  
- `duplicate-cn`：允许多个客户端使用同一个VPN帐号连接服务端  
- `keepalive 10 120`：每10秒ping一次，120秒后没收到ping就说明对方挂了  
- `tls-auth ta.key 0`：加强认证方式，防攻击。如果配置文件中启用此项(默认是启用的)，需要执行`openvpn --genkey --secret ta.key`，并把ta.key放到/etc/openvpn目录，服务端第二个参数为0。同时客户端也要有此文件，且client.conf中此指令的第二个参数需要为1。  
- `compress lz4-v2`  
- `push "compress lz4-v2"`：openvpn 2.4版本的vpn才能设置此选项。表示服务端启用lz4的压缩功能，传输数据给客户端时会压缩数据包，Push后在客户端也配置启用lz4的压缩功能，向服务端发数据时也会压缩。如果是2.4版本以下的老版本，则使用用comp-lzo指令  
- `comp-lzo`：启用lzo数据压缩格式。此指令用于低于2.4版本的老版本。且如果服务端配置了该指令，客户端也必须要配置  
- `max-clients 100`：并发客户端的连接数  
- `persist-key`：  
- `persist-tun`:通过ping得知超时时，当重启vpn后将使用同一个密钥文件以及保持tun连接状态  
- `status openvpn-status.log`：在文件中输出当前的连接信息，每分钟截断并重写一次该文件  
- `log openvpn.log`：  
`log-append openvpn.log`：默认vpn的日志会记录到rsyslog中，使用这两个选项可以改变。log指令表示每次启动vpn时覆盖式记录到指定日志文件中，log-append则表示每次启动vpn时追加式的记录到指定日志中。但两者只能选其一，或者不选时记录到rsyslog中  
- `verb 3`：日志记录的详细级别。  

以下是完整的配置文件的内容。
```
cat /etc/openvpn/server.conf
local 192.168.100.5
port 1194
proto tcp
dev tun
ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
key /etc/openvpn/certs/server.key
dh /etc/openvpn/certs/dh2048.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.0.10.0 255.255.255.0"
client-to-client
keepalive 10 120
tls-auth ta.key 0
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log  /var/log/openvpn.log
verb 3
```
由于使用了tls-auth功能，所以要生成ta.key。(客户端和服务端都持有)
```
openvpn --genkey --secret /etc/openvpn/ta.key 
```

**(2).开启服务端内核转发功能**

因为数据包要在vpn内部交换网卡进出，所以要开启转发功能。
```
echo 1 >/proc/sys/net/ipv4/ip_forward
sed -i "s/ip_forward.*$/ip_forward = 1/" /etc/sysctl.conf
```

**(3).启动openvpn**

```
[root@node1 certs]# openvpn --config /etc/openvpn/server.conf &
[1] 32673
[root@node1 certs]# netstat -tnlp | grep 1194
tcp        0      0 192.168.100.5:1194     0.0.0.0:*    LISTEN    32673/openvpn
```
将其放入到rc.local中让它下次开机自启动。
```
echo "`which openvpn` --config /etc/openvpn/server.conf &" >>/etc/rc.d/rc.local
```

## 配置客户端

windows版客户端下载地址：https://openvpn.net/index.php/open-source/downloads.html

安装的时候有一个是否信任的选项，选择信任。假设安装路径为e:/openvpn目录下。安装完成后，从"开始菜单"那里打开客户端。

![](/img/web_architecture/733013-20180317223929381-1230072309.png)

点开它，可能会提示缺少配置文件，暂时不用管。这时在win的通知区域会出现一个小图标。

要真正能够连接到服务端，需要提供**客户端配置文件、ca证书、客户端证书、客户端密钥**。如果在服务端配置文件中启用了tls-auth认证防攻击，则**还需要提供ta.key文件**。
```
sz -y /etc/openvpn/{client.conf,ta.key,certs/boss.key,certs/boss.crt,certs/ca.crt} 
```

将这些文件放到安装目录的config目录下，可以为它们建立一个单独的目录，如boss。

![](/img/web_architecture/733013-20180317224054524-987702695.png)

修改配置文件，客户端的配置文件使用如下的配置。
```
egrep -v "^$|^;|^#" client.conf
client
dev tun
proto tcp
remote 192.168.100.5 1194 
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert boss.crt
key boss.key
remote-cert-tls server
tls-auth ta.key 1
cipher AES-256-CBC
verb 3
```
其中只修改了以下的4项：
```
proto tcp
remote 192.168.100.5 1194  # vpn server的监听地址和端口
cert boss.crt
key boss.key
```

并删除了remote-cert-tls server项，此项是为了防止黑客攻击的。所以完整的客户端配置文件为：
```
client
dev tun
proto tcp
remote 192.168.100.5 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
tls-auth ta.key 1
cert boss.crt
key boss.key
cipher AES-256-CBC
verb 3
```
然后将配置文件重命名为".ovpn"后缀，如改为boss.ovpn。然后在通知区域vpn的小图标上右键选择connect。

![](/img/web_architecture/733013-20180317224300437-732779293.png)

连接成功的话会出现如下图分配地址的情形。如果始终连接不上，查看下vpn server是不是开启了防火墙，如果是，就暂时关闭它。

![](/img/web_architecture/733013-20180317224334835-89569628.png)

尽管成功了，但是还是要检验下是否能够ping通vpn服务端的虚拟地址10.8.0.1。

## 查看vpn的连接状态

在status日志中记录了连接状态信息。
```
[root@xuexi openvpn]# cat /var/log/openvpn-status.log 
OpenVPN CLIENT LIST
Updated,Mon Feb 27 18:44:38 2017
Common Name,Real Address,Bytes Received,Bytes Sent,Connected Since
boss.longshuai.com,192.168.100.1:4554,14902,7872,Mon Feb 27 18:28:04 2017
ROUTING TABLE
Virtual Address,Common Name,Real Address,Last Ref
10.8.0.6,boss.longshuai.com,192.168.100.1:4554,Mon Feb 27 18:43:36 2017
GLOBAL STATS
Max bcast/mcast queue length,0
END
```

## 让客户端和vpn所在内网通信

虽然上面已经能够ping通vpn服务端了，但是并不代表能Ping通vpn服务器所在的内网机器。继续看下面的图。

![](/img/web_architecture/733013-20180317225017911-1689438076.png)

由于客户端已经使用10.8.0.6这个地址，在此客户端上有一条到10.0.10.0网段的路由（在配置文件中push指令指定的），还有到10.8.0.0网段的路由，这几条路由都指明了到这几个网段都的下一跳都是交给10.8.0.5，这是VPN服务器生成的网关地址。
所以，客户端要Ping通10.0.10.0/24网段，那么数据包就会先交到10.8.0.1这个tun0上，然后交到10.0.10.0/24这个网段的主机上，但是回应的时候数据包却回不来，除非在私有网络的机器上配置了到10.8.0.0网段的路由，否则它不知道10.8.0.6的ping包应该交给谁。

所以私网内的主机要能与10.8.0.6通信，方法一是加一条到10.8.0.0网段的路由，甚至针对此客户端主机来添加主机路由，它们的下一跳要交给vpn server的内网地址。
```
route add -host 10.8.0.6 gw 10.0.10.5
# 或者
route add -net 10.8.0.0/24 gw 10.0.10.5
```
以下是route -n的结果。

![](/img/web_architecture/733013-20180317225048158-2109021488.png)

这时候vpn客户端就能和私网内的主机通信了，但要注意，此时私网内主机不能和客户端通信。

显然，这样的方法并不合适。这里说明这种方法是为了解释双方通信过程中的本质。

## 使用NAT让客户端和VPN所在内网通信

为了让vpn服务端私网内的机器可以把数据包交回给vpn服务器，可以在vpn服务器上使用网络地址转换(NAT)功能，让源地址为10.8.0.0网段的机器发过来的数据包的源地址全部改变为自己的内网IP地址，即10.0.10.5，这样私网内的机器回应数据包时就会回应给10.0.10.5，然后vpn服务器就可以将数据包还给vpn客户端，实现通信的目的。
```
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth1 -j SNAT --to-source 10.0.10.5
```
到此为止，配置site-to-client的vpn就配置完成了。

# 使用密码拨号以及帐号吊销

**(1).使用密码拨号**

要让客户端使用密码才能拨号，需要做的是在生成客户端证书申请文件(csr)的时候使用一个密码。

```
openssl req -newkey rsa:2048 -keyout shuai.key -out shuai.csr
```

![](/img/web_architecture/733013-20180317225233858-1671721327.png)

```
openssl ca -in shuai.csr -out shuai.crt -days 365
```

然后再将ca.crt、ta.key、shuai.crt、shuai.key、client.conf传到客户端，重复前面部署客户端的步骤。最后在openvpn的图标处右键会有两个拨号选择。

![](/img/web_architecture/733013-20180317225348421-11033430.png)

![](/img/web_architecture/733013-20180317225343143-2079778862.png)

**(2).吊销帐号**

吊销拨号的帐号的本质是吊销客户端证书。

吊销证书的命令是：
```
openssl ca -revoke crt_file
```
生成吊销列表的命令是：
```
openssl ca -gencrl -out crl_file
```
假设要吊销shuai客户端的证书。
```
openssl ca -revoke /etc/openvpn/certs/shuai.crt
```
然后生成吊销列表：
```
openssl ca -gencrl -out /etc/openvpn/certs/shuai.crl
```
如果使用的是openssl的默认配置文件，则在生成吊销列表的时候可能会报错提示找不到crlnumber文件。可以创建此文件，并向其中写入一个序列号，然后再去生成吊销列表。
```
echo "01" >/etc/pki/CA/crlnumber
```
生成吊销列表后，只要服务端的配置文件，加上crl-verify指令然后重启openvpn服务就可以吊销了。如果要吊销多个帐号，则写多个crl-verify即可。
```
vim server.conf 
local 192.168.100.5
port 1194
proto tcp
dev tun
ca /etc/openvpn/certs/ca.crt
cert  /etc/openvpn/certs/server.crt
key  /etc/openvpn/certs/server.key  # This file should be kept secret
dh  /etc/openvpn/certs/dh2048.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.0.10.0 255.255.255.0"
;duplicate-cn
keepalive 10 120
tls-auth ta.key 0
;compress lz4-v2
;push "compress lz4-v2"
comp-lzo
;max-clients 100
persist-key
persist-tun
status openvpn-status.log
log         openvpn.log
verb 3
crl-verify /etc/openvpn/certs/shuai.crl
```
再重启openvpn。
```
pkill openvpn
openvpn --config /etc/openvpn/server.conf &
```
这时客户端拨号就会如下图，无限重连状态。

![](/img/web_architecture/733013-20180317225618241-417542158.png)

# Linux主机上配置openvpn客户端

要在Linux主机上配置openvpn的客户端，其实也很简单，安装完后把客户端需要持有的文件拷贝过去然后启动即可。

以下是环境需求，以上海分公司的防火墙暂时当作openvpn的客户端，IP地址是192.168.100.151，确保它能和vpn server的eth0通信。

![](/img/web_architecture/733013-20180317225644772-1969351083.png)

安装过程和安装服务端vpn过程相同，此处给出简易步骤。
```
yum -y install openssl-devel pam-devel lzo* lz4*
tar xf openvpn-2.4.0.tar.gz
cd openvpn-2.4.0
./configure --enable-lz4 --enable-lzo
make && make install
mkdir -p /etc/openvpn/certs
```
再提供配置文件和相关证书、密钥等。
```
scp -p root@192.168.100.5:/etc/openvpn/ta.key /etc/openvpn/
scp -p root@192.168.100.5:/etc/openvpn/certs/{boss.*,ca.crt} /etc/openvpn/certs/
cat >/etc/openvpn/boss.conf<<eof
client
dev tun
proto tcp
remote 192.168.100.5 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca    /etc/openvpn/certs/ca.crt
cert   /etc/openvpn/certs/boss.crt
key   /etc/openvpn/certs/boss.key
tls-auth /etc/openvpn/ta.key 1
cipher AES-256-CBC
comp-lzo
verb 3
eof
```
启动vpn客户端。会有些消息输出在屏幕，多按几次enter键就可以回到bash提示符下。
```
openvpn --config /etc/openvpn/boss.conf & 
```
查看网卡信息是否有tun网卡。
```
ifconfig tun0
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:10.8.0.6  P-t-P:10.8.0.5  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:2 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100 
          RX bytes:168 (168.0 b)  TX bytes:168 (168.0 b)
```
测试是否能ping通vpn server的内网机器10.0.10.6。
```
ping 10.0.10.6
PING 10.0.10.6 (10.0.10.6) 56(84) bytes of data.
64 bytes from 10.0.10.6: icmp_seq=1 ttl=63 time=3.07 ms
64 bytes from 10.0.10.6: icmp_seq=2 ttl=63 time=2.21 ms
64 bytes from 10.0.10.6: icmp_seq=3 ttl=63 time=2.19 ms
```


