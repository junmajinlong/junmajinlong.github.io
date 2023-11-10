---
title: NFS网络文件系统基本应用
p: linux/nfs_basic.md
date: 2021-04-16 18:20:38
tags: Linux
categories: Linux
---

--------

**[回到Linux基础(rsync)系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# NFS网络文件系统基本应用

## 概述

类似ext家族、xfs格式的本地文件系统，它们都是通过单个文件名称空间(name space)来包含很多文件，并提供基本的文件管理和空间分配功能。而文件是存放在文件系统中(上述名称空间内)的单个命名对象，每个文件都包含了文件实际数据和属性数据。但是，这些类型的文件系统和其内文件都是存放在本地主机上的。

实际上，还有网络文件系统。顾名思义，就是跨网络的文件系统，将远程主机上的文件系统(或目录)存放在本地主机上，就像它本身就是本地文件系统一样。在Windows环境下有cifs协议实现的网络文件系统，在Unix环境下，最出名是由NFS协议实现的NFS文件系统。

NFS即network file system的缩写，nfs是属于用起来非常简单，研究起来非常难的东西。相信，使用过它或学过它的人都不会认为它的使用有任何难点，只需将远程主机上要共享给客户端的目录导出(export)，然后在客户端上挂载即可像本地文件系统一样。到目前为止，nfs已经有5个版本，NFSv1是未公布出来的版本，v2和v3版本目前来说基本已经淘汰，v4版本是目前使用最多的版本，nfsv4.1是目前最新的版本。

## RPC不可不知的原理

要介绍NFS，必然要先介绍RPC。RPC是remote procedure call的简写，人们都将其译为"远程过程调用"，它是一种框架，这种框架在大型公司应用非常多。而NFS正是其中一种，此外NIS、hadoop也是使用rpc框架实现的。

### RPC原理

所谓的remote procedure call，就是在本地调用远程主机上的procedure。以本地执行`cat -n ~/abc.txt`命令为例，在本地执行cat命令时，会发起某些系统调用(如open()、read()、close()等)，并将cat的选项和参数传递给这些函数，于是最终实现了文件的查看功能。在RPC层面上理解，上面发起的系统调用就是procedure，每个procedure对应一个或多个功能。而rpc的全名remote procedure call所表示的就是实现远程procedure调用，让远程主机去调用对应的procedure。

上面的cat命令只是本地执行的命令，如何实现远程cat，甚至其他远程命令？通常有两种可能实现的方式：

(1).使用ssh类的工具，将要执行的命令传递到远程主机上并执行。但ssh无法直接调用远程主机上cat所发起的那些系统调用(如open()、read()、close()等)。

(2).使用网络socket的方式，告诉远程服务进程要调用的函数。但这样的主机间进程通信方式一般都是daemon类服务，daemon类的客户端(即服务的消费方)每调用一个服务的功能，都需要编写一堆实现网络通信相关的代码。不仅容易出错，还比较复杂。

而rpc是最好的解决方式。rpc是一种框架，在此框架中已经集成了网络通信代码和封包、解包方式(编码、解码)。以下是rpc整个过程，以cat NFS文件系统中的a.sh文件为例。

![](/img/linux/1699410786698.png)

nfs客户端执行`cat a.sh`，由于a.sh是NFS文件系统内的文件，所以cat会发起一些procedure调用(如open/read/close)，这些procedure对应的ID号码和对应的参数会发送给rpc client(可能是单个procedure ID，也可能是多个procedure组合在一起一次性发送给rpc client，在NFSv4上是后者)，rpc client会将这些数据进行编码封装(封装和解封装功能由stub代码实现)，封装后的消息称为"call message"，然后将call message通过网络发送给rpc server，rpc server会对封装的数据进行解封提取，于是就得到了要调用的procedures ID和对应的参数，然后将它们交给NFS服务进程，最终调用procedure ID对应的procedure来执行，并返回结果。NFS服务发起procedure调用后，会得到数据(可能是数据本身，可能是状态消息等)，于是将返回结果交给rpc server，rpc server会将这些数据封装，这部分数据称为"reply message"，然后将reply message通过网络发送给rpc client，rpc client解封提取，于是得到最终的返回结果。

从上面的过程可以知道，**rpc的作用是数据封装，rpc client封装待调用的procedure ID及其参数(其实还有一个program ID，关于program，见下文)，rpc server封装返回的数据。**

举个更简单的例子，使用google搜索时，实现搜索功能的program ID以及涉及到的procedure ID和要搜索的内容就是rpc client封装的对象，也是rpc server要解封的对象，搜索的结果则是rpc server封装的对象，也是rpc client要解封的对象。解封后的最终结果即为google搜索的结果。

### RPC工具介绍

在CentOS 6/7上，rpc server由rpcbind程序实现，该程序由rpcbind包提供。

```
[root@xuexi ~]# yum -y install rpcbind

[root@xuexi ~]# rpm -ql rpcbind | grep bin/
/usr/sbin/rpcbind
/usr/sbin/rpcinfo
```

其中rpcbind是rpc主程序，在rpc服务端该程序必须处于已运行状态，其**默认监听在111端口**。rpcinfo是rpc相关信息查询工具。

对于rpc而言，其所直接管理的是programs，programs由一个或多个procedure组成。这些program称为RPC program或RPC service。

如下图，其中NFS、NIS、hadoop等称为网络服务，它们由多个进程或程序(program)组成。例如NFS包括rpc.nfsd、rpc.mountd、rpc.statd和rpc.idmapd等programs，其中每个program都包含了一个或多个procedure，例如rpc.nfsd这个程序包含了如OPEN、CLOSE、READ、COMPOUND、GETATTR等procedure，rpc.mountd也主要有MNT和UMNT两个procedure。

![](/img/linux/1699410807155.png)

对于RPC而言，它是不知道NFS/NIS/hadoop这一层的，它直接管理programs。每个program启动时都需要找111端口的rpc服务登记注册，然后RPC服务会为该program映射一个program number以及分配一个端口号。其中每个program都有一个唯一与之对应的program number，它们的映射关系定义在/etc/rpc文件中。以后rpc server将使用program number来判断要调用的是哪个program中的哪个procedure(因为这些都是rpc client封装在"call message"中的)，并将解包后的数据传递给该program和procedure。

例如只启动rpcbind时。

```
[root@xuexi ~]# systemctl start rpcbind.service

[root@xuexi ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
```

其中第一列就是program number，第二列vers表示对应program的版本号，最后一列为RPC管理的RPC service名，其实就是各program对应的称呼。

当客户端获取到rpc所管理的service的端口后，就可以与该端口进行通信了。**但注意，即使客户端已经获取了端口号，客户端仍会借助rpc做为中间人进行通信。也就是说，无论何时，客户端和rpc所管理的服务的通信都必须通过rpc来完成。之所以如此，是因为只有rpc才能封装和解封装数据。**

既然客户端不能直接拿着端口号和rpc service通信，那还提供端口号干嘛？这个端口号是为rpc server提供的，rpc server解包数据后，会将数据通过此端口交给对应的rpc service。

## 启动NFS

NFS本身是很复杂的，它由很多进程组成。这些进程的启动程序由nfs-utils包提供。由于nfs是使用RPC框架实现的，所以需要先安装好rpcbind。不过安装nfs-utils时会自动安装rpcbind。

```
[root@xuexi ~]# yum -y install nfs-utils

[root@xuexi ~]# rpm -ql nfs-utils | grep /usr/sbin
/usr/sbin/blkmapd
/usr/sbin/exportfs
/usr/sbin/mountstats
/usr/sbin/nfsdcltrack
/usr/sbin/nfsidmap
/usr/sbin/nfsiostat
/usr/sbin/nfsstat
/usr/sbin/rpc.gssd
/usr/sbin/rpc.idmapd
/usr/sbin/rpc.mountd
/usr/sbin/rpc.nfsd
/usr/sbin/rpc.svcgssd
/usr/sbin/rpcdebug
/usr/sbin/showmount
/usr/sbin/sm-notify
/usr/sbin/start-statd
```

其中以"rpc."开头的程序都是rpc service，分别实现不同的功能，启动它们时每个都需要向rpcbind进行登记注册。

```
[root@xuexi ~]# systemctl start nfs.service

[root@xuexi ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  56229  status
    100024    1   tcp  57226  status
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  48609  nlockmgr
    100021    3   udp  48609  nlockmgr
    100021    4   udp  48609  nlockmgr
    100021    1   tcp  50915  nlockmgr
    100021    3   tcp  50915  nlockmgr
    100021    4   tcp  50915  nlockmgr
```

可以看到，每个program都启动了不同版本的功能。其中nfs program为rpc.nfsd对应的program，为nfs服务的主进程，端口号为2049。mountd对应的program为rpc.mountd，它为客户端的mount和umount命令提供服务，即挂载和卸载NFS文件系统时会联系mountd服务，由mountd维护相关挂载信息。nlockmgr对应的program为rpc.statd，用于维护文件锁和文件委托相关功能，在NFSv4以前，称之为NSM(network status manager)。nfs\_acl和status，很显然，它们是访问控制列表和状态信息维护的program。

再看看启动的相关进程信息。

```
[root@xuexi ~]# ps aux | grep -E "[n]fs|[r]pc"
root      748  0.0  0.0      0     0 ?   S< Jul26  0:00 [rpciod]
rpc      6127  0.0  0.0  64908  1448 ?   Ss Jul26  0:00 /sbin/rpcbind -w
rpcuser  6128  0.0  0.0  46608  1836 ?   Ss Jul26  0:00 /usr/sbin/rpc.statd --no-notify
root     6242  0.0  0.0      0     0 ?   S< Jul26  0:00 [nfsiod]
root     6248  0.0  0.0      0     0 ?   S  Jul26  0:00 [nfsv4.0-svc]
root    17128  0.0  0.0  44860   976 ?   Ss 02:49  0:00 /usr/sbin/rpc.mountd
root    17129  0.0  0.0  21372   420 ?   Ss 02:49  0:00 /usr/sbin/rpc.idmapd
root    17134  0.0  0.0      0     0 ?   S< 02:49  0:00 [nfsd4]
root    17135  0.0  0.0      0     0 ?   S< 02:49  0:00 [nfsd4_callbacks]
root    17141  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17142  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17143  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17144  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17145  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17146  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17147  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
root    17148  0.0  0.0      0     0 ?   S  02:49  0:00 [nfsd]
```

其中有一项/usr/sbin/rpc.idmapd进程，该进程是提供服务端的`uid/gid <==> username/groupname`的映射翻译服务。客户端的`uid/gid <==> username/groupname`的映射翻译服务则由"nfsidmap"工具实现，详细说明见下文。

## 配置导出目录和挂载使用

### 配置nfs导出目录

在将服务端的目录共享(share)或者说导出(export)给客户端之前，需要先配置好要导出的目录。比如何人可访问该目录，该目录是否可写，以何人身份访问导出目录等。

配置导出目录的配置文件为`/etc/exports`或`/etc/exports.d/*.exports`文件，在nfs服务启动时，会自动加载这些配置文件中的所有导出项。以下是导出示例：

```
/www  172.16.0.0/16(rw,async,no_root_squash)
```

其中/www是导出目录，即共享给客户端的目录；172.16.0.0/16是访问控制列表ACL，只有该网段的客户端主机才能访问该导出目录，即挂载该导出目录；紧跟在主机列表后的括号及括号中的内容定义的是该导出目录对该主机的导出选项，例如(rw,async,no_root_squash)表示客户端挂载/www后，该目录可读写、异步、可保留root用户的权限，具体的导出选项稍后列出。

以下是可接收的几种导出方式：

```
# 导出给所有主机，此时称为导出给world
/www1    (rw,async,no_root_squash)

# 仅导出给单台主机172.16.1.1
/www2    172.16.1.1(rw,async)

# 导出给网段172.16.0.0/16，还导出给单台主机192.168.10.3，
# 且它们的导出选项不同
/www3    172.16.0.0/16(rw,async) 192.168.10.3(rw,no_root_squash)

# 导出给单台主机www.a.com主机，但要求能解析该主机名
/www4    www.a.com(rw,async)

# 导出给b.com下的所有主机，要求能解析对应主机名
/www     *.b.com(rw,async)
```

以下是常用的一些导出选项说明，更多的导出选项见`man exports`：常见的默认项是：`ro,sync,root_squash,no_all_squash,wdelay`。

| 导出选项(后者为默认)           | 选项说明                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| rw,ro                          | 导出目录可读写还是只读(read-only)。                          |
| async,sync                     | 同步共享还是异步共享。异步时，客户端提交要写入的数据到服务端，服务端接收数据后直接响应客户端，但此时数据并不一定已经写入磁盘中，而同步则是必须等待服务端已将数据写入磁盘后才响应客户端。也就是说，给定异步导出选项时，虽然能提升一些性能，但在服务端突然故障或重启时有丢失一部分数据的风险。当然，对于只读(ro)的导出目录，设置sync或async是没有任何差别的。 |
| anonuid,<br>anongid            | 此为匿名用户(anonymous)的uid和gid值，默认都为65534，在/etc/passwd和/etc/shadow中它们对应的用户名为nfsnobody。该选项指定的值用于身份映射被压缩时。 |
| no_root_squash,<br>root_squash | 是否将发起请求(即客户端进行访问时)的uid/gid=0的root用户映射为anonymous用户。即是否压缩root用户的权限。 |
| all_squash,<br/>no_all_squash  | 是否将发起请求(即客户端进行访问时)的所有用户都映射为anonymous用户，即是否压缩所有用户的权限。 |

对于root用户，将取`(no_)root_squash`和`(no_)all_squash`的交集。例如，`no_root_squash`和`all_squash`同时设置时，root仍被压缩，`root_squash`和`no_all_squash`同时设置时，root也被压缩。

有些导出选项需要配合其他设置。例如，导出选项设置为rw，但如果目录本身没有w权限，或者mount时指定了ro挂载选项，则同样不允许写操作。

在配置文件写好要导出的目录后，直接重启nfs服务即可，它会读取这些配置文件。随后就可以在客户端执行mount命令进行挂载。

例如，exports文件内容如下：

```
/vol/vol0       *(rw,no_root_squash)
/vol/vol2       *(rw,no_root_squash)
/backup/archive *(rw,no_root_squash)
```

### 挂载nfs文件系统

然后去客户端上挂载它们。

```
[root@xuexi ~]# mount -t nfs 172.16.10.5:/vol/vol0 /mp1
[root@xuexi ~]# mount 172.16.10.5:/vol/vol2 /mp2
[root@xuexi ~]# mount 172.16.10.5:/backup/archive /mp3
```

挂载时`-t nfs`可以省略，因为对于mount而言，只有挂载nfs文件系统才会写成`host:/path`格式。当然，除了mount命令，nfs-utils包还提供了独立的mount.nfs命令，它其实和`mount -t nfs`命令是一样的。

mount挂载时可以指定挂载选项，其中包括mount通用挂载选项，如rw/ro，atime/noatime，async/sync，auto/noauto等，也包括针对nfs文件系统的挂载选项。以下列出几个常见的，更多的内容查看`man nfs`和`man mount`。

| 选项                    | 参数意义                                                     | 默认值 |
| ----------------------- | ------------------------------------------------------------ | ------ |
| suid,nosuid             | 如果挂载的文件系统上有设置了suid的二进制程序，使用nosuid可以取消它的suid | suid   |
| rw,ro                   | 尽管服务端提供了rw权限，但是挂载时设定ro，则还是ro权限**权限取交集** | rw     |
| exec,noexec             | 是否可执行挂载的文件系统里的二进制文件                       | exec   |
| user,nouser             | 是否运行普通用户进行档案的挂载和卸载                         | nouser |
| auto,noauto             | auto等价于mount -a，意思是将/etc/fstab里设定的全部重挂一遍   | auto   |
| sync,nosync             | 同步挂载还是异步挂载                                         | async  |
| atime,noatime           | 是否修改atime，对于nfs而言，该选项是无效的，理由见下文       |        |
| diratime,<br>nodiratime | 是否修改目录atime，对于nfs而言，该挂载选项是无效的，理由见下文 |        |
| remount                 | 重新挂载                                                     |        |

以下是针对nfs文件系统的挂载选项。其中没有给出关于缓存选项(ac/noac、cto/nocto、lookupcache)的说明，它们可以直接采用默认值，如果想要了解缓存相关内容，可以查看man nfs。

| 选项           | 功能                                                     | 默认值 |
| ------------------ | ------------------------------------------------------------ | ---------- |
| fg,bg          | **挂载失败后**mount命令的行为。默认为fg，表示挂载失败时将直接报错退出，如果是bg，挂载失败后会创建一个子进程不断在后台挂载，而父进程mount自身则立即退出并返回0状态码。 | fg     |
| timeo          | NFS客户端等待下一次重发NFS请求的时间间隔，单位为十分之一秒。基于TCP的NFS的默认timeo的值为600(60秒)。 |            |
| hard,soft      | 决定NFS客户端当NFS请求超时时的恢复行为方式。如果是hard，将无限重新发送NFS请求。例如在客户端使用df -h查看文件系统时就会不断等待。设置soft，当retrans次数耗尽时，NFS客户端将认为NFS请求失败，从而使得NFS客户端返回一个错误给调用它的程序。 | hard   |
| retrans        | NFS客户端最多发送的请求次数，次数耗尽后将报错表示连接失败。如果hard挂载选项生效，则会进一步尝试恢复连接。 | 3      |
| rsize,wsize | 一次读出(rsize)和写入(wsize)的区块大小。如果网络带宽大，这两个值设置大一点能提升传输能力。最好设置到带宽的临界值。单位为字节，大小只能为1024的倍数，且最大只能设置为1M。 |            |

注意三点：

(1).soft在特定的环境下超时后会导致静态数据中断。因此，**仅当客户端响应速度比数据完整性更重要时才使用soft选项。**使用基于TCP的NFS(除非显示指定使用UDP，否则现在总是默认使用TCP)或增加retrans重试次数可以降低使用soft选项带来的风险。

如果真的出现NFS服务端下线，导致NFS客户端无限等待的情况，可以强制将NFS文件系统卸载，卸载方法：

```
umount -f -l MOUNT_POINT
```

其中"-f"是强制卸载，"-l"是lazy umount，表示将该文件系统从当前目录树中剥离，让所有对该文件系统内的文件引用都强制失效。对于丢失了NFS服务端的文件系统，卸载时"-l"选项是必须的。

(2).由于nfs的客户端挂载后会缓存文件的属性信息，其中包括各种文件时间戳，所以**mount指定时间相关的挂载选项是没有意义的**，它们不会有任何效果，包括**atime/noatime，diratime/nodiratime**，relatime/norelatime以及strictatime/nostrictatime等。具体可见man nfs中"DATA AND METADATA COHERENCE"段的"File timestamp maintainence"说明，或者见本文末尾的翻译。

(3).如果是要开机挂载NFS文件系统，方式自然是写入到/etc/fstab文件或将mount命令放入rc.local文件中。如果是将/etc/fstab中，那么在系统环境初始化(exec /etc/rc.d/rc.sysinit)的时候会加载fstab中的内容，如果挂载fstab中的文件系统出错，则会导致系统环境初始化失败，结果是系统开机失败。所以，要开机挂载nfs文件系统，则需要在/etc/fstab中加入一个挂载选项"\_rnetdev"或"\_netdev"(centos 7中已经没有"\_rnetdev")，防止无法联系nfs服务端时导致开机启动失败。例如：

> 172.16.10.5:/www    /mnt    nfs    defaults,_rnetdev    0    0

当导出目录后，将在/var/lib/nfs/etab文件中写入一条对应的导出记录，这是nfs维护的导出表，该表的内容会交给rpc.mountd进程，并在必要的时候(mountd接受到客户端的mount请求时)，将此导出表中的内容加载到内核中，内核也单独维护一张导出表。

### nfs伪文件系统

服务端导出/vol/vol0、/vol/vol2和/backup/archive后，其中vol0和vol1是连在一个目录下的，但它们和archive目录没有连在一起，nfs采用伪文件系统的方式来桥接这些不连接的导出目录。桥接的方式是创建那些未导出的连接目录，如伪vol目录，伪backup目录以及顶级的伪根，如下图所示。

![](/img/linux/1699410836852.png)

当客户端挂载后，每次访问导出目录时，其实都是通过找到伪文件系统(文件系统都有id，在nfs上伪文件系统的id称为fsid)并定位到导出目录的。

## showmount命令

使用showmount命令可以查看某一台主机的导出目录情况。因为涉及到rpc请求，所以如果rpc出问题，showmount一样会傻傻地等待。

主要有3个选项。

```
showmount [ -ade]  host
-a：以host:dir格式列出客户端名称/IP以及所挂载的目录。
    但注意该选项是读取NFS服务端/var/lib/nfs/rmtab文件，
    而该文件很多时候并不准确，所以showmount -a的输出信
    息很可能并非准确无误的
-e：显示NFS服务端所有导出列表。
-d：仅列出已被客户端挂载的导出目录。
```

另外showmount的结果是排序过的，所以和实际的导出目录顺序可能并不一致。

例如：

```
[root@xuexi ~]# showmount -e 172.16.10.5
Export list for 172.16.10.5:
/backup/archive *
/vol/vol2       *
/vol/vol0       *
/www            172.16.10.4
[root@xuexi ~]# showmount -d 172.16.10.5
Directories on 172.16.10.5:
/backup/archive
/vol/vol0
/vol/vol2
```

## nfs身份映射

NFS的目的是导出目录给各客户端，因此导出目录中的文件在服务端和客户端上必然有两套属性、权限集。

例如，服务端导出目录中某a文件的所有者和所属组都为A，但在客户端上不存在A，那么在客户端上如何显示a文件的所有者等属性。再例如，在客户端上，以用户B在导出目录中创建了一个文件b，如果服务端上没有用户B，在服务端上该如何决定文件b的所有者等属性。

所以，NFS采用`uid/gid <==> username/groupname`映射的方式解决客户端和服务端两套属性问题。由于服务端只能控制它自己一端的身份映射，所以客户端也同样需要身份映射组件。也就是说，服务端和客户端两端都需要对导出的所有文件的所有者和所属组进行映射。

但要注意，服务端的身份映射组件为rpc.idmapd，它以守护进程方式工作。而客户端使用nfsidmap工具进行身份映射。

**服务端映射时以uid/gid为基准，**意味着客户端以身份B(假设对应uid=Xb，gid=Yb)创建的文件或修改了文件的所有者属性时，在服务端将从/etc/passwd(此处不考虑其他用户验证方式)文件中搜索uid=Xb，gid=Yb的用户，如果能搜索到，则设置该文件的所有者和所属组为此uid/gid对应的username/groupname，如果搜索不到，则文件所有者和所属组直接显示为uid/gid的值。

**客户端映射时以username/groupname为基准，**意味着服务端上文件所有者为A时，则在客户端上搜索A用户名，如果搜索到，则文件所有者显示为A，否则都将显示为nobody。注意，客户端不涉及任何uid/gid转换翻译过程，即使客户端上A用户的uid和服务端上A用户的uid不同，也仍显示为用户A。也就是说，客户端上文件所有者只有两种结果，要么和服务端用户同名，要么显示为nobody。

因此考虑一种特殊情况，客户端上以用户B(其uid=B1)创建文件，假如服务端上没有uid=B1的用户，那么创建文件时提交给服务端后，在服务端上该文件所有者将显示为B1(注意它是一个数值)。再返回到客户端上看，客户端映射时只简单映射username，不涉及uid的转换，因此它认为该文件的所有者为B1(不是uid，而是username)，但客户端上必然没有用户名为B1的用户(尽管有uid=B1对应的用户B)，因此在客户端，此文件所有者将诡异地将显示为nobody，其诡异之处在于，客户端上以身份B创建的文件，结果在客户端上却显示为nobody。

综上考虑，强烈建议客户端和服务端的用户身份要统一，且尽量让各uid、gid能对应上。

## 使用exportfs命令导出目录

除了启动nfs服务加载配置文件/etc/exports来导出目录，使用exportfs命令也可以直接导出目录，它无需加载配置文件/etc/exports，当然exportfs也可以加载/etc/exports文件来导出目录。实际上，nfs服务启动脚本中就是使用exportfs命令来导出/etc/exports中内容的。

例如，CentOS 6上/etc/init.d/nfs文件中，导出和卸载导出目录的命令为：

```
[root@xuexi ~]# grep exportfs /etc/init.d/nfs  
        [ -x /usr/sbin/exportfs ] || exit 5
        action $"Starting NFS services: " /usr/sbin/exportfs -r
        cnt=`/usr/sbin/exportfs -v | /usr/bin/wc -l`
                action $"Shutting down NFS services: " /usr/sbin/exportfs -au
        /usr/sbin/exportfs -r
```

在CentOS 7上则如下：

```
[root@xuexi ~]# grep exportfs /usr/lib/systemd/system/nfs.service      
ExecStartPre=-/usr/sbin/exportfs -r
ExecStopPost=/usr/sbin/exportfs -au
ExecStopPost=/usr/sbin/exportfs -f
ExecReload=-/usr/sbin/exportfs -r
```

当然，无论如何，nfsd等守护进程是必须已经运行好的。

以下是CentOS 7上exportfs命令的用法。注意， CentOS 7比CentOS 6多一些选项。

```
-a     导出或卸载所有目录。
-o options,...
       指定一系列导出选项(如rw,async,root_squash)，这些导出选项在exports(5)的man文档中有记录。
-i     忽略/etc/exports和/etc/exports.d目录下文件。此时只有命令行中给定选项和默认选项会生效。
-r     重新导出所有目录，并同步修改/var/lib/nfs/etab文件中关于/etc/exports和/etc/exports.d/
       *.exports的信息(即还会重新导出/etc/exports和/etc/exports.d/*等导出配置文件中的项)。该
       选项会移除/var/lib/nfs/etab中已经被删除和无效的导出项。
-u     卸载(即不再导出)一个或多个导出目录。
-f     如果/prof/fs/nfsd或/proc/fs/nfs已被挂载，即工作在新模式下，该选项将清空内核中导出表中
       的所有导出项。客户端下一次请求挂载导出项时会通过rpc.mountd将其添加到内核的导出表中。
-v     输出详细信息。
-s     显示适用于/etc/exports的当前导出目录列表。
```

例如：

**(1).导出/www目录给客户端172.16.10.6。**

```
exportfs 172.16.10.6:/www
```

**(2).导出/www目录给所有人，并指定导出选项。**

```
exportfs :/www -o rw,no_root_squash
```

**(3).导出exports文件中的内容。**

```
exportfs -a
```

**(4).重新导出所有已导出的目录。包括exports文件中和exportfs单独导出的目录。**

```
exportfs -ar
```

**(5).卸载所有已导出的目录，包括exports文件中的内容和exportfs单独导出的内容。即其本质为清空内核维护的导出表。**

```
exportfs -au
```

**(6).只卸载某一个导出目录。**

```
exportfs -u 172.16.10.6:/www
```

## RPC的调试工具rpcdebug

在很多时候NFS客户端或者服务端出现异常，例如连接不上、锁状态丢失、连接非常慢等等问题，都可以对NFS进行调试来发现问题出在哪个环节。NFS有不少进程都可以直接支持调试选项，但最直接的调试方式是调试rpc，因为NFS的每个请求和响应都会经过RPC去封装。但显然，调试RPC比直接调试NFS时更难分析出问题所在。以下只介绍如何调试RPC。

rpc单独提供一个调试工具rpcdebug。

```
[root@xuexi ~]# rpcdebug -vh
usage: rpcdebug [-v] [-h] [-m module] [-s flags...|-c flags...]
       set or cancel debug flags.
 
Module     Valid flags
rpc        xprt call debug nfs auth bind sched trans svcsock svcdsp misc cache all
nfs        vfs dircache lookupcache pagecache proc xdr file root callback client mount fscache pnfs pnfs_ld state all
nfsd       sock fh export svc proc fileop auth repcache xdr lockd all
nlm        svc client clntlock svclock monitor clntsubs svcsubs hostcache xdr all
```

其中：

```
-v：显示更详细信息
-h：显示帮助信息
-m：指定调试模块，有rpc/nfs/nfsd/nlm共4个模块可调试。
  ：顾名思义，调试rpc模块就是直接调试rpc的问题，将记录rpc相关的日志信息；
  ：调试nfs是调试nfs客户端的问题，将记录nfs客户端随之产生的日志信息；
  ：nfsd是调试nfs服务端问题，将记录nfsd随之产生的日志信息；
  ：nlm是调试nfs锁管理器相关问题，将只记录锁相关信息
-s：指定调试的修饰符，每个模块都有不同的修饰符，见上面的usage中"Valid flags"列的信息
-c：清除或清空已设置的调试flage
```

例如设置调试nfs客户端的信息。

```
rpcdebug -m nfs -s all
```

当有信息出现时，将记录到syslog中。例如以下是客户端挂载nfs导出目录产生的信息，存放在/var/log/messages中，非常多，所以排解问题时需要有耐心。

```
Jul 29 11:24:04 xuexi kernel: NFS: nfs mount opts='vers=4,addr=172.16.10.9,clientaddr=172.16.10.3'
Jul 29 11:24:04 xuexi kernel: NFS:   parsing nfs mount option 'vers=4'
Jul 29 11:24:04 xuexi kernel: NFS:   parsing nfs mount option 'addr=172.16.10.9'
Jul 29 11:24:04 xuexi kernel: NFS:   parsing nfs mount option 'clientaddr=172.16.10.3'
Jul 29 11:24:04 xuexi kernel: NFS: MNTPATH: '/tmp/testdir'
Jul 29 11:24:04 xuexi kernel: --> nfs4_try_mount()
Jul 29 11:24:04 xuexi kernel: --> nfs4_create_server()
Jul 29 11:24:04 xuexi kernel: --> nfs4_init_server()
Jul 29 11:24:04 xuexi kernel: --> nfs4_set_client()
Jul 29 11:24:04 xuexi kernel: --> nfs_get_client(172.16.10.9,v4)
Jul 29 11:24:04 xuexi kernel: NFS: get client cookie (0xffff88004c561800/0xffff8800364cd2c0)
Jul 29 11:24:04 xuexi kernel: nfs_create_rpc_client: cannot create RPC client. Error = -22
Jul 29 11:24:04 xuexi kernel: --> nfs4_realloc_slot_table: max_reqs=1024, tbl->max_slots 0
Jul 29 11:24:04 xuexi kernel: nfs4_realloc_slot_table: tbl=ffff88004b715c00 slots=ffff880063f32280 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_realloc_slot_table: return 0
Jul 29 11:24:04 xuexi kernel: NFS: nfs4_discover_server_trunking: testing '172.16.10.9'
Jul 29 11:24:04 xuexi kernel: NFS call  setclientid auth=UNIX, 'Linux NFSv4.0 172.16.10.3/172.16.10.9 tcp'
Jul 29 11:24:04 xuexi kernel: NFS reply setclientid: 0
Jul 29 11:24:04 xuexi kernel: NFS call  setclientid_confirm auth=UNIX, (client ID 578d865901000000)
Jul 29 11:24:04 xuexi kernel: NFS reply setclientid_confirm: 0
Jul 29 11:24:04 xuexi kernel: NFS: <-- nfs40_walk_client_list using nfs_client = ffff88004c561800 ({2})
Jul 29 11:24:04 xuexi kernel: NFS: <-- nfs40_walk_client_list status = 0
Jul 29 11:24:04 xuexi kernel: nfs4_schedule_state_renewal: requeueing work. Lease period = 5
Jul 29 11:24:04 xuexi kernel: NFS: nfs4_discover_server_trunking: status = 0
Jul 29 11:24:04 xuexi kernel: --> nfs_put_client({2})
Jul 29 11:24:04 xuexi kernel: <-- nfs4_set_client() = 0 [new ffff88004c561800]
Jul 29 11:24:04 xuexi kernel: <-- nfs4_init_server() = 0
Jul 29 11:24:04 xuexi kernel: --> nfs4_get_rootfh()
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=040000
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=4651240235397459983
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0x0/0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=2
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=0555
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=23
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=1501990255
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501989952
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501989952
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=1
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_supported: bitmask=fdffbfff:00f9be3e:00000000
Jul 29 11:24:04 xuexi kernel: decode_attr_fh_expire_type: expire type=0x0
Jul 29 11:24:04 xuexi kernel: decode_attr_link_support: link support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_symlink_support: symlink support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_aclsupport: ACLs supported=3
Jul 29 11:24:04 xuexi kernel: decode_server_caps: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_lease_time: file size=90
Jul 29 11:24:04 xuexi kernel: decode_attr_maxfilesize: maxfilesize=18446744073709551615
Jul 29 11:24:04 xuexi kernel: decode_attr_maxread: maxread=131072
Jul 29 11:24:04 xuexi kernel: decode_attr_maxwrite: maxwrite=131072
Jul 29 11:24:04 xuexi kernel: decode_attr_time_delta: time_delta=1 0
Jul 29 11:24:04 xuexi kernel: decode_attr_pnfstype: bitmap is 0
Jul 29 11:24:04 xuexi kernel: decode_attr_layout_blksize: bitmap is 0
Jul 29 11:24:04 xuexi kernel: decode_fsinfo: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: <-- nfs4_get_rootfh() = 0
Jul 29 11:24:04 xuexi kernel: Server FSID: 0:0
Jul 29 11:24:04 xuexi kernel: Pseudo-fs root FH at ffff880064c4ad80 is 8 bytes, crc: 0x62d40c52:
Jul 29 11:24:04 xuexi kernel: 01000100 00000000
Jul 29 11:24:04 xuexi kernel: --> nfs_probe_fsinfo()
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_supported: bitmask=fdffbfff:00f9be3e:00000000
Jul 29 11:24:04 xuexi kernel: decode_attr_fh_expire_type: expire type=0x0
Jul 29 11:24:04 xuexi kernel: decode_attr_link_support: link support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_symlink_support: symlink support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_aclsupport: ACLs supported=3
Jul 29 11:24:04 xuexi kernel: decode_server_caps: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_lease_time: file size=90
Jul 29 11:24:04 xuexi kernel: decode_attr_maxfilesize: maxfilesize=18446744073709551615
Jul 29 11:24:04 xuexi kernel: decode_attr_maxread: maxread=131072
Jul 29 11:24:04 xuexi kernel: decode_attr_maxwrite: maxwrite=131072
Jul 29 11:24:04 xuexi kernel: decode_attr_time_delta: time_delta=1 0
Jul 29 11:24:04 xuexi kernel: decode_attr_pnfstype: bitmap is 0
Jul 29 11:24:04 xuexi kernel: decode_attr_layout_blksize: bitmap is 0
Jul 29 11:24:04 xuexi kernel: decode_fsinfo: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: set_pnfs_layoutdriver: Using NFSv4 I/O
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_maxlink: maxlink=255
Jul 29 11:24:04 xuexi kernel: decode_attr_maxname: maxname=255
Jul 29 11:24:04 xuexi kernel: decode_pathconf: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: <-- nfs_probe_fsinfo() = 0
Jul 29 11:24:04 xuexi kernel: <-- nfs4_create_server() = ffff88007746a800
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_supported: bitmask=fdffbfff:00f9be3e:00000000
Jul 29 11:24:04 xuexi kernel: decode_attr_fh_expire_type: expire type=0x0
Jul 29 11:24:04 xuexi kernel: decode_attr_link_support: link support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_symlink_support: symlink support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_aclsupport: ACLs supported=3
Jul 29 11:24:04 xuexi kernel: decode_server_caps: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=040000
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=4651240235397459983
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0x0/0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=2
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=0555
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=23
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=1501990255
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501989952
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501989952
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=1
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS: nfs_fhget(0:38/2 fh_crc=0x62d40c52 ct=1)
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=00
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=4651240235397459983
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0x0/0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=00
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=1
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=-2
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=-2
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=0
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=0
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501989952
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501989952
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS: nfs_update_inode(0:38/2 fh_crc=0x62d40c52 ct=2 info=0x26040)
Jul 29 11:24:04 xuexi kernel: NFS: permission(0:38/2), mask=0x1, res=0
Jul 29 11:24:04 xuexi kernel: NFS: permission(0:38/2), mask=0x81, res=0
Jul 29 11:24:04 xuexi kernel: NFS: lookup(/tmp)
Jul 29 11:24:04 xuexi kernel: NFS call  lookup tmp
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=040000
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=16540743250786234113
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0xf199fcb4fb064bf5/0xa1b7a15af0f7cb47)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=391681
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=01777
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=5
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=1501990260
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=391681
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS reply lookup: 0
Jul 29 11:24:04 xuexi kernel: NFS: nfs_fhget(0:38/391681 fh_crc=0xb4775a3f ct=1)
Jul 29 11:24:04 xuexi kernel: --> nfs_d_automount()
Jul 29 11:24:04 xuexi kernel: nfs_d_automount: enter
Jul 29 11:24:04 xuexi kernel: NFS call  lookup tmp
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=040000
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=16540743250786234113
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0xf199fcb4fb064bf5/0xa1b7a15af0f7cb47)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=391681
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=01777
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=5
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=1501990260
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=391681
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS reply lookup: 0
Jul 29 11:24:04 xuexi kernel: --> nfs_do_submount()
Jul 29 11:24:04 xuexi kernel: nfs_do_submount: submounting on /tmp
Jul 29 11:24:04 xuexi kernel: --> nfs_xdev_mount()
Jul 29 11:24:04 xuexi kernel: --> nfs_clone_server(,f199fcb4fb064bf5:a1b7a15af0f7cb47,)
Jul 29 11:24:04 xuexi kernel: --> nfs_probe_fsinfo()
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_supported: bitmask=fdffbfff:00f9be3e:00000000
Jul 29 11:24:04 xuexi kernel: decode_attr_fh_expire_type: expire type=0x0
Jul 29 11:24:04 xuexi kernel: decode_attr_link_support: link support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_symlink_support: symlink support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_aclsupport: ACLs supported=3
Jul 29 11:24:04 xuexi kernel: decode_server_caps: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_lease_time: file size=90
Jul 29 11:24:04 xuexi kernel: decode_attr_maxfilesize: maxfilesize=18446744073709551615
Jul 29 11:24:04 xuexi kernel: decode_attr_maxread: maxread=131072
Jul 29 11:24:04 xuexi kernel: decode_attr_maxwrite: maxwrite=131072
Jul 29 11:24:04 xuexi kernel: decode_attr_time_delta: time_delta=1 0
Jul 29 11:24:04 xuexi kernel: decode_attr_pnfstype: bitmap is 0
Jul 29 11:24:04 xuexi kernel: decode_attr_layout_blksize: bitmap is 0
Jul 29 11:24:04 xuexi kernel: decode_fsinfo: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: set_pnfs_layoutdriver: Using NFSv4 I/O
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_maxlink: maxlink=255
Jul 29 11:24:04 xuexi kernel: decode_attr_maxname: maxname=255
Jul 29 11:24:04 xuexi kernel: decode_pathconf: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: <-- nfs_probe_fsinfo() = 0
Jul 29 11:24:04 xuexi kernel: Cloned FSID: f199fcb4fb064bf5:a1b7a15af0f7cb47
Jul 29 11:24:04 xuexi kernel: <-- nfs_clone_server() = ffff88006f407000
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_supported: bitmask=fdffbfff:00f9be3e:00000000
Jul 29 11:24:04 xuexi kernel: decode_attr_fh_expire_type: expire type=0x0
Jul 29 11:24:04 xuexi kernel: decode_attr_link_support: link support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_symlink_support: symlink support=true
Jul 29 11:24:04 xuexi kernel: decode_attr_aclsupport: ACLs supported=3
Jul 29 11:24:04 xuexi kernel: decode_server_caps: xdr returned 0!
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=040000
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=16540743250786234113
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0xf199fcb4fb064bf5/0xa1b7a15af0f7cb47)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=391681
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=01777
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=5
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=1501990260
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=391681
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS: nfs_fhget(0:40/391681 fh_crc=0xb4775a3f ct=1)
Jul 29 11:24:04 xuexi kernel: <-- nfs_xdev_mount() = 0
Jul 29 11:24:04 xuexi kernel: nfs_do_submount: done
Jul 29 11:24:04 xuexi kernel: <-- nfs_do_submount() = ffff880064fdfb20
Jul 29 11:24:04 xuexi kernel: nfs_d_automount: done, success
Jul 29 11:24:04 xuexi kernel: <-- nfs_d_automount() = ffff880064fdfb20
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=00
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=16540743250786234113
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0x0/0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=00
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=1
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=-2
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=-2
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=0
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=0
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1501990117
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS: nfs_update_inode(0:40/391681 fh_crc=0xb4775a3f ct=2 info=0x26040)
Jul 29 11:24:04 xuexi kernel: NFS: permission(0:40/391681), mask=0x1, res=0
Jul 29 11:24:04 xuexi kernel: NFS: lookup(/testdir)
Jul 29 11:24:04 xuexi kernel: NFS call  lookup testdir
Jul 29 11:24:04 xuexi kernel: --> nfs4_alloc_slot used_slots=0000 highest_used=4294967295 max_slots=1024
Jul 29 11:24:04 xuexi kernel: <-- nfs4_alloc_slot used_slots=0001 highest_used=0 slotid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_type: type=040000
Jul 29 11:24:04 xuexi kernel: decode_attr_change: change attribute=7393570666420598027
Jul 29 11:24:04 xuexi kernel: decode_attr_size: file size=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_fsid: fsid=(0xf199fcb4fb064bf5/0xa1b7a15af0f7cb47)
Jul 29 11:24:04 xuexi kernel: decode_attr_fileid: fileid=391682
Jul 29 11:24:04 xuexi kernel: decode_attr_fs_locations: fs_locations done, error = 0
Jul 29 11:24:04 xuexi kernel: decode_attr_mode: file mode=0754
Jul 29 11:24:04 xuexi kernel: decode_attr_nlink: nlink=3
Jul 29 11:24:04 xuexi kernel: decode_attr_owner: uid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_group: gid=0
Jul 29 11:24:04 xuexi kernel: decode_attr_rdev: rdev=(0x0:0x0)
Jul 29 11:24:04 xuexi kernel: decode_attr_space_used: space used=4096
Jul 29 11:24:04 xuexi kernel: decode_attr_time_access: atime=1501990266
Jul 29 11:24:04 xuexi kernel: decode_attr_time_metadata: ctime=1497209702
Jul 29 11:24:04 xuexi kernel: decode_attr_time_modify: mtime=1496802409
Jul 29 11:24:04 xuexi kernel: decode_attr_mounted_on_fileid: fileid=391682
Jul 29 11:24:04 xuexi kernel: decode_getfattr_attrs: xdr returned 0
Jul 29 11:24:04 xuexi kernel: decode_getfattr_generic: xdr returned 0
Jul 29 11:24:04 xuexi kernel: nfs4_free_slot: slotid 0 highest_used_slotid 4294967295
Jul 29 11:24:04 xuexi kernel: NFS reply lookup: 0
Jul 29 11:24:04 xuexi kernel: NFS: nfs_fhget(0:40/391682 fh_crc=0xec69f317 ct=1)
Jul 29 11:24:04 xuexi kernel: NFS: dentry_delete(/tmp, 202008c)
Jul 29 11:24:04 xuexi kernel: NFS: clear cookie (0xffff88005eaf4620/0x          (null))
Jul 29 11:24:04 xuexi kernel: NFS: clear cookie (0xffff88005eaf4a40/0x          (null))
Jul 29 11:24:04 xuexi kernel: NFS: releasing superblock cookie (0xffff88007746a800/0x          (null))
Jul 29 11:24:04 xuexi kernel: --> nfs_free_server()
Jul 29 11:24:04 xuexi kernel: --> nfs_put_client({2})
Jul 29 11:24:04 xuexi kernel: <-- nfs_free_server()
Jul 29 11:24:04 xuexi kernel: <-- nfs4_try_mount() = 0
```

## 深入NFS的方向

本文仅简单介绍了一些NFS的基本使用方法，但NFS自身其实是很复杂的，想深入也不是一件简单的事。例如，以下列出了几个NFS在实现上应该要解决的问题。

(1).多台客户端挂载同一个导出的目录后，要同时编辑其中同一个文件时，应该如何处理？这是共享文件更新问题，通用的解决方法是使用文件锁。

(2).客户端上对文件内容做出修改后是否要立即同步到服务端上？这是共享文件的数据缓存问题，体现的方式是文件数据是否要在各客户端上保证一致性。和第一个问题结合起来，就是分布式(集群)文件数据一致性问题。

(3).客户端或服务端进行了重启，或者它们出现了故障，亦或者它们之间的网络出现故障后，它们的对端如何知道它已经出现了故障？这是故障通知或重启通知问题。

(4).出现故障后，正常重启成功了，那么如何恢复到故障前状态？这是故障恢复问题。

(5).如果服务端故障后一直无法恢复，客户端是否能自动故障转移到另一台正常工作的NFS服务节点？这是高可用问题。(NFS版本4(后文将简写为NFSv4)可以从自身实现故障转移，当然使用高可用工具如heartbeat也一样能实现)

总结起来就几个关键词：**锁、缓存、数据和缓存一致性、通知和故障恢复**。从这些关键字中，很自然地会联想到了集群和分布式文件系统，其实NFS也是一种简易的分布式文件系统，但没有像集群或分布式文件系统那样实现几乎完美的锁、缓存一致性等功能。而且NFSv4为了体现其特点和性能，在一些通用问题上采用了与集群、分布式文件系统不同的方式，如使用了文件委托(服务端将文件委托给客户端，由此实现类似锁的功能)、锁或委托的租约等(就像DHCP的IP租约一样，在租约期内，锁和委托以及缓存是有效的，租约过期后这些内容都无效)。

由此知，深入NFS其实就是在深入分布式文件系统，由于网上对NFS深入介绍的文章几乎没有，所以想深入它并非一件简单的事。如果有深入学习的想法，可以阅读[NFSv4的RFC3530文档](https://www.ietf.org/rfc/rfc3530.txt)的前9章以及[RCP的RFC文档](https://tools.ietf.org/html/rfc5531)。以下是本人的一些man文档翻译。

> ```
> 翻译：man rpcbind(rpcbind中文手册)
> 翻译：man nfsd(rpc.nfsd中文手册)
> 翻译：man mountd(rpc.mountd中文手册)
> 翻译：man statd(rpc.statd中文手册)
> 翻译：man sm-notify(sm-notify命令中文手册)
> 翻译：man exportfs(exportfs命令中文手册)
> ```

以下是man nfs中关于传输方法和缓存相关内容的翻译。

```
TRANSPORT METHODS
       NFS客户端是通过RPC向NFS服务端发送请求的。RPC客户端会自动
       发现远程服务端点，处理每请求(per-request)的身份验证，调整
       当客户端和服务端之间出现不同字节字节序(byte endianness)时
       的请求参数，并且当请求在网络上丢失或被服务端丢弃时重传请求。
       rpc请求和响应数据包是通过网络传输的。

       在大多数情形下，mount(8)命令、NFS客户端和NFS服务端可以为挂
       载点自动协商合适的传输方式以及数据传输时的大小。但某些情况下，
       需要使用mount选项显式指定这些设置。
 
       对于的传统NFS，NFS客户端只使用UDP传输请求给服务端。尽管实现
       方式很简单，但基于UDP的NFS有许多限制，在一些通用部署环境下，
       这些限制项会限制平滑运行的能力，还会限制性能。即使是UDP丢包率
       极小的情况，也将导致整个NFS请求丢失；因此，重传的超时时间一般
       都是亚秒级的，这样可以让客户端从请求丢失中快速恢复，但即使如此，
       这可能会导致无关的网络阻塞以及服务端负载加重。
 
       但是在专门设置了网络MTU值是相对于NFS的输出传输大小时(例如在网络
       环境下启用了巨型以太网帧)(注：相对的意思，例如它们成比例关系)，
       UDP是非常高效的。在这种环境下，建议修剪rsize和wsize，以便让每个
       NFS的读或写请求都能容纳在几个网络帧(甚至单个帧)中。这会降低由于
       单个MTU大小的网络帧丢失而导致整个读或写请求丢失的概率。
      
       (译注：UDP是NFSv2和NFSv3所支持的，NFSv4使用TCP，所以以上两段不用考虑)
 
       现代NFS都默认采用TCP传输协议。在几乎所有可想到的网络环境下，TCP
       都表现良好，且提供了非常好的保证，防止因网络不可靠而引起数据损坏。
       但使用TCP传输协议，基本上都会考虑设置相关防火墙。
 
       在正常环境下，网络丢包的频率比NFS服务丢包的频率要高的多。因此，
       没有必要为基于TCP的NFS设置极短的重传超时时间。一般基于TCP的NFS
       的超时时间设置在1分钟和10分钟之间。当客户端耗尽了重传次数(选项
       retrans的值)，它将假定发生了网络分裂，并尝试以新的套接字重新连
       接服务端。由于TCP自身使得网络传输的数据是可靠的，所以，可以安全
       地使用处在默认值和客户端服务端同时支持的最大值之间的rszie和wsize，
       而不再依赖于网络MTU大小。
 
DATA AND METADATA COHERENCE
       现在有些集群文件系统为客户端之间提供了非常完美的缓存一致性
       功能，但对于NFS客户端来说，实现完美的缓存一致性是非常昂贵
       的，特别是大型局域网络环境。因此，NFS提供了稍微薄弱一点的
       缓存一致性功能，以满足大多数文件共享需求。
 
   Close-to-open cache consistency
       一般情况下，文件共享是完全序列化的。首先客户端A打开一个文件，
       写入一些数据，然后关闭文件然后客户端B打开同一个文件，并读取
       到这些修改后的数据。
      
       当应用程序打开一个存储在NFSv3服务端上的文件时，NFS客户端检查
       文件在服务端上是否存在，并通过是否发送GETATTR或ACCESS请求判
       断文件是否允许被打开。NFS客户端发送这些请求时，不会考虑已缓存
       文件属性是否是新鲜有效的。
 
       当应用程序关闭文件，NFS客户端立即将已做的修改写入到文件中，以
       便下一个打开者可以看到所做的改变。这也给了NFS客户端一个机会，
       使得它可以通过文件关闭时的返回状态码向应用程序报告错误。
 
       在打开文件时以及关闭文件刷入数据时的检查行为被称为close-to-open
       缓存一致性，简称为CTO。可通过使用nocto选项来禁用整个挂载点的CTO。
      
       (译者注：NFS关闭文件时，客户端会将所做的修改刷入到文件中，然后
       发送GETATTR请求以确保该文件的属性缓存已被更新。之后其他的打开者
       打开文件时会发送GETATTR请求，根据文件的属性缓存可以判断文件是
       否被打开并做了修改。这可以避免缓存无效)
 
   Weak cache consistency
       客户端的数据缓存仍有几乎包含过期数据。NFSv3协议引入了"weak cache
       consistency"(WCC)，它提供了一种在单个文件被请求之前和之后有效地
       检查文件属性的方式。这有助于客户端识别出由其他客户端对此文件做出
       的改变。
 
       当某客户端使用了并发操作，使得同一文件在同一时间做出了多次更新
       (例如，后台异步写入)，它仍将难以判断是该客户端的更新操作修改了
       此文件还是其他客户端的更新操作修改了文件。
 
   Attribute caching
       使用noac挂载选项可以让多客户端之间实现属性缓存一致性。几乎每个
       文件系统的操作都会检查文件的属性信息。客户端自身会保留属性缓存
       一段时间以减少网络和服务端的负载。当noac生效时，客户端的文件属
       性缓存就会被禁用，因此每个会检查文件属性的操作都被强制返回到服
       务端上来操作文件，这表示客户端以牺牲网络资源为代价来快速查看文
       件发生的变化。
 
       不要混淆noac选项和"no data caching"。noac挂载选项阻止客户端
       缓存文件的元数据，但仍然可能会缓存非元数据的其他数据。
 
       NFS协议设计的目的不是为了支持真正完美的集群文件系统缓存一致性。
       如果要达到客户端之间缓存数据的绝对一致，那么应该使用文件锁的方式。
 
   File timestamp maintainence
       NFS服务端负责管理文件和目录的时间戳(atime,ctime,mtime)。当
       服务端文件被访问或被更新，文件的时间戳也会像本地文件系统那样改变。
 
       NFS客户端缓存的文件属性中包括了时间戳属性。当NFS客户端检索NFS
       服务端文件属性时，客户端文件的时间戳会更新。因此，在NFS服务端的
       时间戳更新后显示在NFS客户端之前可能会有一些延迟。
 
       为了遵守POSIX文件系统标准，Linux NFS客户端需要依赖于NFS服务端
       来保持文件的mtime和ctime时间戳最新状态。实现方式是先让客户端将
       改变的数据刷入到服务端，然后再输出文件的mtime。
 
       然而，Linux客户端可以很轻松地处理atime的更新。NFS客户端可以通过
       缓存数据来保持良好的性能，但这意味着客户端应用程序读取文件(会更
       新atime)时，不会反映到服务端，但实际上服务端的atime已经修改了。
 
       由于文件属性缓存行为，Linux NFS客户端mount时不支持一般的atime
       相关的挂载选项。
 
       特别是mount时指定了atime/noatime，diratime/nodiratime，
       relatime/norelatime以及strictatime/nostrictatime挂载选项，
       实际上它们没有任何效果。
 
   Directory entry caching
       Linux NFS客户端缓存所有NFS LOOKUP请求的结果。如果所请求目录
       项在服务端上存在，则查询的结果称为正查询结果。如果请求的目录在
       服务端上不存在(服务端将返回ENOENT)，则查询的结果称为负查询结果。
 
       为了探测目录项是否添加到NFS服务端或从其上移除了，Linux NFS客
       户端会监控目录的mtime。如果客户端监测到目录的mtime发生了改变，
       客户端将丢弃该目录相关的所有LOOKUP缓存结果。由于目录的mtime是
       一种缓存属性，因此服务端目录mtime发生改变后，客户端可能需要等一
       段时间才能监测到它的改变。
 
       缓存目录提高了不与其他客户端上的应用程序共享文件的应用程序的性
       能。但使用目录缓存可能会干扰在多个客户端上同时运行的应用程序，
       并且需要快速检测文件的创建或删除。lookupcache挂载选项允许对目
       录缓存行为进行一些调整。
     
       如果客户端禁用目录缓存，则每次LOOKUP操作都需要与服务端进行验证，
       那么该客户端可以立即探测到其他客户端创建或删除的目录。可以使用
       lookupcache=none来禁用目录缓存。如果禁用了目录缓存，由于需要额
       外的NFS请求，这会损失一些性能，但禁用目录缓存比使用noac损失的性
       能要少，且对NFS客户端缓存文件属性没有任何影响。
 
   The sync mount option
       NFS客户端对待sync挂载选项的方式和挂载其他文件系统不同。如果即不
       指定sync也不指定async，则默认为async。async表示异步写入，除了
       发生下面几种特殊情况，NFS客户端会延迟发送写操作到服务端：
       ● 内存压力迫使回收系统内存资源。
       ● 客户端应用程序显式使用了sync类系统调用，如sync(2)/msync(2)/fsync(3)。
       ● 关闭文件时。
       ● 文件被锁/解锁。
 
       换句话说，在正常环境下，应用程序写的数据不会立即保存到服务端。
       (当然，关闭文件时是立即同步的)
 
       如果在挂载点上指定sync选项，任何将数据写入挂载点上文件的系统
       调用都会先把数据刷到服务端上，然后系统调用才把控制权返回给用
       户空间。这提供了客户端之间更好的数据缓存一致性，但消耗了大量
       性能。
 
       在未使用sync挂载选项时，应用程序可以使用O_SYNC修饰符强制立即
       把单个文件的数据刷入到服务端。
 
   Using file locks with NFS
       网络锁管理器(Network Lock Manager,NLM)协议是一个独立的协议，
       用于管理NFSv2和NFSv3的文件锁。为了让客户端或服务端在重启后能
       恢复锁，需要使用另一个网络状态管理器(Network Status Manager,
       NSM)协议。在NFSv4中，NFS直协议自身接支持文件锁相关功能，所以
       NLM和NSM就不再使用了。
 
       在大多数情况下，NLM和NSM服务是自动启动的，并且不需要额外的任
       何配置。但需要配置NFS客户端使用的是fqdn名称，以保证NFS服务端
       可以找到客户端并通知它们服务端的重启动作。
 
       NLS仅支持advisory文件锁。要锁定NFS文件，可以使用待F_GETLK和
       F_SETLK的fcntl(2)命令。NFS客户端会转换通过flock(2)获取到的
       锁为advisory文件锁。
 
       当服务端不支持NLM协议，或者当NFS服务端是通过防火墙但阻塞了NLM
       服务端口时，则需要指定nolock挂载选项。当客户端挂载导出的/var
       目录时，必须使用nolock禁用NLM锁，因为/var目录中包含了NLM锁相
       关的信息。
       (注，因为NLM仅在nfsv2和nfsv3中支持，所以NFSv4不支持nolock选项)
 
       当仅有一个客户端时，使用nolock选项可以提高一定的性能。
 
   NFS version 4 caching features
       NFSv4上的数据和元数据缓存行为和之前的版本相似。但是NFSv4添加了
       两种特性来提升缓存行为：change attributes以及delegation。
       (注：nfsv4中取消了weak cache consistency)
 
       change attribute是一种新的被跟踪的文件/目录元数据信息。它替代
       了使用文件mtime和ctime作为客户端验证缓存内容的方式。但注意，
       change attributes和客户端/服务端文件的时间戳的改变无关。
 
       文件委托(file delegation)是NFSv4的客户端和NFSv4服务端之前的
       一种合约，它允许客户端临时处理文件，就像没有其他客户端正在访问
       该文件一样。当有其他客户端尝试访问被委托文件时，服务端一定会通
       知服务端(通过callback请求)。一旦文件被委托给某客户端，客户端可
       以尽可能大地缓存该文件的数据和元数据，而不需要联系服务端。
 
       文件委托有两种方式：read和write。读委托意味着当其他客户端尝
       试向委托文件写入数据时，服务端通知客户端。而写委托意味着其他客
       户端无论是尝试读还是写，服务端都会通知客户端。
 
       NFSv4上当文件被打开的时候，服务端会授权文件委托，并且当其他客
       户端想要访问该文件但和已授权的委托冲突时，服务端可以在任意时间
       点重调(recall)委托关系。不支持对目录的委托。
       (译注：只有读委托和读委托不会冲突)
 
       为了支持委托回调(delegation callback)，在客户端初始联系服务端
       时，服务端会检查网络并返回路径给客户端。如果和客户端的联系无法建
       立，服务端将不会授权任何委托给客户端。
```