---
title: 熟练使用vagrant(8)：修改vagrant虚拟机的几个name
p: virtual/vagrant/vagrant_about_names.md
date: 2020-10-01 17:37:37
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(8)：修改vagrant虚拟机的几个name

在使用vagrant的过程中，有几个名称让人不太舒适。

![](/img/virtual/2020_10_28_1603865182682.png)

想要使用vagrant管理好虚拟机，修改它们是必要的：  
- 一个是`vagrant status`或`vagrant global-status`中显示的name，它们默认都是default，可通过`config.vm.define`修改  
- 一个是virtualbox(或hyperv)中显示的虚拟机的名称，对于virtualbox，可使用name属性来修改，对于hyperv，可使用vmname属性来修改  
- 还有一个是启动的虚拟机中操作系统的主机名，可通过config.vm.hostname修改  

例如，如下Vagrantfile配置：
```
Vagrant.configure('2') do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.box_version = "3.0.34"
  config.vm.define "ubuntu2004_junmajinlong"
  config.vm.hostname = "longshuai"
  config.vm.provider :virtualbox do |vb|
    vb.name = "ubuntu2004"
  end
end
```

使用上面的配置去安装虚拟机时：  
- vagrant为该虚拟机设置的name标识符为ubuntu2004_junmajinlong  
- 虚拟机管理工具(virtualbox)上虚拟机的名称为ubuntu2004  
- 虚拟机内ubuntu系统的主机名为longshuai  

```
$ vagrant global-status
id       name                    provider   state   directory
----------------------------------------------------------------------------------------
3cc30cd  ubuntu2004_junmajinlong virtualbox running V:/vagrant_test/test

$ vagrant ssh
vagrant@longshuai:~$ hostname
longshuai
```

![](/img/virtual/2020_10_14_1602609713763.png)
