---
title: CentOS 8的国内镜像和dnf包管理器
p: linux/centos8_dnf.md
date: 2020-09-07 13:20:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------


# CentOS 8的国内镜像和dnf包管理器

dnf(不是毒奶粉与勇士)一直以来是Fedora的包管理工具(从Fedora 22开始成为默认包管理器)，号称是yum的下一代包管理器。

在CentOS 8中也加入了dnf且成为默认的包管理工具，因此CentOS 8中可以使用dnf或yum来管理包。实际上，CentOS 8的yum是dnf的一个软链接：

```bash
$ ls -l /usr/bin/yum
lrwxrwxrwx. 1 root root 5 May 14  2019 /usr/bin/yum -> dnf-3
```

## 配置CentOS 8 dnf/yum国内镜像

在CentOS 7及之前的版本，yum官方仓库一直都是Base库、Extras库、Updates库、centosplus库等。其中Base库是安装CentOS时必须提供的仓库，它提供CentOS安装(比如可以选择安装桌面环境、开发工具等)、运行以及一些常用用户空间程序。

```bash
$ yum repolist all
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * epel: mirrors.bfsu.edu.cn
 * extras: mirrors.163.com
 * updates: mirrors.163.com
repo id                  repo name                                       status
!base/7/x86_64           CentOS-7 - Base                                 enabled: 10,070
base-debuginfo/x86_64    CentOS-7 - Debuginfo                            disabled
base-source/7            CentOS-7 - Base Sources                         disabled
centos-kernel/7/x86_64   CentOS LTS Kernels for x86_64                   disabled
centosplus/7/x86_64      CentOS-7 - Plus                                 disabled
centosplus-source/7      CentOS-7 - Plus Sources                         disabled
!epel/x86_64             Extra Packages for Enterprise Linux 7 - x86_64  enabled: 13,455
!extras/7/x86_64         CentOS-7 - Extras                               enabled:    413
extras-source/7          CentOS-7 - Extras Sources                       disabled
!updates/7/x86_64        CentOS-7 - Updates                              enabled:  1,134
updates-source/7         CentOS-7 - Updates Sources                      disabled
repolist: 25,072
```

在CentOS 8中发生了变化，原来的base库被拆分成两部分：**AppStream**和Base库。安装CentOS 8时必须提供这两个库。

其中：  

- CentOS 8的Base库提供安装和运行CentOS 8时必须的包，即CentOS核心包。这个仓库中全都是rpm包。  

- CentOS 8的AppStream库提供常用用户空间程序，它们并不一定是安装和运行CentOS 8所必须的，比如Python包、Perl包等语言包都在AppStream。AppStream中包含rpm包和dnf的模块。

AppStream库中的包一般是用户空间程序包，这些程序的更新速度一般比CentOS系统更新快的多，将它们单独提取到AppStream库，意味着这些程序包和系统相关的包被解绑分开到两个仓库，这可以让系统包和常用程序包分开升级，有利于提供这些程序包的最新的版本。

使用过CentOS的人可能都会庆幸这种改变。最近这些年互联网的发展极为迅速，很多程序包的迭代速度也非常快，以前的CentOS版本中，很多程序的版本往往都非常古老，要升级这些程序包，只能单独配置它们的镜像仓库(相当于是第三方仓库)，甚至很多程序只能自己编译新版本。

在配置CentOS 8仓库时，还有一个**PowerTools**仓库很可能会用上，它主要用于提供：

- 其他程序包的依赖包(大量的devel包和lib包)  
- 那些语言的模块或包，比如Perl-DataTime。这意味着，如果不用语言自身的包管理器(比如pip/gem/cpan/npm等)安装它们的模块，就需要启用PowerTools仓库后用yum/dnf来安装  

配置CentOS 8的国内镜像仓库和之前CentOS版本倒没什么区别，只是要注意提供AppStream和可能要使用到的PowerTools仓库。

下面是CentOS 8中国内镜像仓库的配置示例：

