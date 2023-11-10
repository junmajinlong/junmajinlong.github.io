---
title: yum安装php时遇到的坑
p: linux/yum_install_php_errors.md
date: 2020-07-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# yum安装php Requires: libzip5(x86-64) >= 1.3.2

在做实验时，使用rpm包安装php时，系统自带的版本总是很旧。想安装新版本的php rpm包时，又发现各种依赖包版本达不到要求。

```vhdl
Error: Package: php-pecl-zip-1.15.2-1.el6.remi.5.6.x86_64 (remi)
           Requires: libzip5(x86-64) >= 1.3.2
Error: Package: php-pecl-zip-1.15.2-1.el6.remi.5.6.x86_64 (remi)
           Requires: libzip.so.5()(64bit)
```

所以，只能从remi源来获取php，但是只配置remi还不够，因为它只有php各个版本相关的包，其他依赖包(如libzip5)和相关工具(如php-fpm)都放在remi safe源中的，因此需要同时配置remi源和remi safe源，remi safe中存放的是安装php过程中可能需要的包以及其他相关工具，例如依赖包libzip5、php容器php-fpm。

```ini
[root@xuexi ~]# cat /etc/yum.repos.d/remi.repo
[remi]
name=remirepo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/6/php56/x86_64/
enable=1
gpgcheck=0

[remisafe]
name=remisaferepo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/6/safe/x86_64/
enable=1
gpgcheck=0
```

再yum安装就一路顺利了。

```bash
yum --disablerepo=* --enablerepo=remi* -y install php php-fpm
```