---
title: ssh和ssh服务
p: linux/ssh.md
date: 2021-03-15 14:17:18
tags: Linux
categories: Linux
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

SSH系列文章：  
[SSH基础：SSH和SSH服务](/linux/ssh)  
[SSH转发代理：ssh-agent用法详解](/linux/ssh_agent)  
[SSH隧道：端口转发功能详解](/linux/ssh_port_forward)  

------

# ssh和ssh服务

本文对SSH连接验证机制进行了非常详细的分析，还详细介绍了ssh客户端工具的各种功能，相信能让各位对ssh有个全方位较透彻的了解，而不是仅仅只会用它来连接远程主机。

另外，本人翻译了ssh客户端命令的man文档，如本文有不理解的地方，可以参考man文档手册：</linux/man_ssh_translate>。

## 1.非对称加密基础知识

**对称加密：加密和解密使用一样的算法，只要解密时提供与加密时一致的密码就可以完成解密**。例如QQ登录密码，银行卡密码，只要保证密码正确就可以。

**非对称加密：通过公钥(public key)和私钥(private key)来加密、解密。公钥加密的内容可以使用私钥解密，私钥加密的内容可以使用公钥解密**。一般使用公钥加密，私钥解密，但并非绝对如此，例如CA签署证书时就是使用自己的私钥加密。在接下来介绍的SSH服务中，虽然一直建议分发公钥，但也可以分发私钥。

所以，如果A生成了(私钥A，公钥A)，B生成了(私钥B，公钥B)，那么A和B之间的非对称加密会话情形包括：  
- (1).A将自己的公钥A分发给B，B拿着公钥A将数据进行加密，并将加密的数据发送给A，A将使用自己的私钥A解密数据。  
- (2).A将自己的公钥A分发给B，并使用自己的私钥A加密数据，然后B使用公钥A解密数据。  
- (3).B将自己的公钥B分发给A，A拿着公钥B将数据进行加密，并将加密的数据发送给B，B将使用自己的私钥B解密数据。  
- (4).B将自己的公钥B分发给A，并使用自己的私钥B加密数据，然后A使用公钥B解密数据。  

虽然理论上支持4种情形，但在SSH的**身份验证阶段，SSH只支持服务端保留公钥，客户端保留私钥的方式**，所以方式只有两种：客户端生成密钥对，将公钥分发给服务端；服务端生成密钥对，将私钥分发给客户端。只不过出于安全性和便利性，一般都是客户端生成密钥对并分发公钥。后文将给出这两种分发方式的示例。

## 2.SSH概要

1. SSH是传输层和应用层上的安全协议，它只能通过加密连接双方会话的方式来保证连接的安全性。当使用ssh连接成功后，将建立客户端和服务端之间的会话，该会话是被加密的，之后客户端和服务端的通信都将通过会话传输。  
2. SSH服务的守护进程为sshd，默认监听在22端口上。  
3. 所有ssh客户端工具，包括ssh命令，scp，sftp，ssh-copy-id等命令都是借助于ssh连接来完成任务的。也就是说它们都连接服务端的22端口，只不过连接上之后将待执行的相关命令转换传送到远程主机上，由远程主机执行。  
4. ssh客户端命令(ssh、scp、sftp等)读取两个配置文件：全局配置文件`/etc/ssh/ssh_config`和用户配置文件`~/.ssh/config`。实际上命令行上也可以传递配置选项。它们生效的优先级是：`命令行配置选项 > ~/.ssh/config > /etc/ssh/ssh_config`。  
5. ssh涉及到两个验证：主机验证和用户身份验证。通过主机验证，再通过该主机上的用户验证，就能唯一确定该用户的身份。一个主机上可以有很多用户，所以每台主机的验证只需一次，但主机上每个用户都需要单独进行用户验证。  
6. ssh支持多种身份验证，最常用的是密码验证机制和公钥认证机制，其中公钥认证机制在某些场景实现双机互信时几乎是必须的。虽然常用上述两种认证机制，但认证时的顺序默认是gssapi-with-mic,hostbased,publickey,keyboard-interactive,password。注意其中的主机认证机制hostbased不是主机验证，由于主机认证用的非常少(它所读取的认证文件为/etc/hosts.equiv或/etc/shosts.equiv)，所以网络上比较少见到它的相关介绍。总的来说，通过在ssh配置文件(注意不是sshd配置文件)中使用指令PreferredAuthentications改变认证顺序不失为一种验证的效率提升方式。  
7. ssh客户端其实有不少很强大的功能，如端口转发(隧道模式)、代理认证、连接共享(连接复用)等。  
8. ssh服务端配置文件为`/etc/ssh/sshd_config`，注意和客户端的全局配置文件`/etc/ssh/ssh_config`区分开来。  
9. 最重要的一点，**ssh登录时会请求分配一个伪终端(仅执行远程命令不会申请分配终端，但可以使用"-tt"选项来强制请求)**。但有些身份认证程序如sudo可以禁止这种类型的终端分配，导致ssh连接失败。例如使用ssh执行sudo命令时sudo就会验证是否要分配终端给ssh。  

## 3.SSH认证过程分析
假如从客户端A(172.16.10.5)连接到服务端B(172.16.10.6)上，将包括主机验证和用户身份验证两个过程，以RSA非对称加密算法为例。
```
[root@xuexi ~]# ssh 172.16.10.6 
```
服务端B上首先启动了sshd服务程序，即开启了ssh服务，打开了22端口(默认)。

<a name="host_auth"></a>

### 3.1 主机验证过程

当客户端A要连接B时，首先将进行主机验证过程，即判断主机B是否是否曾经连接过。

判断的方法是**读取`~/.ssh/known_hosts`文件和`/etc/ssh/known_hosts`文件，搜索是否有172.16.10.6的主机信息(主机信息称为host key，表示主机身份标识)。如果没有搜索到对应该地址的host key，则询问是否保存主机B发送过来的host key，如果搜索到了该地址的host key，则将此host key和主机B发送过来的host key做比对，如果完全相同，则表示主机A曾经保存过主机B的host key，无需再保存，直接进入下一个过程——身份验证，如果不完全相同，则提示是否保存主机B当前使用的host key**。

询问是否保存host key的过程如下所示：
```
[root@xuexi ~]# ssh 172.16.10.6  
The authenticity of host '172.16.10.6 (172.16.10.6)' can't be established.
RSA key fingerprint is f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf.
Are you sure you want to continue connecting (yes/no)? yes
```
或者windows端使用图形界面ssh客户端工具时：

![](/img/linux/733013-20170706225637503-2146670509.png)

