---
title: 熟练使用vagrant(11)：vagrant配置虚拟机网络
p: virtual/vagrant/vagrant_network.md
date: 2020-10-01 17:37:40
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(11)：vagrant配置虚拟机网络

一般来说，如果没有更多的需求，vagrant对虚拟机网络所做的默认配置应该足够了。如果有进一步配置需求，除了要求了解virtualbox或hyperv或vmware的网络模型，还需了解vagrant对不同provider的默认网络配置行为。

- 对于virtualbox的网络模型，可参考：[理解VirtualBox网络](/virtual/network/virtualbox_net/)  
- 对于Hyper-V的网络模型，可参考：[理解Hyper-V外部网络、内部网络和私有网络](/virtual/network/hyperv_net)  
- 对于VMware的网络模型，可参考：[理解VMware网络模式：桥接、仅主机和NAT](/virtual/network/vmware_net)  

## vagrant配置端口转发

注：  
- 通常来说，无需使用vagrant的端口转发配置功能，使用场景并不多  
- vagrant无法配置hyper-v的端口转发  

配置端口转发的方式很简单：
```ruby
Vagrant.configure("2") do |config|
  # 虚拟机80端口 <=> 宿主机8080端口
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # 端口映射时，指定绑定的IP地址
  config.vm.network "forwarded_port", 
     guest: 111,  guest_ip: "192.168.1.3", 
     host: 10111, host_up: "192.168.1.5"

  # 指定协议
  config.vm.network "forwarded_port", guest: 222, host: 20222, protocol: "tcp"

  # 宿主机上端口冲突时，自动修正为其他可用端口
  config.vm.network "forwarded_port", guest: 333, host: 30333, auto_correct: true

  # 可指定宿主机上端口冲突时，自动修正时允许的端口范围
  config.vm.usable_port_range = 8000..8999
end
```

可以使用`vagrant port`命令查看端口映射情况：
```powershell
$ vagrant port test
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2222 (host)
```

## vagrant配置virtualbox虚拟机的网络

对于virtualbox，默认情况下，vagrant总会设置第一个网卡(eth0/ens0等)并将其加入virtualbox的NAT模式，以便虚拟机能够访问外网并提供端口转发功能。但使用virtualbox的NAT模式时，各虚拟机之间不能互相通信，且宿主机也无法访问NAT模式的虚拟机。

对于virtualbox，vagrant允许配置两种模型的网络：  
- private_network：私有网络，对应于virtualbox的host-only网络模型，这种模型下，虚拟机之间和宿主机(的虚拟网卡)之间可以互相通信，但不在该网络内的设备无法访问虚拟机  
- public_network：公有网络，对应于virtualbox的桥接模式，这种模式下，虚拟机的网络和宿主机的物理网卡是平等的，它们在同一个网络内，虚拟机可以访问外网，外界网络(特指能访问物理网卡的设备)也能访问虚拟机  

vagrant允许同时为某个虚拟机配置多个网络，每个网络都会在虚拟机中创建一个对应的网卡设备。

需要特别注意的是，无论是配置private_network还是public_network，第一个网卡(eth0/ens0等)一定是vagrant为virtualbox默认添加的加入到NAT模式的网卡。

## vagrant配置virtualbox的private_network

vagrant为virtualbox配置的private_network，其本质是将虚拟机加入到了virtualbox的host-only网络内。

例如，下面是一个Vatrantfile的配置。
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos7"

  config.vm.define "test1" do |t|
    t.vm.hostname = "test1"
    t.vm.network "private_network", ip: "192.168.50.11"
    t.vm.network "private_network", ip: "192.168.100.111"
    
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test1"
    end
  end
  
  config.vm.define "test2" do |t|
    t.vm.hostname = "test2"
    t.vm.network "private_network", ip: "192.168.100.22"
    t.vm.network "private_network", ip: "192.168.100.222"
    
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test2"
    end
  end