```ini
# $tencent=https://mirrors.cloud.tencent.com
# $contentdir=centos
# $releasever=8
[AppStream]
name=CentOS-$releasever - AppStream
baseurl=$tencent/$contentdir/$releasever/AppStream/$basearch/os/
gpgcheck=0
enabled=1

[BaseOS]
name=CentOS-$releasever - Base
baseurl=$tencent/$contentdir/$releasever/BaseOS/$basearch/os/
gpgcheck=0
enabled=1

[centosplus]
name=CentOS-$releasever - Plus
baseurl=$tencent/$contentdir/$releasever/centosplus/$basearch/os/
gpgcheck=0
enabled=1

[extras]
name=CentOS-$releasever - Extras
baseurl=$tencent/$contentdir/$releasever/extras/$basearch/os/
gpgcheck=0
enabled=1

[PowerTools]
name=CentOS-$releasever - PowerTools
baseurl=$tencent/$contentdir/$releasever/PowerTools/$basearch/os/
enabled=1
gpgcheck=0

[epel]
name = epel repo
baseurl = $tencent/epel/$releasever/Everything/$basearch
enabled=1
gpgcheck=0
```

配置国内镜像之后，重建缓存：

```bash
sudo dnf clean all
sudo dnf makecache
```

## dnf用法

参考手册：<https://dnf.readthedocs.io/en/latest/command_ref.html>。

dnf和yum的常用用法基本一致，不过dnf支持更多的功能。dnf最典型的功能是支持模块。

```
$ dnf --help         
usage: dnf [options] COMMAND

List of Main Commands:

alias                     List or create command aliases
autoremove                remove all unneeded packages that were originally installed as dependencies
check                     check for problems in the packagedb
check-update              check for available package upgrades
clean                     remove cached data
deplist                   List package's dependencies and what packages provide them
distro-sync               synchronize installed packages to the latest available versions
downgrade                 Downgrade a package
group                     display, or use, the groups information
help                      display a helpful usage message
history                   display, or use, the transaction history
info                      display details about a package or group of packages
install                   install a package or packages on your system
list                      list a package or groups of packages
makecache                 generate the metadata cache
mark                      mark or unmark installed packages as installed by user.
module                    Interact with Modules.
provides                  find what package provides the given value
reinstall                 reinstall a package
remove                    remove a package or packages from your system
repolist                  display the configured software repositories
repoquery                 search for packages matching keyword
repository-packages       run commands on top of all packages in given repository
search                    search package details for the given string
shell                     run an interactive DNF shell
swap                      run an interactive dnf mod for remove and install one spec
updateinfo                display advisories about packages
upgrade                   upgrade a package or packages on your system
upgrade-minimal           upgrade, but only 'newest' package match which fixes a problem that affects your system

List of Plugin Commands:

builddep                  Install build dependencies for package or spec file
changelog                 Show changelog data of packages
config-manager            manage dnf configuration options and repositories
copr                      Interact with Copr repositories.
debug-dump                dump information about installed rpm packages to file
debug-restore             restore packages recorded in debug-dump file
debuginfo-install         install debuginfo packages
download                  Download package to current directory
needs-restarting          determine updated binaries that need restarting
playground                Interact with Playground repository.
repoclosure               Display a list of unresolved dependencies for repositories
repodiff                  List differences between two sets of repositories
repograph                 Output a full package dependency graph in dot format
repomanage                Manage a directory of rpm packages
reposync                  download all packages from remote repo

Optional arguments:
  -c [config file], --config [config file]
                        config file location
  -q, --quiet           quiet operation
  -v, --verbose         verbose operation
  --version             show DNF version and exit
  --installroot [path]  set install root
  --nodocs              do not install documentations
  --noplugins           disable all plugins
  --enableplugin [plugin]
                        enable plugins by name
  --disableplugin [plugin]
                        disable plugins by name
  --releasever RELEASEVER
                        override the value of $releasever in config and repo
                        files
  --setopt SETOPTS      set arbitrary config and repo options
  --skip-broken         resolve depsolve problems by skipping packages
  -h, --help, --help-cmd
                        show command help
  --allowerasing        allow erasing of installed packages to resolve
                        dependencies
  -b, --best            try the best available package versions in
                        transactions.
  --nobest              do not limit the transaction to the best candidate
  -C, --cacheonly       run entirely from system cache, don't update cache
  -R [minutes], --randomwait [minutes]
                        maximum command wait time
  -d [debug level], --debuglevel [debug level]
                        debugging output level
  --debugsolver         dumps detailed solving results into files
  --showduplicates      show duplicates, in repos, in list/search commands
  -e ERRORLEVEL, --errorlevel ERRORLEVEL
                        error output level
  --obsoletes           enables dnf's obsoletes processing logic for upgrade
                        or display capabilities that the package obsoletes for
                        info, list and repoquery
  --rpmverbosity [debug level name]
                        debugging output level for rpm
  -y, --assumeyes       automatically answer yes for all questions
  --assumeno            automatically answer no for all questions
  --enablerepo [repo]
  --disablerepo [repo]
  --repo [repo], --repoid [repo]
                        enable just specific repositories by an id or a glob,
                        can be specified multiple times
  --enable, --set-enabled
                        enable repos with config-manager command
                        (automatically saves)
  --disable, --set-disabled
                        disable repos with config-manager command
                        (automatically saves)
  -x [package], --exclude [package], --excludepkgs [package]
                        exclude packages by name or glob
  --disableexcludes [repo], --disableexcludepkgs [repo]
                        disable excludepkgs
  --repofrompath [repo,path]
                        label and path to additional repository, can be
                        specified multiple times.
  --noautoremove        disable removal of dependencies that are no longer
                        used
  --nogpgcheck          disable gpg signature checking (if RPM policy allows)
  --color COLOR         control whether color is used
  --refresh             set metadata as expired before running the command
  -4                    resolve to IPv4 addresses only
  -6                    resolve to IPv6 addresses only
  --destdir DESTDIR, --downloaddir DESTDIR
                        set directory to copy packages to
  --downloadonly        only download packages
  --comment COMMENT     add a comment to transaction
  --bugfix              Include bugfix relevant packages, in updates
  --enhancement         Include enhancement relevant packages, in updates
  --newpackage          Include newpackage relevant packages, in updates
  --security            Include security relevant packages, in updates
  --advisory ADVISORY, --advisories ADVISORY
                        Include packages needed to fix the given advisory, in
                        updates
  --bzs BUGZILLA        Include packages needed to fix the given BZ, in
                        updates
  --cves CVES           Include packages needed to fix the given CVE, in
                        updates
  --sec-severity {Critical,Important,Moderate,Low}, --secseverity {Critical,Important,Moderate,Low}
                        Include security relevant packages matching the
                        severity, in updates
  --forcearch ARCH      Force the use of an architecture
```

