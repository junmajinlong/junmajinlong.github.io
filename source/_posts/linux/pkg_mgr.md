---
title: 学习CentOS的包管理
p: linux/pkg_mgr.md
date: 2019-07-07 11:19:46
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 学习CentOS的包管理

## 包基础知识

### 包名称

在rhel/centos/fedora上，包的名称以rpm结尾，它们分为二进制包和源码包。

源码包以`.src.rpm`结尾，它是未编译过的包，可以自行进行编译或者用其制作自己的二进制rpm包。非`.src.rpm`结尾的包都是二进制包，都是已经编译完成的。

安装rpm包的过程实际上就是将包中的文件复制到Linux上，有可能还会在复制文件的前后执行一些命令，如创建一个必要的用户，删除非必要文件等。关于rpm包的制作，可能是件很烦人的事，但学习它还是非常有必要的，至少也得能制作简单的包吧？网上有很多文章，也可以参考官网：

- <https://docs.fedoraproject.org/en-US/Fedora_Draft_Documentation/0.1/html/Packagers_Guide/index.html>

- <https://docs.fedoraproject.org/en-US/Fedora_Draft_Documentation/0.1/html/RPM_Guide/index.html>  

注意区分源码包和源码的概念，源码一般是打包压缩后的文件(如`.tar.gz`结尾的文件)。源码包中包含了源码，还包含了一些有助于制作二进制rpm的文件。最有力的说明就是源码编译安装的程序都没有服务启动脚本(/etc/init.d/下对应的启动脚本)，而二进制rpm包安装的就有，因为二进制rpm包都是通过源码包`.src.rpm`定制而来，在源码包中提供了必要的文件(如服务启动脚本)，并在安装rpm的时候复制到指定路径下。

回归正题，一个rpm包的名称分为包全名和包名，包全名如`httpd-2.2.15-39.el6.centos.x86_64.rpm`，包全名中各部分的意义如下：

```
httpd   	  包名
2.2.15  	  版本号，版本号格式[ 主版本号.[ 次版本号.[ 修正号 ] ] ]
39          软件发布次数
el6.centos  适合的操作系统平台以及适合的操作系统版本
x86_64		 适合的硬件平台，硬件平台根据cpu来决定，
            有i386、i586、i686、x86_64、noarch或者省略，noarch或省略表示不区分硬件平台
rpm			   软件包后缀扩展名
```

使用rpm工具管理包时，如果要操作未安装的包，则使用包全名，如安装包，查看未安装包的信息等；如果要操作已安装的rpm包，则只需要给定其包名即可，如查询已装包生成了哪些文件，查看已装包的信息等。

而对于yum工具来说，只需给定其包名即可，若有需要，再指定版本号，如明确指明要安装1.6.10版本的tree工具，`yum install tree-1.6.10`。

### 主包和子包

对于一个程序而言，在制作rpm包时，很多时候都将其按功能分割成多个子包，如客户端程序包、服务端程序包等。

以mysql这个程序来说，它分有以下几个包。

```
mysql-server.x86_64
mysql.x86_64
mysql-bench.x86_64
mysql-libs.x86_64
mysql-devel.x86_64
```

其中mysql-server.x86_64是提供服务的主包，mysql.x86_64是客户端主包，mysql-bench是用于对MySQL进行压力测试的包，mysql-libs和mysql-devel分别是库文件包和头文件包。后两者是提供给其他需要联合mysql的程序使用的，仅就实现mysql服务而言，只需安装mysql-server即可。

而源码编译安装的包会包含所有功能包，也就是说编译安装一个程序后，它的客户端工具、服务提供程序、库文件、头文件等等都已经安装了。

## rpm管理包

rpm包被安装后，会在/var/lib/rpm下会建立已装rpm数据库，以后有任何rpm的升级、查询、版本比较等包的操作都是从这个目录下获取信息并完成相应操作的。