end
```

在上面的配置中，为test1虚拟机配置了两个private_network网络，它们对应的网络适配器分别是eth1和eth2，它们都加入virtualbox的host-only网络。此外test1还有一个默认加入NAT的网卡eth0。也就是说，test1虚拟机有三个网卡。同理test2也是三个网卡。

值得注意的是，vagrant为这些private_network网络配置的IP地址并不在同一个网段。**vagrant会自动为不同网段创建对应的host-only网络**。因此，在上面的示例中，virtualbox将创建两个host-only网络(除非对应网段的host-only网络已存在)。

![](/img/virtual/2020_10_16_1602857624979.png)

下图是我根据上面的vagrantfile创建虚拟机后，物理机上为virtualbox的host-only创建的虚拟网卡，其中`#2`和`#3`是本次为虚拟机创建的虚拟网卡，另一个是virtualbox默认的host-only虚拟网卡。

![](/img/virtual/2020_10_16_1602857524968.png)

另外，vagrant还可以让private_network使用virtualbox的dhcp功能，这将会自动为虚拟机分配IP地址。这时，vagrant会自动决定是否创建新的virtualbox host-only网络。
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos7"

  config.vm.define "test1" do |t|
    t.vm.hostname = "test1"
    t.vm.network "private_network", type: "dhcp"
    t.vm.network "private_network", type: "dhcp"
    
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test1"
    end
  end
  
  config.vm.define "test2" do |t|
    t.vm.hostname = "test2"
    t.vm.network "private_network", type: "dhcp"
    
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test2"
    end
  end
end
```

这三个dhcp类型的网络配置，都会加入同一个virtualbox的host-only网络内。

![](/img/virtual/2020_10_16_1602858475961.png)


## vagrant配置virtualbox的public_network

vagrant为virtualbox配置的public_network，其本质是将虚拟机加入到了virtualbox的桥接网络内。

vagrant在将虚拟机的网卡加入桥接网络时，默认会交互式地询问用户要和哪个宿主机上的网卡进行桥接，一般来说，应该选择可以上外网的物理设备进行桥接。

由于需要非交互式选择或者需要先指定要桥接的设备名，而且不同用户的网络环境不一样，因此**如非必要，一般不在vagrant中为虚拟机配置public_network**。

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos7"

  config.vm.define "test" do |t|
    t.vm.hostname = "test"
    ## 假设宿主机上要桥接的物理网卡地址为192.168.1.0网段

    # 这是对的
    # 且通过bridge参数指定要桥接到哪个设备，如果不指定bridge，则会交互式询问
    t.vm.network "public_network", ip: "192.168.1.111", bridge: "Intel(R) Wi-Fi 6 AX200 160MHz"  
    # 这是错的，和宿主机物理网卡不在同一个网段
    t.vm.network "public_network", ip: "192.168.111.111" 
    
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test"
    end
  end
end
```

还可以使用virtualbox的DHCP自动分配地址：
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos7"

  config.vm.define "test" do |t|
    t.vm.hostname = "test"
    t.vm.network "public_network"  # 使用DHCP

    t.vm.provider "virtualbox" do |vb|
      vb.name = "test"
    end
  end
end
```

## vagrant配置hyper-v的网络

注意，当Hyper-V中具有多个虚拟交换机时，vagrant无法自动配置hyper-v的网络，只能在vagrant up创建虚拟机时通过交互式询问的方式由用户选择要使用hyper-v的哪个网络，Vagrantfile中的所有网络配置参数(包括端口转发的配置)对于hyper-v虚拟机也都不会生效。如果Hyper-V只有一个虚拟交换机(不管它是哪种网络模型)，vagrant都会自动将虚拟机放入这个网络中。

一般来说，在交互式选择的时候，选择Hyper-V的默认网络(default switch)最有可能是期待的网络模式。如果有特殊需求，可了解Hyper-V的网络模型之后再做选择。
