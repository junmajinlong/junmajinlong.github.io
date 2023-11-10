---
title: 为CentOS配置常用软件源：epel和IUS
p: linux/centos_repos.md
date: 2020-07-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 为CentOS配置常用软件源：epel和IUS

## EPEL

```
Extra Packages for Enterprise Linux (or EPEL) is a Fedora Special Interest Group that creates, maintains, and manages a high quality set of additional packages for Enterprise Linux, including, but not limited to, Red Hat Enterprise Linux (RHEL), CentOS and Scientific Linux (SL), Oracle Linux (OL).
```

简言之，EPEL是专门为RHEL、CentOS等Linux发行版提供额外rpm包的。很多os中没有或比较旧的rpm，在epel仓库中可以找到。

例如配置阿里云的epel：

```
rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-6.noarch.rpm
rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
```

## IUS

在kernel.org内，清楚地说明了IUS项目是干什么的：

```
IUS is a community project that provides RPM packages for newer versions of select software for Enterprise Linux distributions.
 
Project Goals
   Create high quality RPM packages for Red Hat Enterprise Linux (RHEL) and CentOS.
   Promptly release updated RPM packages once new versions are released by the upstream developers.
   No automatic replacement of stock RPM packages.
```

IUS只为RHEL和CentOS这两个发行版提供较新版本的rpm包。如果在os或epel找不到某个软件的新版rpm，软件官方又只提供源代码包的时候，可以来ius源中找，几乎都能找到。例如haproxy，在CentOS 6的epel中只有1.5版本的，但ius中却提供了1.6和1.7版本。

IUS源的站点根目录：<https://dl.iuscommunity.org/pub/ius/>。

IUS提供4个分支的rpm包：stable、archive、development和testing。显然，我们应该选择stable分支的包。

![](/img/linux/733013-20180302202435489-1077304590.png)

配置IUS源：

```
rpm -ivh https://rhel5.iuscommunity.org/ius-release.rpm     # RHEL 5
rpm -ivh https://rhel6.iuscommunity.org/ius-release.rpm     # RHEL 6
rpm -ivh https://rhel7.iuscommunity.org/ius-release.rpm     # RHEL 7

rpm -ivh https://centos5.iuscommunity.org/ius-release.rpm  # CentOS 5
rpm -ivh https://centos6.iuscommunity.org/ius-release.rpm  # CentOS 6
rpm -ivh https://centos7.iuscommunity.org/ius-release.rpm  # CentOS 7
```

rpm安装ius-release.rpm时，依赖于epel。所以必须先安装epel源。注意，这是包的依赖关系，因此必须是安装了epel，而不是仅仅在repo文件中配置了epel源。

```
yum -y install epel-release
```

安装后，建议修改为国内ius源。在<https://mirrors.iuscommunity.org/mirrors>内可以查看到IUS项目的mirrorlist中所有的IUS站点。我看了下，中国地区只有两个站点：清华大学镜像站点和同济大学镜像站点。(阿里镜像mirrors.aliyun.com也在2018-03-28日上线了ius，同日还上线了remi)

```
https://mirrors.tuna.tsinghua.edu.cn/ius/stable/CentOS/6/$basearch  # CentOS 6
https://mirrors.tuna.tsinghua.edu.cn/ius/stable/Redhat/6/$basearch  # RHEL 6

https://mirrors.tongji.edu.cn/ius/stable/CentOS/6/$basearch         # CentOS 6
https://mirrors.tongji.edu.cn/ius/stable/Redhat/6/$basearch         # RHEL 6
```

或者，直接在repo文件中添加ius仓库，更方便，这样不依赖于epel。

```
[root@xuexi ~]# vim /etc/yum.repos.d/ius.repo
[ius]
name=iusrepo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/ius/stable/CentOS/6/$basearch
gpgcheck=0
enable=1
```

然后清除缓存再建立缓存即可。

```
yum clean all ; yum makecache 
```