```
[root@xuexi ~]# ls /var/lib/rpm/  
Basenames     __db.003     Group           Packages        
Requirename   Triggername  Conflictname    __db.004
Installtid    Providename  Requireversion  __db.001
Dirnames     Name          Provideversion  Sha1header
__db.002     Filedigests   Obsoletename    Pubkeys
Sigmd5
```

### 安装包后的文件分布

rpm安装完成后，相关的文件会复制到多个目录下(具体复制的路径是在制作rpm包时指定的)。一般来说，分布形式差不多如下表。

| 路径                               | 一般作用                         |
| ---------------------------------- | -------------------------------- |
| /etc                               | 放置配置文件的目录               |
| /bin、/sbin、/usr/bin或/usr/sbin   | 一些可执行文件                   |
| /lib、/lib64、/usr/lib(/usr/lib64) | 一些库文件                       |
| /usr/include                       | 一些头文件                       |
| /usr/share/doc                     | 一些基本的软件使用手册与帮助文件 |
| /usr/share/man                     | 一些 man page 档案               |

### rpm安装、升级、卸载(install、update、uninstall)

rpm工具安装、升级和卸载的功能都很少使用。对于安装来说，它需要人为解决包的依赖关系，这是极其令人恶心的事对于升级来说，基本上都会使用yum工具进行安装和升级，而卸载行为在Linux上很少出现，大不了直接覆盖重装。

```
rpm -ivhUe --nodeps --test --force --prefix

选项说明：
-i 表示安装，install的意思
-v 显示安装信息，还可以"-vv"、"-vvv"，v提供的越多显示信息越多
-h 显示安装进度，以#显示安装的进度
-U 升级或升级包
-F 只升级已安装的包 
-e 卸载包，卸载也有依赖性,"--erase"
--nodeps 忽略依赖性强制安装或卸载(no dependencies)
--test 测试是否能够成功安装指定的rpm包
--prefix 新路径 自行指定安装路径而不是使用默认路径，基本上都不支持该功能，功能极其简单的软件估计才支持重定位安装路径
--force 强制动作
--replacepkgs 替换安装，即重新覆盖安装。
```

有时误删文件可以不用卸载再装，直接使用`--replacepkgs`选项再次安装即可。

rpm包另一个缺陷是只能安装本地或给定url路径的rpm包。

注意：尽量不要对内核进行升级；多版本的内核可以并存，因此可以执行安装操作。

### rpm查询功能

rpm工具的安装功能很少使用，毕竟解决依赖关系不是件容易的事。但是rpm的查询功能则非常实用。

```
-q[p] -q查询已安装的包，-qp查询未安装的包。它们都可接下面的参数
	-a 查询所有已安装的包，也可以指定通配符名称进行查询
	-i 查询指定包的信息（版本、开发商、安装时间等）。从这里面可以查看到软件包属于哪个包组。
	-l 查询包的文件列表和目录（包在生产的时候就指定了文件路径，因此可查未装包）
	-R 查询包的依赖性(Required)
	-c 查询安装后包生成的配置文件
	-d 查询安装后包生成的帮助文档
-f 查询系统文件属于哪个已安装的包(接的是文件而不是包)
--scripts 查询包相关的脚本文档。脚本文档分四类：安装前运行、安装后运行、卸载前运行、卸载后运行
```

例如：

(1).查询文件/etc/yum.conf是通过哪个包安装的。

```
[root@xuexi cdrom]# rpm -qf /etc/yum.conf 
yum-3.2.29-60.el6.centos.noarch
```

(2).查询安装httpd时生成了哪些目录和文件，还可以过滤出提供了哪些命令行工具。

```
rpm -ql httpd
rpm -ql httpd | grep 'bin/'
```

(3).查询某个未安装包的依赖性如zip-3.0-1.el6.x86_64.rpm的依赖性。