## dnf基本用法

这里只介绍和yum差不多的常用的功能，dnf模块相关的后文专门介绍。

- **清理缓存和重建缓存**  

  ```bash
  sudo dnf clean [ all ]
  sudo dnf makecache   # 注：不兼容以前yum的用法yum makecache fast
  ```

- **列出已配置的镜像仓库**  

  ```bash
  # 列出已启用(enabled=1)、已禁用或所有已配置仓库
  dnf repolist
  dnf repolist --enabled
  dnf repolist --disabled 
  dnf repolist --all
  
  # 查看某个或所有仓库的详细信息
  dnf repolist -v
  dnf repolist BaseOS -v
  ```

- **列出包**  

  ```bash
  # 列出所有可获取的包(包括已安装和未安装)，下面等价
  dnf list          # 默认--all选项
  dnf list --all
  
  # 分别列出已安装和未安装的包
  dnf list --installed
  dnf list --available
  
  # 列出可升级的包，等价
  dnf list --updates
  dnf list --upgrades
  dnf check-updates
  
  # 列出可被移除(已不再被依赖)的包
  dnf list --autoremove
  ```

- **搜索包**  

  ```bash
  # 从列出的包中搜索某个包，使用grep即可
  dnf list --installed | grep 'net-tools'
  
  # 根据pattern匹配包名，它会从包名、包的描述信息中匹配
  dnf search 'net*tools'
  
  # 只从包名匹配
  dnf repoquery 'net*tools'
  
  # 根据文件搜索包
  dnf provides '*bin/netstat'
  ```

- **查看包的信息**

  ```
  dnf info net-tools
  ```

- **查看包中包含的文件**

  ```bash
  # 属于rpm的用法
  rpm -ql net-tools
  ```

- **安装和卸载包**

  ```bash
  # 下面命令加上-y选项，可以在交互式中自动yes确认
  
  # 根据包名安装
  dnf install net-tools
  
  # 根据文件名安装(dnf有而以前yum没有但好用的功能)
  dnf install /usr/bin/netstat
  dnf install $(which netstat)
  
  # 移除包，同时会移除依赖包
  dnf remove net-tools
  
  # 自动移除已经不被依赖的包
  dnf autoremove
  ```

