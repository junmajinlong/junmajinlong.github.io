---
title: 熟练使用vagrant(7)：使用vagrant box做虚拟机模板
p: virtual/vagrant/vagrant_box.md
date: 2020-10-01 17:37:36
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(7)：使用vagrant box做虚拟机模板

box是一个开箱即用的虚拟机，可直接由vagrant启动运行。

每一个box都是由他人打包好的虚拟机，只不过它是特殊格式的文件，且后缀名一般为`.box`。我们也可以使用`vagrant package`打包自己的虚拟机并分发给别人使用。

安装一个box，相当于提供了一个base image，即虚拟机模板。之后就可以基于这个模板去创建新的虚拟机并启动，vagrant将自动从box导入虚拟机所需数据。
```powershell
# 自动从vagrant官方的仓库中搜索centos/7，
# 这种添加方式，在国内速度可能会非常慢
vagrant box add centos/7
vagrant box add centos/7 --provider virtualbox
vagrant box add centos/7 --provider hyperv

# URL方式，可自定义添加后的box名称
vagrant box add centos-7 http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2004_01.VirtualBox.box

# 添加本地Box文件，可自定义添加后的Box名称
vagrant box add centos_7 V:\vagrant_imgs\CentOS-7-x86_64-Vagrant-2004_01.VirtualBox.box
```

添加box后，已经添加的box名称就可以直接作为虚拟机模板。比如：
```powershell
vagrant init centos-7
vagrant up
```

另外，对于vagrant官方仓库的所有box都可以直接使用box名称。例如：
```powershell
vagrant init generic/centos7
vagrant init generic/ubuntu2004
vagrant init ubuntu/bionic64
```

当没有指定box镜像路径或URL，就像上面示例直接在init上使用类似`xxx/yyy`作为box时，此时执行`vagrant up`，vagrant将首先从本地box存放目录(`$HOME/.vagrant.d/`或`$VAGRANT_HOME/.vagrant.d/`)下寻找名为`xxx/yyy`的box镜像，如果找不到，则自动从官方参考上下载对应的box镜像然后初始化启动，这种方式下载box时可能会很慢。

vagrant支持的管理box的子命令：
```
$ vagrant box -h
Usage: vagrant box <subcommand> [<args>]
subcommands:
  add
  list
  outdated
  prune
  remove
  repackage
  update
```

子命令    | 功能说明
---------|-----
add      | 安装box
list     | 列出已安装的box。已安装的box存放在`$HOME/.vagrant.d/`或`$VAGRANT_HOME/.vagrant.d/` 
outdated | 检查是否有新版本的box
prune    | 删除已有新版本的box
remove   | 删除指定的box
repackage| 重新打包指定的box
update   | 更新box到最新版(不会删除并新建box，而是直接在当前box上更新)