```
[root@xuexi cdrom]# rpm -qRp zip-3.0-1.el6.x86_64.rpm
libc.so.6()(64bit)  
libc.so.6(GLIBC_2.2.5)(64bit)  
libc.so.6(GLIBC_2.3)(64bit)  
libc.so.6(GLIBC_2.3.4)(64bit)  
libc.so.6(GLIBC_2.4)(64bit)  
libc.so.6(GLIBC_2.7)(64bit)  
rpmlib(CompressedFileNames) <= 3.0.4-1
rpmlib(FileDigests) <= 4.6.0-1
rpmlib(PayloadFilesHavePrefix) <= 4.0-1
rtld(GNU_HASH)  
rpmlib(PayloadIsXz) <= 5.2-1
```

实际上，查看包的依赖性时，使用yum-utils包中的repoquery工具更好，`repoquery -R pkg_name`会更简洁。yum-utils中包含了好几个非常实用的包管理工具，可以大致都了解下。

### 提取rpm包中文件

安装rpm包会安装rpm中所有文件，如果将某个文件删除了，除了重装rpm，还可以通过从rpm包提取缺失文件的方式来修复。在win上安装个万能压缩工具【好压】(嫌其流氓可以换其他工具)，可以直接打开rpm包，然后从中解压需要的文件出来。但是在Linux上，过程还是有点小复杂的，其中涉及了cpio这个古来的归档工具。

