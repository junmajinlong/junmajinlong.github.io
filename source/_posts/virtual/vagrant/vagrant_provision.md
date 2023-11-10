---
title: 熟练使用vagrant(9)：vagrant provision初始化虚拟机时执行
p: virtual/vagrant/vagrant_provision.md
date: 2020-10-01 17:37:38
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(9)：vagrant provision初始化虚拟机时执行

使用别人打包好的box所安装的虚拟机，其中的配置都是默认的或对方配置的，这些配置可能并不适合每个人。

比如我们想要在安装虚拟机之后就自动安装好docker、配置好国内镜像源、配置好网络、启动好服务，等等。我们当然可以在安装好虚拟机后，通过ssh连接到虚拟机中手动去做这些配置，但这样并不方便。

vagrant支持provision功能，它允许用户在Vagrantfile中指定第一次`vagrant up`创建并初始化虚拟机成功后要执行的操作，比如执行一个shell脚本或sed命令等。这些操作都是第一次初始化虚拟机时自动执行的，无需用户手动交互式地去执行，非常方便。

vagrant支持多种provision，包括shell、ansible、chef、puppet、saltstack、docker、podman以及file。

例如，对于最常用的shell provision，vagrant允许执行一个指定的脚本或执行指定的命令行。对于file provision，vagrant允许从宿主机上传一个文件或目录到虚拟机指定位置。对于ansible provision，vagrant允许执行ansible playbook。对于docker provision，vagrant允许自动安装docker、pull镜像、启动docker容器等。

vagrant还支持同时使用多个provision的功能，且允许用户定义它们的依赖关系。

除了默认在第一次执行`vagrant up`初始化虚拟机的过程中会执行provision指定的操作，以下情况也会执行：  
- `vagrant up --provision`  
- `vagrant reload --provision`  
- `vagrant provision`  
- config.vm.provision的参数run指定为always  

本文介绍最基本也最常用的[shell provision](https://www.vagrantup.com/docs/provisioning/shell)用法，其他provision以及provision更详细的用法可参考官方手册[vagrant provision](https://www.vagrantup.com/docs/provisioning)。

例如，对于CentOS7，开机时为其配置国内镜像源，配置epel源，安装常用包：
```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos7"
  config.vm.define "generic_centos7"
  config.vm.hostname = "longshuai-vm"
  
  config.vm.provider :virtualbox do |vb|
    vb.name = "generic_centos7"
  end
  
  config.vm.provider :hyperv do |h|
    h.vmname = "generic_centos7"
  end

  # 单行命令，且每次vagrant up/reload都执行
  config.vm.provision "shell", run: "always", inline: "echo hello world"

  # 多行命令，且只有第一次vagrant up初始化启动vm时执行
  config.vm.provision "shell", inline: <<-SHELL
    sudo bash
    mkdir /etc/yum.repos.d/bak
    mv /etc/yum.repos.d/CentOS*.repo /etc/yum.repos.d/bak
    # yum base repo
    cat <<'EOF' >>/etc/yum.repos.d/base.repo
[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.ustc.edu.cn/centos/$releasever/os/$basearch/
gpgcheck=0

[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.ustc.edu.cn/centos/$releasever/updates/$basearch/
gpgcheck=0

[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.ustc.edu.cn/centos/$releasever/extras/$basearch/
gpgcheck=0
EOF
    # epel repo
    yum install -y epel-release
    sed -e 's|^metalink=|#metalink=|g' \
          -e 's|^#baseurl=https\?://download.fedoraproject.org/pub/epel/|baseurl=https://mirrors.ustc.edu.cn/epel/|g' \
          -i.bak \
          /etc/yum.repos.d/epel.repo
    # install packages
    yum makecache
    yum install -y curl wget make
  SHELL
end
```

上面定义了一个安装`generic/centos7`的Vagrantfile，这个Vagrantfile文件即可用于在virtualbox上安装generic/centos7，也可以在hyperv上安装，但只能二选一，vagrant不支持在同一个vagrant环境(即Vagrantfile所在目录)下在不同的provider上安装同一个box的虚拟机。

此外，上面示例中virtualbox的定义在hyperv之前，意味着`vagrant up`默认或优先安装在virtualbox，如果要安装在排在后面的hyperv上，需`vagrant up --provider hyperv`。

对于上面的示例而言，无论generic/centos7是安装在virtualbox上还是安装在hyperv上，都会执行指定的provision操作：输出hello world信息，然后配置国内yum源和epel源，并安装几个软件包。

如果vagrant provision要执行的是一个写好的外部shell脚本，则可采用如下格式：
```
# 本地shell脚本
config.vm.provision "shell", path: "script.sh"
# 或指定shell脚本的URL
config.vm.provision "shell", path: "https://example.com/provisioner.sh"
```

对于本地shell脚本，如果使用的是相对路径，例如上面的示例，表示相对Vagrantfile所在的目录。所以`path: "script.sh"`表示执行Vagrantfile所在目录内的script.sh。
