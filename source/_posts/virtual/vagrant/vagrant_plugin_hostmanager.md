---
title: 熟练使用vagrant(15)：使用hostmanager插件自动管理DNS解析
p: virtual/vagrant/vagrant_plugin_hostmanager.md
date: 2020-10-01 17:37:44
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(15)：使用hostmanager插件自动管理DNS解析

使用vagrant批量创建虚拟机纵然方便，但在做类似集群的实验时，经常需要去配置集群中主机的主机名解析(/etc/hosts)。

在各虚拟机上手动配置/etc/hosts会比较麻烦，好在有一个vagrant的插件hostmanager，它可以自动将vagrant创建的虚拟机互相添加到各自的/etc/hosts文件中，也允许用户在宿主机上添加各虚拟机的主机名解析信息。

安装hostmanager插件：

```powershell
$ vagrant plugin install vagrant-hostmanager
```

使用hostmanager的示例：
```ruby
Vagrant.configure("2") do |config|

  config.vm.box = "generic/centos7"
  config.vm.provider "hyperv"

  # 激活hostmanager插件
  config.hostmanager.enabled = true

  # 在宿主机上的hosts文件中添加虚拟机的主机名解析信息
  config.hostmanager.manage_host = true

  # 在各自虚拟机中添加各虚拟机的主机名解析信息
  config.hostmanager.manage_guest = true

  # 不忽略私有网络的地址
  config.hostmanager.ignore_private_ip = false

  config.vm.define 'vm1' do |node|
    node.vm.hostname = 'vm1'
    node.vm.network :private_network, ip: '192.168.42.42'
  end

  config.vm.define 'vm2' do |node|
    node.vm.hostname = 'vm2'
  end
end
```

对于virtualbox的虚拟机来说，强烈建议结合private_network或public_network使用，否则会因虚拟机默认使用的NAT模式的端口映射，而导致添加的主机名解析信息都是127.0.0.1。

执行`vagrant up`后，在宿主机、vm1以及vm2上的hosts文件中添加如下内容：
```
## vagrant-hostmanager-start
192.168.42.42  vm1

172.25.190.89  vm2

## vagrant-hostmanager-end
```

添加后，就可以通过主机名和对方通信，例如无论是在宿主机上还是在vm2上，都可以直接`ping vm1`。

如果想要更新已启动的虚拟机的hosts文件，执行如下命令即可：
```
$ vagrant hostmanager
```