方法：使用rpm2cpio命令组合`cpio -idv`命令的方式来提取。cpio具体用法参见[cpio归档工具用法详细说明](https://www.junmajinlong.com/linux/cpio_usage)。

`rpm2cpio`是将rpm转换为cpio格式归档文件的命令，有了cpio文件，就可以使用cpio命令对其进行相关的操作。

cpio命令是从归档文件中提取文件或向归档文件中写入文件的工具，一般都从标准输入或输出操作归档文件，所以都使用管道或重定向符号。

```
-i： 运行在copy-in模式，即从归档文件中将文件copy出来，即提取文件（提取）
-o： 运行在copy-out模式，将文件copy到归档文件中，即将文件拷贝到归档文件中（写入）
-d： 需要目录时自动建立目录
-v： 显示信息
```

提取rpm包文件的一般格式为以下格式：

```
rpm2cpio package_full_name|cpio -idv dir_name
```

例如，删除/bin/ls文件，将导致ls命令不可用，使用文件提取的方式去修复。

```
[root@xuexi cdrom]# which ls     # 查找需要删除的ls文件位置
alias ls='ls --color=auto'
        /bin/ls
[root@xuexi cdrom]# rpm -qf /bin/ls   # 查找ls命令属于哪个包
coreutils-8.4-37.el6.x86_64
[root@xuexi cdrom]# rm -f /bin/ls      # 删除ls命令
[root@xuexi cdrom]# ls					# ls命令已不可用
-bash: /bin/ls: No such file or directory
[root@xuexi ~]# yumdownloader coreutils		# 下载所需的rpm包
[root@xuexi ~]# rpm2cpio coreutils-8.4-37.el6.x86_64.rpm | cpio -id ./bin/ls   # 提取bin/ls到当前目录下
[root@xuexi ~]# dir ~/bin     # 使用dir命令查看已经提取成功，dir命令功能等价于ls
ls
[root@xuexi tmp]# cp bin/ls /bin/ # 复制ls命令到/bin下
[root@xuexi tmp]# ls				     # 测试，ls已经可用
```

## yum管理包

yum工具通过仓库的方式简化rpm包的管理。它从仓库中搜索相关的软件包，并自动下载和解决软件包的依赖性，非常方便。

### /etc/yum.conf

/etc/yum.conf是yum的默认文件，里面配置的也是全局默认项。

```
[root@server2 ~]# cat /etc/yum.conf
[main]
cachedir=/var/cache/yum/$basearch/$releasever   # 缓存目录
keepcache=0          # 是否保留缓存，设置为1时，安装包时所下载的包将不会被删除
debuglevel=2         # 调试信息的级别
logfile=/var/log/yum.log   # 日志文件位置
exactarch=1         # 设置为1将只会安装和系统架构完全匹配的包
obsoletes=1         # 是否允许更新旧的包
gpgcheck=1          # 是否要进行gpg check
plugins=1           # 是否允许使用yum插件
installonly_limit=5
bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?category=yum
distroverpkg=centos-release    # 指定基准包，yum会根据这个包判断发行版本
```

### 配置yum仓库

首先配置yum仓库，配置文件为/etc/yum.conf和/etc/yum.repos.d/中的`.repo`文件，其中/etc/yum.conf配置的是仓库的默认项，一般配置yum源都是在`/etc/yum.repos.d/*.repo`中配置。注意，该目录中任意repo文件都会被读取。

默认/etc/yum.repos.d/下会有以下几个仓库文件，除了CentOS-Base.repo，其他的都可以删掉，基本没用。

```
$ ls /etc/yum.repos.d/
CentOS-Base.repo       CentOS-fasttrack.repo  CentOS-Vault.repo
CentOS-Debuginfo.repo  CentOS-Media.repo
```

repo文件的配置格式如下：

```
$ vim CentOS-Base.repo
[base]      # 仓库ID，ID必须保证唯一性
name        # 仓库名称，可随意命名
mirrorlist  # 该地址下包含了仓库地址列表，包含一个或多个镜像站点，和baseurl使用一个就可以了
#baseurl    # 仓库地址。网络上的地址则写网络地址，
            # 本地地址则写本地地址，格式为"file://"后接路径，如file:///mnt/cdrom
gpgcheck=1  # 指定是否需要gpg签名，1表示需要，0表示不需要
gpgkey  =   # 签名文件的路径
enabled      # 该仓库是否生效，enabled=1表示生效，enabled=0表示不生效
cost=       # 开销越高，优先级越低
```

repo配置文件中默认可用的宏：

- `$releasever`：程序的版本(release version)，对yum而言指的是redhat-release版本。只替换为主版本号，如redhat6.5 则替换为6

- `$arch`：系统架构

- `$basharch`：系统基本架构，如i686，i586等的基本架构为i386

- `$YUM0-9`：在系统定义的环境变量，可以在yum中使用

也可以自己在/etc/yum/vars目录中定义变量文件。例如：

```
$ cat /etc/yum/vars/huawei 
https://mirrors.huaweicloud.com
```

然后就可以在repo的配置文件中使用变量`$huawei`：

```
$ cat /etc/yum.repos.d/base.repo 
[base]
name=os
baseurl=$huawei/centos/$releasever/os/$basearch/
gpgcheck=0
enabled=1
```

### repo配置示例：配置epel仓库

系统发行商在系统中放置的rpm包一般版本都较老，可能有些包有较大的延后性。而epel是由fedora社区维护的高质量高可靠性的安装源，有很多包是比系统包更新的，且多出很多系统没有的包。总之，用到epel的机会很多很多，所以就拿来当配置示例了。

有两种方式可以使用epel源。

方法一：安装epel-release-noarch.rpm

```
$ rpm -ivh epel-release-latest-6.noarch.rpm
```

安装后会在/etc/yum.repo.d/目录下生成两个epel相关的repo文件，其中一个是epel.repo。此文件中epel的源设置在了fedora的镜像站点上，这对国内网来说可能会较慢，可以修改它为下面的内容。

```
[epel]
name=Extra Packages for Enterprise Linux 6 - $basearch
baseurl=http://mirrors.sohu.com/fedora-epel/6Server/$basearch/
#mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-6&arch=$basearch
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
```

方法二：直接增加epel仓库

在/etc/yum.repos.d/下任意一个repo文件中添加上epel的仓库即可。

```
[epel]
name=epel
baseurl=http://mirrors.sohu.com/fedora-epel/6Server/$basearch/
enabled=1
gpgcheck=0
```

然后清除缓存再建立缓存即可。

```
yum clean all ; yum makecache 
```

### yum命令

不同的版本，yum命令可能功能上有所不同，例如在CentOS 7上的yum有`--downloadonly`的功能，而在CentOS 6.6上就没有(更新yum包后就有了)。此处介绍几个yum命令。

```
Usage: yum [options] COMMAND
 
List of Commands:
help         命令的帮助信息，用法：yum help command
clean        清除缓存数据，如yum clean all 
makecache    生成元数据缓存数据，yum makecache
deplist      列出包的依赖关系
erase        卸载包
fs           为当前文件系统创建快照，或者列出或删除当前已有快照。
             快照是非常有用的，升级或打补丁前拍个快照，就能放心地升级或打补丁了
fssnapshot   同fs一样
groups       操作包组
history      查看yum事务信息，yum是独占模式的进程，所以有时候查看事务信息还是有用的
info         输出包或包组的信息，例如该包是谁制作的，大概是干什么用的，来源于哪个包组等信息
install      包安装命令
list         列出包名，一般会结合grep来搜索包，如yum list all | grep -i zabbix
provides     搜索给定的内容是谁提供的，可用来搜索文件来源于哪个包，如
             yum provides '*bin/mysql'可搜索mysql命令来源于何处
reinstall    重新安装包
repolist     列出可用的仓库列表
search       给定字符串搜索相关包，并给出相关包较为详细的信息
update       更新包
 
Options:
  -R [minutes], --randomwait=[minutes]：最多等待时间
  -q, --quiet           安静模式
  -v, --verbose         详细模式
  -y, --assumeyes       对所有问题回答yes
  --assumeno            对所有问题回答no
  --enablerepo=[repo]   启用一个或多个仓库，可用通配符通配仓库ID
  --disablerepo=[repo]  禁用一个或多个仓库，可用通配符通配仓库ID
  -x [package], --exclude=[package]  通配要排除的包
  --nogpgcheck          禁用gpgcheck
  --color=COLOR         带颜色
  --downloadonly        仅下载包，不安装或升级。默认下载在yum的缓存目录
                        /var/cache/yum/$basearch/$releasever
  --downloaddir=DLDIR   指定下载目录
```

很多时候，yum操作失败的原因是repo的配置错误，或者缓存未更新。

### yum操作包组

```
[root@server2 ~]# yum help groups
Loaded plugins: fastestmirror, langpacks
groups [list|info|summary|install|upgrade|remove|mark] [GROUP]
 
Display, or use, the groups information
 
aliases: group, grouplist, groupinfo, groupinstall, groupupdate, groupremove, grouperase
```

所以，可以使用别名group, grouplist, groupinfo, groupinstall, groupupdate, groupremove, grouperase替代相应的命令。

如安装包组`yum groups install pkg_grpname`，也可以`yum groupinstall pkg_grpname`。

## CentOS 8的dnf包管理工具

我专门写了一篇文章介绍CentOS 8上包管理相关的内容，参考：[CentOS 8的国内镜像和dnf包管理器](https://www.junmajinlong.com/linux/centos8_dnf)。

## 补丁工具diff和patch

补丁文件都是通过diff命令生成的，patch工具则是应用补丁，即将补丁打到旧文件中。绝对不能跨版本打补丁，只能一个版本一个版本按顺序慢慢打。

生成补丁：

```
diff -[r]Nu old_file new_file > patch_file
```

该命令将对`old_file`和`new_file`做比较，然后生成补丁，将补丁内容保存到`patch_file`中。其中`-r`选项是递归比较，也就是说可以对整个目录生成补丁。

应用补丁：

```
patch -p数字 [file_name] < patch_file
```

去除补丁：

```
patch -R -p数字 [file_name] < patch_file
```

对于-p选项后的数字，它表示从前向后要忽略的目录层数，其实没必要想的那么复杂。首先，进入到待打补丁所在目录，如果是对改目录下单个文件打补丁，则使用`-p0`，对目录打补丁，则使用`-p1`。

```
# 对文件而言
diff –uN from-file to-file > to-file.patch 
patch –p0 < to-file.patch 
patch –R –p0 < to-file.patch 

# 对目录而言
diff –uNr from-dir to-dir > to-dir.patch 
cd from-dir
patch –p1 < to-dir.patch  
patch –R –p1 < to-dir.patch 
```

## 源码编译安装程序

### 码编译的几个阶段

拿到源码的压缩包后，首先就是解压，再进入解压目录，之后就是源码编译的一般步骤。这些一般步骤并非适用所有程序的编译，但可举一反三。

1.阅读解压目录中的INSTALL/README文件。如果不是对着官方手册或文档，那么在安装前务必读一读INSTALL文件或README文件，只需读其中如何安装的部分即可。

2.解压后的目录里一般还有configure文件（也可能是config文件）。执行`./configure`或带有编译选项的`./configure`，检查系统环境是否符合满足安装要求，并将定义好的安装配置写入和系统环境信息写入Makefile文件中。里面包含了如何编译、启用哪些功能、安装路径等信息。

3.执行`make`命令进行编译。make命令会根据Makefile文件进行编译。编译工作主要是调用编译器(如gcc)将源码编译为可执行文件，通常需要一些函数库才能产生一个完整的可执行文件。

4.`make install`将上一步所编译的数据复制到指定的目录下。

这就已经完成编译程序的过程了。

### configure脚本的通用功能

configure一般都会接受以下几个编译选项：

```
--prefix=          ：指定安装的路径
--sysconfdir=      ：指定配置文件目录
--enable-feature   ：启用某个特性
--disable-fecture  ：禁用特性
--with-function    ：启用某功能
--without-function ：禁用某功能 
```

不同的程序，其configure选项不尽相同，应使用`./configure --help`获取具体的信息。

### 源码编译安装须知

1.上面的每一个步骤都不能出错，否则后一步都不能正常进行。

2.上面的步骤每一步如果出现警告或错误，如果步骤未停止而是继续，则属于可忽略错误或警告，不影响安装。但是进行的步骤停止了出现警告或错误，则根据步骤考虑对策。可以使用`$?`命令查看上一个命令是否正确执行，如果是返回0则是正确，其他的则是错误。

3.卸载时，只需删除安装目录即可。如果安装的路径比较分散，不方便直接删除，那么也可回到源码目录(如果删除了需要重新下载并解压)，执行`make uninstall`。

此外，如果是debian/ubuntu，可以先安装一个checkinstall工具包，以后编译安装时，在`make install`之前执行一下`checkinstall`，就可以让apt跟踪编译安装的包，卸载时直接使用apt卸载即可。

4.通过源码编译的软件，需要做一些后续操作，虽非必须，但是都是个性化定制，方便以后的操作。个性化定制大致包括以下几项：

(1).将安装路径下的命令路径加入到环境变量。

```
echo "export PATH=/usr/local/apache/bin:$PATH" > /etc/profile.d/apache.sh
chmod +x /etc/profile.d/apache.sh
source /etc/profile.d/apache.sh
```

(2).按需求定制服务启动脚本，并考虑是否加入开机启动项。

(3).输出头文件和库文件。

头文件库文件很多时候只是为其他程序提供的，所以可能不输出它们的路径也不会影响该程序的运行。

```
# 输出头文件
ln -s /usr/local/apache/include /usr/include/apache

# 输出库文件
echo "/usr/local/apache/lib" >/etc/ld.so.conf.d/apache.conf
ldconfig
```

(4).导出man路径

```
echo  "MANPATH /usr/local/apache/man" >> /etc/man.conf
```

