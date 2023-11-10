---
title: 熟练使用vagrant(16)：WSL中使用vagrant
p: virtual/vagrant/vagrant_in_wsl.md
date: 2020-10-01 17:37:45
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(16)：WSL中使用vagrant

可以在wsl中使用vagrant，且wsl中的vagrant可以和宿主机Win10上的virtualbox、hyper-v交互。

假如wsl是ubuntu，那么wsl中就可以直接`apt install vagrant`。但是要求wsl中安装的vagrant的版本和宿主机win10上安装的vagrant版本要相同。

因此，如果win10上安装的vagrant是2.2.10版，那么wsl ubuntu中也必须安装2.2.10版的vagrant。通过包管理工具(比如apt/yum)安装的vagrant可能会版本不够，需要在vagrant官网下载对应版本的包：[各版本vagrant下载地址](https://releases.hashicorp.com/vagrant/)。

如果要使用virtualbox作为Provider，建议不要使用wsl2，因为wsl2自身是完整的虚拟机，其IP地址和宿主机是隔离的，这会导致vagrant ssh访问不到vagrant创建的不带private_network、public_network的virtualbox的虚拟机。

如果非要使用wsl2，需确保自己有能力解决以上问题。另外可参考下面我给出的wsl2上Vagrantfile的配置。

在wsl 1中安装好vagrant后，还需要在wsl中设置下面两个环境变量：
```bash
# 允许wsl中的vagrant使用宿主机上的信息
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"

# 将宿主机win10上的VAGRANT_HOME设置为该环境变量的值
# 例如我的Win10中，VAGRANT_HOME的值为"V:\vagrant_imgs"
# 但我个人不建议将其设置为Win10中的VAGRANT_HOME，而是设置自己的目录
# (然后拷贝Win10 VAGRANT_HOME里的内容)，这样wsl和win10两者不会混乱
export VAGRANT_WSL_WINDOWS_ACCESS_USER_HOME_PATH="/mnt/v/vagrant_imgs"
```

为了以后方便使用，可见上面设置环境变量的操作写入`.bashrc`中。

另外注意，有些box默认已经配置了`synced_folder`，在wsl中是不能使用目录同步功能的，如果有相关报错，需要在Vagrantfile中加上一行：
```ruby
config.vm.synced_folder ".", "/vagrant", disabled: true
```

下面是我在wsl2上使用vagrant的Vagrantfile：
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.define "wsl_ubuntu2004"
  # 手动禁止synced_folder
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # 配置端口转发，将22端口转发到一个固定端口上
  config.vm.network "forwarded_port", guest: 22, host:23333

  # 172.22.192.1是宿主机win10上WSL对应的网络IP地址
  # 可在wsl2中使用如下命令获得：
  # ipconfig.exe | dos2unix | grep -A4 'WSL' | awk -F': ' '/IPv4/{print $2}'
  # 或Powershell中如下命令：
  # (Get-NetIPAddress -InterfaceAlias "vEthernet (WSL)" -AddressFamily IPv4).IPAddress
  config.ssh.host = "172.22.192.1"
  config.ssh.port = 23333
  config.vm.provider "virtualbox" do |vb|
    vb.name = "wsl_ubuntu2004"
  end
end
```

下面是wsl2上三个虚拟机的Vagrantfile：
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.ssh.host = "172.22.192.1"

  config.vm.define "wsl_ubuntu1" do |t|
    t.vm.network "forwarded_port", guest: 22, host:23333
    t.ssh.port = 23333
    t.vm.provider "virtualbox" do |vb|
      vb.name = "wsl_ubuntu1"
    end
  end

  config.vm.define "wsl_ubuntu2" do |t|
    t.vm.network "forwarded_port", guest: 22, host:23334
    t.ssh.port = 23334
    t.vm.provider "virtualbox" do |vb|
      vb.name = "wsl_ubuntu2"
    end
  end

  config.vm.define "wsl_ubuntu3" do |t|
    t.vm.network "forwarded_port", guest: 22, host:23335
    t.ssh.port = 23335
    t.vm.provider "virtualbox" do |vb|
      vb.name = "wsl_ubuntu3"
    end
  end
end
```

这时甚至可让`vagrant up`并行创建这三个虚拟机：
```bash
$ vagrant status --machine-readable | grep -oP '\w+(?=,meta)'
wsl_ubuntu1
wsl_ubuntu2
wsl_ubuntu3

$ vagrant status --machine-readable | grep -oP '\w+(?=,meta)' | xargs -P 3 -i vagrant up {}
```