在说明身份验证过程前，先看下`known_hosts`文件的格式。以`~/.ssh/known_hosts`为例。
```
[root@xuexi ~]# cat ~/.ssh/known_hosts 
172.16.10.6 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC675dv1w+GDYViXxqlTspUHsQjargFPSnR9nEqCyUgm5/32jXAA3XTJ4LUGcDHBuQ3p3spW/eO5hAP9eeTv5HQzTSlykwsu9He9w3ee+TV0JjBFulfBR0weLE4ut0PurPMbthE7jIn7FVDoLqc6o64WvN8LXssPDr8WcwvARmwE7pYudmhnBIMPV/q8iLMKfquREbhdtGLzJRL9DrnO9NNKB/EeEC56GY2t76p9ThOB6ES6e/87co2HjswLGTWmPpiqY8K/LA0LbVvqRrQ05+vNoNIdEfk4MXRn/IhwAh6j46oGelMxeTaXYC+r2kVELV0EvYV/wMa8QHbFPSM6nLz
```
该文件中，每行一个host key，**行首是主机名，它是搜索host key时的索引**，主机名后的内容即是host key部分。以此文件为例，它表示客户端A曾经试图连接过172.16.10.6这个主机B，并保存了主机B的host key，下次连接主机B时，将搜索主机B的host key，并与172.16.10.6传送过来的host key做比较，如果能匹配上，则表示该host key确实是172.16.10.6当前使用的host key，如果不能匹配上，则表示172.16.10.6修改过host key，或者此文件中的host key被修改过。

那么主机B当前使用的host key保存在哪呢？在`/etc/ssh/ssh_host*`文件中，这些文件是服务端(此处即主机B)的sshd服务程序启动时重建的。以rsa算法为例，则保存在`/etc/ssh/ssh_host_rsa_key`和`/etc/ssh/ssh_host_rsa_key.pub`中，其中公钥文件`/etc/ssh/ssh_host_rsa_key.pub`中保存的就是host key。
```
[root@xuexi ~]# cat /etc/ssh/ssh_host_rsa_key.pub   # 在主机B上查看
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC675dv1w+GDYViXxqlTspUHsQjargFPSnR9nEqCyUgm5/32jXAA3XTJ4LUGcDHBuQ3p3spW/eO5hAP9eeTv5HQzTSlykwsu9He9w3ee+TV0JjBFulfBR0weLE4ut0PurPMbthE7jIn7FVDoLqc6o64WvN8LXssPDr8WcwvARmwE7pYudmhnBIMPV/q8iLMKfquREbhdtGLzJRL9DrnO9NNKB/EeEC56GY2t76p9ThOB6ES6e/87co2HjswLGTWmPpiqY8K/LA0LbVvqRrQ05+vNoNIdEfk4MXRn/IhwAh6j46oGelMxeTaXYC+r2kVELV0EvYV/wMa8QHbFPSM6nLz
```
发现`/etc/ssh/ssh_host_rsa_key.pub`文件内容和`~/.ssh/known_hosts`中该主机的host key部分完全一致，只不过`~/.ssh/known_hosts`中除了host key部分还多了一个主机名，这正是搜索主机时的索引。

综上所述，**在主机验证阶段，服务端持有的是私钥，客户端保存的是来自于服务端的公钥**。注意，这和身份验证阶段密钥的持有方是相反的。
实际上，ssh并非直接比对host key，因为host key太长了，比对效率较低。所以ssh将host key转换成host key指纹，然后比对两边的host key指纹即可。指纹格式如下：

