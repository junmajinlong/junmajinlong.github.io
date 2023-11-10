---
title: 熟练使用vagrant(12)：vagrant目录同步synced_folder
p: virtual/vagrant/vagrant_synced_folder.md
date: 2020-10-01 17:37:41
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(12)：vagrant目录同步synced_folder

vagrant提供了目录同步的功能，它可以将宿主机上的目录自动同步到虚拟机中。

vagrant加上它的目录同步功能，对于搭建相同的开发环境、测试环境非常有帮助：所有人都可以用同一个Vagrantfile创建出相同环境的虚拟机并运行相同的程序代码。

目录同步的方式非常简单：
```ruby
# 将宿主机项目目录挂载到虚拟机的/vagrant目录
config.vm.synced_folder ".", "/vagrant"

# 将宿主机项目目录内的src子目录挂载到虚拟机的/src/website目录
config.vm.synced_folder "src/", "/srv/website"

# 禁用挂载项：不要在vagrant up或reload时自动挂载
config.vm.synced_folder "src/", "/srv/website", disabled: true

# 修改虚拟机上挂载目录的owner和group
# 不指定owner/group时，它们的owner默认是vagrant ssh所使用的用户，
# 即一般情况下是vagrant用户
config.vm.synced_folder "src/", "/srv/website",
  owner: "root", group: "root"

# 指定同步的方式，支持的方式有：nfs、rsync、smb、virtualbox
# 同步方式和provider有关，有些同步方式在某种provider上无效
config.vm.synced_folder ".", "/vagrant", type: "nfs"
```

对于不同provider和不同宿主机平台、虚拟机平台，vagrant支持的同步方式不一样：  
- 对于virtualbox，默认使用virtualbox方式，即virtualbox自带的目录同步功能(shared folder)，这种同步方式性能很差，但通用性更好  
- 如果使用nfs同步方式，首先Windows宿主机类型不支持nfs方式，另外要求宿主机安装了nfs工具，虚拟机安装了nfsd，且如果provider是VirtualBox，还要求至少配置一个private_network  
- 如果使用rsync同步方式，它默认虚拟机启动时才同步一次，可以执行`vagrant rsync`临时进行一次同步，或者执行`vagrant rsync-auto`使得目录出现变化时自动同步。要求宿主机中已经存在rsync命令  
- 如果使用smb同步方式，smb同步方式的性能相比Virtualbox内置的virtualbox shared folder同步方式要更高。smb同步方式要求宿主机只能是Windows或MacOS，且如果是Windows，则要求安装PowerShell 3.0+  

文件同步功能要求虚拟机安装VBoxGuestAdditions，并且它的版本要和VirtualBox版本匹配，如果因为缺失VBoxGuestAdditions或版本错误导致文件同步失败，可从<https://download.virtualbox.org/virtualbox/>查找和VirtualBox相同版本的手动安装。

可安装插件`vagrant-vbguest`，它可以在每次安装虚拟机时(vagrant up)默认安装VBoxGuestAdditions。

```ruby
Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
    config.vbguest.no_remote = true
  end
end
```