- **升级系统中的包**

  ```bash
  # 列出可升级的包，其中check-updates还列出如果升级时必须升级的包
  # 其他包都不是必须升级的
  dnf list --updates
  dnf list --upgrades
  dnf check-updates
  
  # 升级一个包或所有可升级的包
  yum upgrade
  yum upgrade <pkg_name>
  
  # 最小化升级，只升级check-updates中最后列出的obsolete类型的包
  yum upgrade-minimal
  ```

## dnf模块

![](/img/linux/1603803096826.png)

也就是说，**dnf模块将某些具有关联关系的包和它们的版本号绑定在一起**。或者，可以将模块看作是带有版本号、可共存、可配置的包组。

dnf模块都由stream提供，例如其中一个官方的stream就是AppStream。

例如，AppStream提供了两个版本的perl dnf模块：

```bash
$ dnf module list perl
AppStream
Name    Stream    Profiles              Summary                                        
perl    5.24      common [d], minimal   Practical Extraction and Report Language       
perl    5.26 [d]  common [d], minimal   Practical Extraction and Report Language       

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```

perl 5.24版本的dnf模块包含的是所有perl 5.24相关的包，perl 5.26包含的是所有perl 5.26相关的包。

关注一下上面输出的两行perl的内容。

它们的模块名(Name)叫做perl，它们来自AppStream，且perl有两个版本的Stream，分别是5.24和5.26，其中5.26后面有个`[d]`表示这是默认的perl模块，只有标记为`[d]`或`[e]`的模块才可以已经启用(激活)的模块，可以安装，否则不可安装。另外，这两个perl模块都有两个Profiles，其中`common`是默认的profiles，还有一个minimal，Profiles的含义不言自明：配置了哪些rpm包需要被安装。所以，minimal是最小化安装的方式，而common在安装perl模块的时候还会安装一些额外的包。

### dnf module用法

参考资料：

- <https://dnf.readthedocs.io/en/latest/conf_ref.html>  
- <https://docs.fedoraproject.org/en-US/modularity/>  

下面截取自`dnf help module`

```bash
$ dnf help module
usage: dnf module <modular command> [module-spec [module-spec ...]]
...
# 下面是module可用的选项
Module command-specific options:
  --enabled             show only enabled modules
  --disabled            show only disabled modules
  --installed           show only installed modules or packages
  --profile             show profile content
  --available           show only available packages
  --all                 remove all modular packages

# 下面是module可用的子命令
  <modular command>     disable: disable a module with all its streams
                        enable: enable a module stream
                        info: print detailed information about a module
                        install: install a module profile including its packages
                        list: list all module streams, profiles and states
                        provides: list modular packages
                        remove: remove installed module profiles and their packages
                        repoquery: list packages belonging to a module
                        reset: reset a module
                        update: update packages associated with an active stream
  module-spec           Module specification
```

列出模块：

```bash
# 列出当前系统所有已配置的模块
dnf module list
# 列出已启用的模块(可直接安装)
dnf module list --enabled

# 列出已被禁用的模块(不可直接安装，需启用后才可安装)
dnf module list --disabled
```

查看模块信息：

```bash
dnf module info perl
```

安装、移除模块：

```bash
# 安装默认版本(stream)、默认profile
dnf module install perl

# 指定版本
dnf module install perl:5.26

# 指定Profile
dnf module install perl/minimal
dnf module install perl:5.26/minimal

# 移除模块
dnf module remove perl
dnf module remove perl:5.24
dnf module remove perl/minimal
dnf module remove perl:5.24/minimal
```

安装模块的另一种写法：

```bash
dnf install @perl
dnf install @perl:5.26
dnf install @perl:5.26/common
```

启用、禁用模块：

```bash
dnf module disable perl:5.24
dnf module enable perl:5.24
```

perl版本切换(perl 5.24 -> perl 5.26)：

```bash
# 即切换模块，分两步操作

# 重置enabled状态，这不会卸载已安装的perl 5.24模块中的包
dnf module reset perl

# 安装perl:5.26
dnf module install perl:5.26
```