```
[root@xuexi ~]# ssh 172.16.10.6  
The authenticity of host '172.16.10.6 (172.16.10.6)' can't be established.
RSA key fingerprint is f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf.
Are you sure you want to continue connecting (yes/no)? yes
```
host key的指纹可由ssh-kegen计算得出。例如，下面分别是主机A(172.16.10.5)保存的host key指纹，和主机B(172.16.10.6)当前使用的host key的指纹。可见它们是完全一样的。
```
[root@xuexi ~]# ssh-keygen -l -f ~/.ssh/known_hosts 
2048 f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf 172.16.10.6 (RSA)

[root@xuexi ~]# ssh-keygen -l -f /etc/ssh/ssh_host_rsa_key
2048 f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf   (RSA)
```
其实ssh还支持host key模糊比较，即将host key转换为图形化的指纹。这样，图形结果相差大的很容易就比较出来。之所以说是模糊比较，是因为对于非常近似的图形化指纹，ssh可能会误判。图形化指纹的生成方式如下：只需在上述命令上加一个"-v"选项进入详细模式即可。
```
[root@xuexi ~]# ssh-keygen -lv -f ~/.ssh/known_hosts
2048 f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf 172.16.10.6 (RSA)
+--[ RSA 2048]----+
|                 |
|                 |
|           .     |
|          o      |
|        S. . +   |
|      . +++ + .  |
|       B.+.= .   |
|      + B.  +.   |
|       o.+.  oE  |
+-----------------+
```
更详细的主机认证过程是：先进行密钥交换(DH算法)生成session key(rfc文档中称之为shared secret)，然后从文件中读取host key，并用host key对session key进行签名，然后对签名后的指纹进行判断。(In SSH, the key exchange is signed with the host key to provide host authentication.来源：<https://tools.ietf.org/html/rfc4419>)

过程如下图：

![](/img/linux/733013-20180909221238150-2018158097.png)

<a name="ssh_auth"></a>

### 3.2 身份验证过程

主机验证通过后，将进入身份验证阶段。SSH支持多种身份验证机制，它们的**验证顺序如下：gssapi-with-mic,hostbased,publickey,keyboard-interactive,password**，但常见的是密码认证机制(password)和公钥认证机制(public key)。当公钥认证机制未通过时，再进行密码认证机制的验证。这些认证顺序可以通过ssh配置文件(注意，不是sshd的配置文件)中的指令PreferredAuthentications改变。

如果使用公钥认证机制，**客户端A需要将自己生成的公钥(`~/.ssh/id_rsa.pub`)发送到服务端B的`~/.ssh/authorized_keys`文件中**。当进行公钥认证时，客户端将告诉服务端要使用哪个密钥对，并告诉服务端它已经访问过密钥对的私钥部分~/.ssh/id_rsa(客户端从自己的私钥中推导，或者从私钥同目录下读取公钥，**计算公钥指纹后发送给服务端**。所以有些版本的ssh不要求存在公钥文件，有些版本的ssh则要求私钥和公钥同时存在且在同目录下)，然后服务端将检测密钥对的公钥部分，判断该客户端是否允许通过认证。如果认证不通过，则进入下一个认证机制，以密码认证机制为例。

当使用密码认证时，将提示输入要连接的远程用户的密码，输入正确则验证通过。

### 3.3 验证通过

当主机验证和身份验证都通过后，分两种情况：直接登录或执行ssh命令行中给定某个命令。如：
```
[root@xuexi ~]# ssh 172.16.10.6  
[root@xuexi ~]# ssh 172.16.10.6  'echo "haha"'
```
(1).前者ssh命令行不带任何命令参数，表示使用远程主机上的某个用户(此处为root用户)登录到远程主机172.16.10.6上，所以远程主机会为ssh分配一个伪终端，并进入bash环境。

(2).后者ssh命令行带有命令参数，表示在远程主机上执行给定的命令`echo "haha"`。ssh命令行上的远程命令是通过fork ssh-agent得到的子进程来执行的，当命令执行完毕，子进程消逝，ssh也将退出，建立的会话和连接也都将关闭。(之所以要在这里明确说明远程命令的执行过程，是为了说明后文将介绍的ssh实现端口转发时的注意事项)

实际上，在ssh连接成功，登录或执行命令行中命令之前，可以指定要在远程执行的命令，这些命令放在`~/.ssh/rc`或`/etc/ssh/rc`文件中，也就是说，ssh连接建立之后做的第一件事是在远程主机上执行这两个文件中的命令。


## 4.各种文件分布

以主机A连接主机B为例，主机A为SSH客户端，主机B为SSH服务端。

**在服务端即主机B上**：  

`/etc/ssh/sshd_config`  
ssh服务程序sshd的配置文件。

`/etc/ssh/ssh_host_*`  
服务程序sshd启动时生成的服务端公钥和私钥文件。如`ssh_host_rsa_key`和`ssh_host_rsa_key.pub`。其中.pub文件是主机验证时的host key，将写入到客户端的`~/.ssh/known_hosts`文件中。其中**私钥文件严格要求权限为600，若不是则sshd服务可能会拒绝启动**。

`~/.ssh/authorized_keys`  
保存的是基于公钥认证机制时来自于客户端的公钥。在基于公钥认证机制认证时，服务端将读取该文件。

**在客户端即主机A上**：  

`/etc/ssh/ssh_config`  
客户端的全局配置文件。

`~/.ssh/config`  
客户端的用户配置文件，生效优先级高于全局配置文件。一般该文件默认不存在。该文件对权限有严格要求只对所有者有读/写权限，对其他人完全拒绝写权限。

`~/.ssh/known_hosts`  
保存主机验证时服务端主机host key的文件。文件内容来源于服务端的`ssh_host_rsa_key.pub`文件。

`/etc/ssh/known_hosts`  
全局host key保存文件。作用等同于`~/.ssh/known_hosts`。

`~/.ssh/id_rsa`  
客户端生成的私钥。由ssh-keygen生成。**该文件严格要求权限，当其他用户对此文件有可读权限时，ssh将直接忽略该文件**。

`~/.ssh/id_rsa.pub`  
私钥id_rsa的配对公钥。对权限不敏感。当采用公钥认证机制时，该文件内容需要复制到服务端的~/.ssh/authorized_keys文件中。

`~/.ssh/rc`  
保存的是命令列表，这些命令在ssh连接到远程主机成功时将第一时间执行，执行完这些命令之后才开始登陆或执行ssh命令行中的命令。

`/etc/ssh/rc`  
作用等同于`~/.ssh/rc`。

## 5.配置文件简单介绍

分为服务端配置文件`/etc/ssh/sshd_config`和客户端配置文件`/etc/ssh/ssh_config`(全局)或`~/.ssh/config`(用户)。

虽然服务端和客户端配置文件默认已配置项虽然非常少非常简单，但它们可配置项非常多。`sshd_config`完整配置项参见金步国翻译的[sshd_config中文手册](http://www.jinbuguo.com/openssh/sshd_config.html)，`ssh_config`也可以参考`sshd_config`的配置，它们大部分配置项所描述的内容是相同的。

### 5.1 sshd_config

简单介绍下该文件中比较常见的指令。
```
[root@xuexi ~]# cat /etc/ssh/sshd_config
#Port 22                # 服务端SSH端口，可以指定多条表示监听在多个端口上
#ListenAddress 0.0.0.0  # 监听的IP地址。0.0.0.0表示监听所有IP
Protocol 2              # 使用SSH 2版本

#####################################
#          私钥保存位置             #
#####################################
# HostKey for protocol version 1
#HostKey /etc/ssh/ssh_host_key      # SSH 1保存位置/etc/ssh/ssh_host_key
# HostKeys for protocol version 2
#HostKey /etc/ssh/ssh_host_rsa_key  # SSH 2保存RSA位置/etc/ssh/ssh_host_rsa _key
#HostKey /etc/ssh/ssh_host_dsa_key  # SSH 2保存DSA位置/etc/ssh/ssh_host_dsa _key


###################################
#           杂项配置              #
###################################
#PidFile /var/run/sshd.pid        # 服务程序sshd的PID的文件路径
#ServerKeyBits 1024               # 服务器生成的密钥长度
#SyslogFacility AUTH              # 使用哪个syslog设施记录ssh日志。日志路径默认为/var/log/secure
#LogLevel INFO                    # 记录SSH的日志级别为INFO

###################################
#   以下项影响认证速度             #
###################################
#UseDNS yes                       # 指定是否将客户端主机名解析为IP，以检查此主机名是否与其IP地址真实对应。默认yes。
                                  # 由此可知该项影响的是主机验证阶段。建议在未配置DNS解析时，将其设置为no，否则主机验证阶段会很慢

###################################
#   以下是和安全有关的配置        #
###################################
#PermitRootLogin yes              # 是否允许root用户登录
#GSSAPIAuthentication no          # 是否开启GSSAPI身份认证机制，默认为yes 
#PubkeyAuthentication yes         # 是否开启基于公钥认证机制
#AuthorizedKeysFile  .ssh/authorized_keys  # 基于公钥认证机制时，来自客户端的公钥的存放位置
PasswordAuthentication yes        # 是否使用密码验证，如果使用密钥对验证可以关了它
#PermitEmptyPasswords no          # 是否允许空密码，如果上面的那项是yes，这里最好设置no
#MaxSessions 10                   # 最大客户端连接数量
#LoginGraceTime 2m                # 身份验证阶段的超时时间，若在此超时期间内未完成身份验证将自动断开
#MaxAuthTries 6                   # 指定每个连接最大允许的认证次数。默认值是6。
                                  # 如果失败认证次数超过该值一半，将被强制断开，且生成额外日志消息。
MaxStartups 10                    # 最大允许保持多少个未认证的连接。默认值10。

###################################
#   以下可以自行添加到配置文件    #
###################################
DenyGroups  hellogroup testgroup  # 表示hellogroup和testgroup组中的成员不允许使用sshd服务，即拒绝这些用户连接
DenyUsers	hello test            # 表示用户hello和test不能使用sshd服务，即拒绝这些用户连接

###################################
#   以下一项和远程端口转发有关    #
###################################
#GatewayPorts no                  # 设置为yes表示sshd允许被远程主机所设置的本地转发端口绑定在非环回地址上
                                  # 默认值为no，表示远程主机设置的本地转发端口只能绑定在环回地址上，见后文"远程端口转发"
```
一般来说，如非有特殊需求，只需修改下监听端口和UseDNS为no以加快主机验证阶段的速度即可。

配置好后直接重启启动sshd服务即可。
```
[root@xuexi ~]# service sshd restart
```

### 5.2 ssh_config

需要说明的是，客户端配置文件有很多配置项和服务端配置项名称相同，但它们一个是在连接时采取的配置(客户端配置文件)，一个是sshd启动时开关性的设置(服务端配置文件)。例如，两配置文件都有GSSAPIAuthentication项，在客户端将其设置为no，表示连接时将直接跳过该身份验证机制，而在服务端设置为no则表示sshd启动时不开启GSSAPI身份验证的机制。即使客户端使用了GSSAPI认证机制，只要服务端没有开启，就绝对不可能认证通过。

下面也简单介绍该文件。
```
# Host *              # Host指令是ssh_config中最重要的指令，只有ssh连接的目标主机名能匹配此处给定模式时，
                      # 下面一系列配置项直到出现下一个Host指令才对此次连接生效
#   ForwardAgent no
#   ForwardX11 no
#   RhostsRSAAuthentication no
#   RSAAuthentication yes
#   PasswordAuthentication yes     # 是否启用基于密码的身份认证机制
#   HostbasedAuthentication no     # 是否启用基于主机的身份认证机制
#   GSSAPIAuthentication no        # 是否启用基于GSSAPI的身份认证机制
#   GSSAPIDelegateCredentials no
#   GSSAPIKeyExchange no
#   GSSAPITrustDNS no
#   BatchMode no          # 如果设置为"yes"，将禁止passphrase/password询问。比较适用于
                          # 在那些不需要询问提供密码的脚本或批处理任务任务中。默认为"no"
#   CheckHostIP yes
#   AddressFamily any
#   ConnectTimeout 0
#   StrictHostKeyChecking ask   # 设置为"yes"，ssh将从不自动添加host key到~/.ssh/known_hosts文件，
                                # 且拒绝连接那些未知主机(即未保存host key的主机或host key已改变的主机)。
                                # 它将强制用户手动添加host key到~/.ssh/known_hosts中。
                                # 设置为ask将询问是否保存到~/.ssh/known_hosts文件。
                                # 设置为no将自动添加到~/.ssh/known_hosts文件。
#   IdentityFile ~/.ssh/identity     # ssh v1版使用的私钥文件
#   IdentityFile ~/.ssh/id_rsa       # ssh v2使用的rsa算法的私钥文件
#   IdentityFile ~/.ssh/id_dsa       # ssh v2使用的dsa算法的私钥文件
#   Port 22                          # 当命令行中不指定端口时，默认连接的远程主机上的端口
#   Protocol 2,1
#   Cipher 3des                      # 指定ssh v1版本中加密会话时使用的加密协议
#   Ciphers aes128-ctr,aes192-ctr,aes256-ctr,arcfour256,arcfour128,aes128-cbc,3des-cbc  # 指定ssh v1版本中加密会话时使用的加密协议
#   MACs hmac-md5,hmac-sha1,umac-64@openssh.com,hmac-ripemd160
#   EscapeChar ~
#   Tunnel no
#   TunnelDevice any:any
#   PermitLocalCommand no    # 功能等价于~/.ssh/rc，表示是否允许ssh连接成功后在本地执行LocalCommand指令指定的命令。
#   LocalCommand    # 指定连接成功后要在本地执行的命令列表，当PermitLocalCommand设置为no时将自动忽略该配置
                    # %d表本地用户家目录，%h表示远程主机名，%l表示本地主机名，%n表示命令行上提供的主机名，
                    # p%表示远程ssh端口，r%表示远程用户名，u%表示本地用户名。
#   VisualHostKey no         # 是否开启主机验证阶段时host key的图形化指纹
Host *
        GSSAPIAuthentication yes
```
如非有特殊需求，ssh客户端配置文件一般只需修改下GSSAPIAuthentication的值为no来改善下用户验证的速度即可，另外在有非交互需求时，将StrictHostKeyChecking设置为no以让主机自动添加host key。


## 6.ssh命令简单功能

语法：
```
ssh [options] [user@]hostname [command]

参数说明：
-b bind_address ：在本地主机上绑定用于ssh连接的地址，当系统有多个ip时才生效。 
-E log_file     ：将debug日志写入到log_file中，而不是默认的标准错误输出stderr。
-F configfile   ：指定用户配置文件，默认为~/.ssh/config。
-f              ：请求ssh在工作在后台模式。该选项隐含了"-n"选项，所以标准输入将变为/dev/null。
-i identity_file：指定公钥认证时要读取的私钥文件。默认为~/.ssh/id_rsa。
-l login_name   ：指定登录在远程机器上的用户名。也可以在全局配置文件中设置。
-N              ：显式指明ssh不执行远程命令。一般用于端口转发，见后文端口转发的示例分析。
-n              ：将/dev/null作为标准输入stdin，可以防止从标准输入中读取内容。ssh在后台运行时默认该项。
-p port         ：指定要连接远程主机上哪个端口，也可在全局配置文件中指定默认的连接端口。
-q              ：静默模式。大多数警告信息将不输出。
-T              ：禁止为ssh分配伪终端。
-t              ：强制分配伪终端，重复使用该选项"-tt"将进一步强制。
-v              ：详细模式，将输出debug消息，可用于调试。"-vvv"可更详细。
-V              ：显示版本号并退出。
-o              ：指定额外选项，选项非常多。

user@hostname   ：指定ssh以远程主机hostname上的用户user连接到的远程主机上，若省略user部分，则表示使
                ：用本地当前用户。如果在hostname上不存在user用户，则连接将失败(将不断进行身份验证)。
command         ：要在远程主机上执行的命令。指定该参数时，ssh的行为将不再
                ：是登录，而是执行命令，命令执行完毕时ssh连接就关闭。
```
例如，以172.16.10.6主机上的longshuai用户登录172.16.10.6。
```
[root@xuexi ~]# ssh longshuai@172.16.10.6
The authenticity of host '172.16.10.6 (172.16.10.6)' can't be established.
ECDSA key fingerprint is 18:d1:28:1b:99:3b:db:20:c7:68:0a:f8:9e:43:e8:b4.
Are you sure you want to continue connecting (yes/no)? yes       # 主机验证
Warning: Permanently added '172.16.10.6' (ECDSA) to the list of known hosts.
longshuai@172.16.10.6's password:                      # 用户验证
Last login: Wed Jul  5 12:27:29 2017 from 172.16.10.6
```
此时已经登录到了172.16.10.6主机上。
```
[longshuai@xuexi ~]$ hostname -I
172.16.10.6
```
要退出ssh登录，使用logout命令或exit命令即可返回到原主机环境。

使用ssh还可以实现主机跳转，即跳板功能。例如主机B能和A、C通信，但A、C之间不同通信，即`A<-->B<-->C<-x->A`的情形。如果要从A登陆到C，则可以借助B这个跳板登录到C。此处一个简单示例为：从172.16.10.5登录到172.16.10.6，再以此为基础登录到172.16.10.3上。
```
[root@xuexi ~]# ssh 172.16.10.6
The authenticity of host '172.16.10.6 (172.16.10.6)' can't be established.
RSA key fingerprint is f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.10.6' (RSA) to the list of known hosts.
Last login: Wed Jul  5 12:36:51 2017 from 172.16.10.6

[root@xuexi ~]# ssh 172.16.10.3
The authenticity of host '172.16.10.3 (172.16.10.3)' can't be established.
ECDSA key fingerprint is 18:d1:28:1b:99:3b:db:20:c7:68:0a:f8:9e:43:e8:b4.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.10.3' (ECDSA) to the list of known hosts.
root@172.16.10.3's password: 
Last login: Thu Jun 29 12:38:56 2017 from 172.16.10.6

[root@xuexi ~]# hostname -I
172.16.10.3 172.16.10.4 
```
同样，在退出时，也是一层一层退出的。
```
[root@xuexi ~]# exit
logout
Connection to 172.16.10.3 closed.

[root@xuexi ~]# hostname -I    
172.16.10.6 

[root@xuexi ~]# exit
logout
Connection to 172.16.10.6 closed.
```
注意，由于在借助172.16.10.6当跳板连接到172.16.10.3，所以172.16.10.3的host key是添加到172.16.10.3上而非172.16.10.5上的。

如果在命令行给出了要执行的命令，默认ssh将工作在前台，如果同时给定了"-f"选项，则ssh工作在后台。但尽管是工作在后台，当远程执行的命令如果有消息返回时，将随时可能显示在本地。当远程命令执行完毕后，ssh连接也将立即关闭。
```
[root@xuexi ~]# ssh 172.16.10.6 'sleep 5'     # 在前台睡眠5秒钟
[root@xuexi ~]# ssh 172.16.10.6 -f 'sleep 5;echo over'   # 在后台睡眠5秒，睡眠完成后echo一段信息
```
由于第二条命令是放在后台执行的，所以该ssh一建立完成ssh会话就立即返回本地bash环境，但当5秒之后，将在本地突然显示"over"。

ssh执行远程命令默认允许从标准输入中读取数据然后传输到远程。可以使用"-n"选项，使得标准输入重定向为/dev/null。例如：
```
[root@xuexi ~]# echo haha | ssh 172.16.10.6 'cat' 
haha

[root@xuexi ~]# ssh 172.16.10.6 'cat' </etc/fstab
```
再看如下两条命令： 
```
[root@xuexi ~]# tar zc /tmp/* | ssh 172.16.10.6 'cd /tmp;tar xz' 
[root@xuexi ~]# ssh 172.16.10.6 'tar cz /tmp' | tar xz
```
第一条命令将/tmp下文件归档压缩，然后传送到远程主机上并被解包；第二条命令将远程主机上的/tmp目录归档压缩，并传输到本地解包。所以它们实现了拷贝的功能。

不妨再分析下面的命令，该命令改编自ssh-copy-id脚本中的主要命令。如果不知道ssh-copy-id命令是干什么的，后文有介绍。
```
[root@xuexi ~]# cat ~/.ssh/id_rsa.pub | ssh 172.16.10.6 "umask 077; test -d ~/.ssh || mkdir ~/.ssh ; cat >> ~/.ssh/authorized_keys"
```
该命令首先建立ssh连接，并在远程执行"umask 077"临时修改远程的umask值，使得远程创建的目录权限为700，然后判断远程主机上是否有`~/.ssh`目录，如果没有则创建，最后从标准输入中读取本地公钥文件`~/.ssh/id_rsa.pub`的内容并将其追加到`~/.ssh/authorized_keys`文件中。

如果将此命令改为如下命令，使用ssh的"-n"选项，并将追加重定向改为覆盖重定向符号。
```
[root@xuexi ~]# cat ~/.ssh/id_rsa.pub | ssh -n 172.16.10.6 "umask 077; test -d ~/.ssh || mkdir ~/.ssh ; cat > ~/.ssh/authorized_keys"
```
该命令的结果是清空远程主机172.16.10.6上的`~/.ssh/authorized_keys`文件，因为ssh的"-n"选项强行改变了ssh读取的标准输入为/dev/null。

## 7.scp命令及过程分析

scp是基于ssh的远程拷贝命令，也支持本地拷贝，甚至支持远程到远程的拷贝。

scp由于基于ssh，所以其端口也是使用ssh的端口。其实，scp拷贝的实质是使用ssh连接到远程，并使用该连接来传输数据。下文有scp执行过程的分析。

另外，scp还非常不占资源，不会提高多少系统负荷，在这一点上，rsync远不及它。虽然 rsync比scp会快一点，但rsync是增量拷贝，要判断每个文件是否修改过，在小文件众多的情况下，判断次数非常多，导致rsync效率较差，而scp基本不影响系统正常使用。
scp每次都是全量拷贝，在某些情况下，肯定是不及rsync的。
```
scp [-12BCpqrv] [-l limit] [-o ssh_option] [-P port] [[user@]host1:]src_file ... [[user@]host2:]dest_file

选项说明：
-1：使用ssh v1版本，这是默认使用协议版本
-2：使用ssh v2版本
-C：拷贝时先压缩，节省带宽
-l limit：限制拷贝速度，Kbit/s，1Byte=8bit，所以"-l 800"表示的速率是100K/S
-o ssh_option：指定ssh连接时的特殊选项，一般用不上。
-P port：指定目标主机上ssh端口，大写的字母P，默认是22端口
-p：拷贝时保持源文件的mtime,atime,owner,group,privileges
-r：递归拷贝，用于拷贝目录。注意，scp拷贝遇到链接文件时，会拷贝链接的源文件内容填充到目标文件中(scp的本质就是填充而非拷贝) 
-v：输出详细信息，可以用来调试或查看scp的详细过程，分析scp的机制
```

`src_file`是源位置，`dest_file`是目标位置，即将`src_file`复制到`dest_file`，其中`src_file`可以指定多个。由于源位置和目标位置都可以使用本地路径和远程路径，所以scp能实现本地拷贝到远程、本地拷贝到本地、远程拷贝到本地、远程拷贝到另一个远程。其中远程路径的指定格式为`user@hostname:/path`，可以省略user，也可以省略`:/path`，省略`:/path`时表示拷贝到目的用户的家目录下。

注意：scp拷贝是强制覆盖型拷贝，当有重名文件时，不会进行任何询问。

例如：

**(1).本地拷贝到本地**：`/etc/fstab-->/tmp/a.txt`
```
[root@xuexi ~]# scp /etc/fstab /tmp/a.txt
```
**(2).本地到远程**：`/etc/fstab-->172.16.10.6:/tmp/a.txt`
```
[root@xuexi ~]# scp /etc/fstab 172.16.10.6:/tmp
fstab             100%  805     0.8KB/s   00:00
```
**(3).远程到本地**：`172.16.10.6:/etc/fstab-->/tmp/a.txt`
```
[root@xuexi ~]# scp 172.16.10.6:/etc/fstab /tmp/a.txt
fstab             100%  501     0.5KB/s   00:00
```
**(4).远程路径1到远程路径2**：`172.16.10.6:/etc/fstab-->/172.16.10.3:/tmp/a.txt`
```
[root@xuexi ~]# scp 172.16.10.6:/etc/fstab 172.16.10.3:/tmp/a.txt
fstab             100%  501     0.5KB/s   00:00    
Connection to 172.16.10.6 closed.
```
远程复制到远程时，可能会因为sudoers的限制不给ssh分配终端，导致无法输入yes/密码或者直接拒绝等而失败。这时可以加上一层ssh：
```
ssh -tt 172.16.10.2 "scp 172.16.10.6:/etc/fstab 172.16.10.3:/tmp/a.txt"
```
其中172.16.10.2是执行ssh命令所在的主机A，也就是在A主机上使用scp将主机B(10.6)上的文件复制到主机C(10.3)上，这里的"-tt"表示强制分配伪终端给ssh。

另外，还可以使用scp的"-3"选项，改变scp远程到远程的默认传输模式(默认传输模式见下面的机制分析)，如果上面的命令失败的话，也可以使用该选项一试。
```
scp -3 172.16.10.6:/etc/fstab 172.16.10.3:/tmp/a.txt
```

### 7.1 scp拷贝机制

scp的拷贝实质是建立ssh连接，然后通过此连接来传输数据。如果是远程1拷贝到远程2，则是将scp命令转换后发送到远程1上执行，在远程1上建立和远程2的ssh连接，并通过此连接来传输数据。

在远程复制到远程的过程中，例如在本地(172.16.10.5)执行scp命令将A主机(172.16.10.6)上的/tmp/copy.txt复制到B主机(172.16.10.3)上的/tmp目录下，如果使用-v选项查看调试信息的话，会发现它的步骤类似是这样的。
```
# 以下是从结果中提取的过程
# 首先输出本地要执行的命令
Executing: /usr/bin/ssh -v -x -oClearAllForwardings yes -t -l root 172.16.10.6 scp -v /tmp/copy.txt root@172.16.10.3:/tmp

# 从本地连接到A主机
debug1: Connecting to 172.16.10.6 [172.16.10.6] port 22.
debug1: Connection established.

# 要求验证本地和A主机之间的连接
debug1: Next authentication method: password
root@172.16.10.6's password: 

# 将scp命令行修改后发送到A主机上
debug1: Sending command: scp -v /tmp/copy.txt root@172.16.10.3:/tmp

# 在A主机上执行scp命令
Executing: program /usr/bin/ssh host 172.16.10.3, user root, command scp -v -t /tmp

# 验证A主机和B主机之间的连接
debug1: Next authentication method: password
root@172.16.10.3's password: 

# 从A主机上拷贝源文件到最终的B主机上
debug1: Sending command: scp -v -t /tmp
Sending file modes: C0770 24 copy.txt
Sink: C0770 24 copy.txt
copy.txt                           100%   24     0.0KB/s

# 关闭本地主机和A主机的连接
Connection to 172.16.10.6 closed.
```
也就是说，远程主机A到远程主机B的复制，实际上是将scp命令行从本地传输到主机A上，由A自己去执行scp命令。也就是说，本地主机不会和主机B有任何交互行为，本地主机就像是一个代理执行者一样，只是帮助传送scp命令行以及帮助显示信息。

其实从本地主机和主机A上的`~/.ssh/know_hosts`文件中可以看出，本地主机只是添加了主机A的host key，并没有添加主机B的host key，而在主机A上则添加了主机B的host key。

![](/img/linux/733013-20170706231201847-1095045054.png)


## 8.基于公钥认证机制实现双机互信

在身份验证阶段，由于默认情况下基于公钥认证的机制顺序优先于基于密码认证的机制，所以基于公钥认证身份，就可以免输入密码，即实现双机互信(实际上只是单方向的信任)。

基于公钥认证机制的认证过程在前文已经详细说明过了，如还不清楚，请跳回上文。

### 8.1 实现步骤

以下是实现基于公钥认证的实现步骤：

**(1).在客户端使用ssh-keygen生成密钥对，存放路径按照配置文件的指示，默认是在~/.ssh/目录下**
```
[root@xuexi ~]# ssh-keygen -t rsa    # -t参数指定算法，可以是rsa或dsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):  # 询问私钥保存路径
Enter passphrase (empty for no passphrase):               # 询问是否加密私钥文件
Enter same passphrase again:            
Your identification has been saved in /root/.ssh/id_rsa.  
Your public key has been saved in /root/.ssh/id_rsa.pub.  
```
如果不想被询问，则可以使用下面一条命令完成："-f"指定私钥文件，"-P"指定passphrase，或者"-N"也一样。

```
[root@xuexi ~]# ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''   # 指定加密私钥文件的密码为空密码，即不加密

[root@xuexi ~]# ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''   # 同上
```
查看`~/.ssh/`目录下私钥的权限。私钥文件有严格的权限要求，当私钥文件的非所有者有可读权限时，将直接忽略该私钥文件导致公钥认证失败。
```
[root@xuexi ~]# ls -l ~/.ssh
total 12
-rw------- 1 root root 1671 Jun 29 00:18 id_rsa  # 私钥权限必须600，属主为自己
-rw-r--r-- 1 root root  406 Jun 29 00:18 id_rsa.pub
-rw-r--r-- 1 root root  393 Jun 29 05:56 known_hosts
```
**(2).将上面生成的公钥使用ssh-copy-id分发(即复制)到远程待信任主机上**

ssh-copy-id用法很简单，只需指定待信任主机及目标用户即可。如果生成的公钥文件，路径不是`~/.ssh/id_rsa.pub`，则使用"-i"选项指定要分发的公钥。
```
ssh-copy-id [-i [identity_file]] [user@]machine
```
例如，将公钥分发到172.16.10.6上的root用户家目录下：
```
[root@xuexi ~]# ssh-copy-id 172.16.10.6
```
ssh-copy-id唯一需要注意的是，如果ssh服务端的端口不是22，则需要给ssh-copy-id传递端口号，传递方式为"`-p port_num [user@]hostname`"，例如"`-p 22222 root@172.16.10.6`"。之所以要如此传递，见下面摘自ssh-copy-id中公钥分发的命令部分。
```
{ eval "$GET_ID" ; } | ssh $1 "umask 077; test -d ~/.ssh || mkdir ~/.ssh ; cat >> ~/.ssh/authorized_keys && (test -x /sbin/restorecon && /sbin/restorecon ~/.ssh ~/.ssh/authorized_keys >/dev/null 2>&1 || true)" || exit 1
```
其中"`{ eval "$GET_ID" ; }`"可理解为待分发的本地公钥内容，"`(test -x /sbin/restorecon && /sbin/restorecon ~/.ssh ~/.ssh/authorized_keys >/dev/null 2>&1 || true)`"和selinux有关，不用管，所以上述命令简化为：
```
cat ~/.ssh/id_rsa.pub | ssh $1 "umask 077; test -d ~/.ssh || mkdir ~/.ssh ; cat >> ~/.ssh/authorized_keys || exit 1
```

可见，ssh-copy-id的所有参数都是存储在位置变量$1中传递给ssh，所以应该将ssh的端口选项"`-p port_num`"和`user@hostname`放在一起传递。

通过分析上面的命令，也即知道了ssh-copy-id的作用：在目标主机的指定用户的家目录下，检测是否有`~/.ssh`目录，如果没有，则以700权限创建该目录，然后将本地的公钥追加到目标主机指定用户家目录下的`~/.ssh/authorized_keys`文件中。

### 8.2 一键脚本

就这样简单的两步就实现了基于公钥的身份认证。当然，不使用ssh-copy-id，也一样能实现上述过程。更便捷地，可以写一个简单的脚本，简化上述两个步骤为1个步骤。
```
#!/bin/bash

privkey="$HOME/.ssh/id_rsa"
publickey="$HOME/.ssh/id_rsa.pub"

# Usage help
if [ $# -ne 1 ];then
   echo "Usage:$0 [user@]hostname"
   exit 1
fi

# test private/publick key exist or not, and the privilege 600 or not
if [ -f "$privkey" -a -f "$publickey" ];then
   privkey_priv=`stat -c %a $privkey`
   if [ "$privkey_priv" -ne 600 ];then
       echo "The privilege of private key ~/.ssh/id_rsa is not 600, exit now."
       exit 1
   fi
else
   echo "private/public key is not exist, it will create it"
   ssh-keygen -t rsa -f $privkey -N ''
   echo "keys created over, it located on $HOME/.ssh/"
fi

ssh-copy-id "-o StrictHostKeyChecking=no $1"

if [ $? -eq 0 ];then
   echo -e "\e[31m publickey copy over \e[0m"
else
   echo "ssh can't to the remote host"
   exit 1
fi
```
该脚本将检查本地密钥对`~/.ssh/{id_rsa,id_rsa.pub}`文件是否存在，还检查私钥文件的权限是否为600。如果缺少某个文件，将自动新创建密钥对文件，最后分发公钥到目标主机。

### 8.3 公钥认证之——服务端分发私钥

对于基于公钥认证的身份验证机制，除了上面客户端分发公钥到服务端的方法，还可以通过分发服务端私钥到客户端来实现。

先理清下公钥认证的原理：客户端要连接服务端，并告诉服务端要使用那对密钥对，然后客户端访问自己的私钥，服务端检测对应的公钥来决定该公钥所对应的客户端是否允许连接。所以对于基于公钥认证的身份验证，无论是客户端分发公钥，还是服务端分发私钥，最终客户端保存的一定是私钥，服务端保存的一定是公钥。

那么服务端分发私钥实现公钥认证是如何实现的呢？步骤如下：假如客户端为172.16.10.5，服务端为172.16.10.6。

**(1).在服务端使用ssh-keygen生成密钥对**
```
[root@xuexi ~]# ssh-keygen -f ~/.ssh/id_rsa -P ''
```
**(2).将上面生成的公钥追加到自己的authorized_keys文件中**
```
[root@xuexi ~]# ssh-copy-id 172.16.10.6
```
**(3).将私钥拷贝到客户端，且路径和文件名为**`~/.ssh/id_rsa`
```
[root@xuexi ~]# scp -p ~/.ssh/id_rsa* 172.16.10.5:/root/.ssh/
```

注意，第三步中也拷贝了公钥，原因是客户端连接服务端时会比较自己的公钥和私钥是否配对，如果不配对将直接忽略公钥认证机制，所以会要求输入密码。可以将客户端的公钥删除掉，或者将服务端生成的公钥覆盖到客户端的公钥上，都能完成公钥认证。

虽说，服务端分发私钥的方式很少用，但通过上面的步骤，想必对ssh基于公钥认证的身份验证过程有了更深入的理解。

## 9.ssh-keyscan和sshpass轻松实现非交互式双机互信

`ssh-keyscan`命令可以生成`~/.ssh/known_hosts`文件中的公钥信息，所以通过它可以免去主机认证过程中的交互式询问。

`sshpass`命令可以自动填充ssh身份验证过程中的交互式密码询问，所以通过它结合`ssh-copy-id`命令可以免去输入密码的交互式询问。

例如，`root@192.168.100.22`主机需要和`root@192.168.100.23`到`root@192.168.100.33`这11台主机之间配置双机互信。参考如下过程：

```bash
# 1.在root@192.168.100.22上生成密钥
$ ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''

# 2. 向root@192.168.100.22的~/.ssh/known_hosts文件追加各主机的主机认证信息
#    并将公钥分发给其它节点
for host in 192.168.100.{23..33};do
    ssh-keyscan $host >>~/.ssh/known_hosts 2>/dev/null
    # -p指定的是密码
    sshpass -p'123456' ssh-copy-id root@$host &>/dev/null
done
```

## 10.expect实现ssh/scp完全非交互(批量)

expect工具可以在程序发出交互式询问时按条件传递所需的字符串，例如询问yes/no自动传递y或yes，询问密码时自动传递指定的密码等，这样就能让脚本完全实现非交互。

显然，ssh等客户端命令基于密码认证时总是会询问密码，即使是基于公钥认证，在建立公钥认证时也要询问一次密码。另外，在ssh主机验证时还会询问是否保存host key。这一切都可以通过expect自动回答。

关于expect工具，它使用的是tcl语言，虽说应用在expect上的tcl语言并非太复杂，但这并非本文内容，如有兴趣，可网上搜索或直接阅读man文档。

首先安装expect工具。
```
[root@xuexi ~]# yum -y install expect
```
### 10.1 scp自动应答脚本

以下是scp自动问答的脚本。
```
[root@xuexi ~]# cat autoscp.exp
#!/usr/bin/expect
set timeout 10
set user_hostname [lindex $argv 0]
set src_file [lindex $argv 1]
set dest_file [lindex $argv 2]
set password [lindex $argv 3]
spawn scp $src_file $user_hostname:$dest_file
    expect {
        "(yes/no)?"
        {
            send "yes\n"
            expect "*assword:" { send "$password\n"}
        }
        "*assword:"
        {
            send "$password\n"
        }
    }
expect "100%"
expect eof 
```
用法：`autoscp.exp [user@]hostname src_file dest_file [password]`

该自动回答脚本可以自动完成主机验证和密码认证，即使已经是实现公钥认证的机器也没问题，因为公钥认证机制默认优先于密码认证，且此脚本的password项是可选的，当然，在没有实现公钥认证的情况下，password是必须项，否则expect实现非交互的目的就失去意义了。

以下是几个示例：
```
[root@xuexi ~]# ./autoscp.exp 172.16.10.6 /etc/fstab /tmp 123456
spawn scp /etc/fstab 172.16.10.6:/tmp
The authenticity of host '172.16.10.6 (172.16.10.6)' can't be established.
RSA key fingerprint is f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf.
Are you sure you want to continue connecting (yes/no)? yes    # 主机验证时询问是否保存host key，自动回答yes
Warning: Permanently added '172.16.10.6' (RSA) to the list of known hosts.
root@172.16.10.6's password:          # 密码认证过程，自动回答指定的密码"123456"
fstab             100%  805     0.8KB/s   00:00
```
也可以指定完成的用户名和主机名。
```
[root@xuexi ~]# ./autoscp.exp root@172.16.10.6 /etc/fstab /tmp 123456
spawn scp /etc/fstab root@172.16.10.6:/tmp
root@172.16.10.6's password:         
fstab             100%  805     0.8KB/s   00:00
```

### 10.2 ssh-copy-id自动应答脚本

以下是在建立公钥认证机制时，ssh-copy-id拷贝公钥到服务端的自动应答脚本。
```
[root@xuexi ~]# cat /tmp/autocopy.exp
#!/usr/bin/expect
set timeout 10
set user_hostname [lindex $argv 0]
set password [lindex $argv 1]
spawn ssh-copy-id $user_hostname
    expect {
        "(yes/no)?"
        {
            send "yes\n"
            expect "*assword:" { send "$password\n"}
        }
        "*assword:"
        {
            send "$password\n"
        }
    }
expect eof 
```
用法：`autocopy.exp [user@]hostname password`

以下是一个示例：
```
[root@xuexi ~]# /tmp/autocopy.exp root@172.16.10.6 123456
spawn ssh-copy-id root@172.16.10.6
The authenticity of host '172.16.10.6 (172.16.10.6)' can't be established.
RSA key fingerprint is f3:f8:e2:33:b4:b1:92:0d:5b:95:3b:97:d9:3a:f0:cf.
Are you sure you want to continue connecting (yes/no)? yes      # 主机认证时，自动应答yes
Warning: Permanently added '172.16.10.6' (RSA) to the list of known hosts.
root@172.16.10.6's password:                                    # 密码认证时自动输入密码"123456"
Now try logging into the machine, with "ssh 'root@172.16.10.6'", and check in:

  .ssh/authorized_keys

to make sure we haven't added extra keys that you weren't expecting.
```
如果要实现批量非交互，则可以写一个shell脚本调用该expect脚本。例如：
```
[root@xuexi ~]# cat /tmp/sci.sh
#!/bin/bash

passwd=123456               # 指定要传递的密码为123456
user_host=`awk '{print $3}' ~/.ssh/id_rsa.pub`   # 此变量用于判断远程主机中是否已添加本机信息成功

for i in $@   
do
        /tmp/autocopy.exp $i $passwd >&/dev/null
        ssh $i "grep "$user_host" ~/.ssh/authorized_keys" >&/dev/null  # 判断是否添加本机信息成功
        if [ $? -eq 0 ];then
                echo "$i is ok"
        else
                echo "$i is not ok"
        fi
done
```
用法：`/tmp/sci.sh [user@]hostname`

其中hostname部分可以使用花括号展开方式枚举。但有个bug，最好ssh-copy-id的目标不要是脚本所在的本机，可能会强制输入本机密码，但批量脚本autocopy.exp则没有此bug。
例如：
```
[root@xuexi tmp]# /tmp/sci.sh 172.16.10.3 172.16.10.6
172.16.10.3 is ok
172.16.10.6 is ok

[root@xuexi tmp]# /tmp/sci.sh 172.16.10.{3,6}
172.16.10.3 is ok
172.16.10.6 is ok

[root@xuexi tmp]# /tmp/sci.sh root@172.16.10.3 172.16.10.6
root@172.16.10.3 is ok
172.16.10.6 is ok
```

## 11.ssh连接速度慢的几个原因和解决办法

ssh连接包括两个阶段：主机验证阶段和身份验证阶段。这两个阶段都可能导致连接速度慢。

具体是哪个阶段的速度慢，完全可以通过肉眼看出来：  
- (1).卡着很久才提示保存host key肯定是主机验证过程慢。  
- (2).主机验证完成后卡着很久才提示输入密码，肯定是身份验证过程慢。  
其中主机验证过程慢的原因，可能是网络连接慢、DNS解析慢等原因。网络连接慢，ssh对此毫无办法，而DNS解析慢，ssh是可以解决的，解决方法是将ssh服务端的配置文件中UseDNS设置为no(默认为yes)。

而身份验证慢的原因，则考虑ssh的身份验证顺序：gssapi,host-based,publickey,keyboard-interactive,password。其中gssapi认证顺序是比较慢的，所以解决方法一是在ssh客户端配置文件中将GSSAPI认证机制给关掉，解决方法二是在ssh客户端配置文件中使用PreferredAuthentications指令修改身份验证顺序。

- 方法一修改：GSSAPIAuthentication yes  
- 方法二修改：PreferredAuthentications publickey,password,gssapi,host-based,keyboard-interactive  

如果感受不到哪个阶段导致速度变慢，可以使用ssh或scp等客户端工具的"-vvv"选项进行调试，看看是卡在哪个地方，不过，想看懂"-vvv"的过程，还是比较考验耐心的。


