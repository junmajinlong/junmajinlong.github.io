---
title: 熟练使用vagrant(14)：vagrant链接克隆
p: virtual/vagrant/vagrant_linked_clone.md
date: 2020-10-01 17:37:43
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(14)：vagrant链接克隆

vagrant支持对Virtualbox的虚拟机进行链接克隆。通过链接克隆的方式创建的虚拟机所占用的空间会比较小。

使用链接克隆的方式很简单：
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic"

  # 在全局范围内设置linked_clone
  # 使得其他虚拟机都会采用链接克隆的方式创建虚拟机
  config.vm.provider "virtualbox" do |vb|
    vb.linked_clone = true
  end

  config.vm.define "clone_vm1" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "clone_vm1"
    end
  end
  config.vm.define "clone_vm2" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "clone_vm2"
    end
  end
  config.vm.define "not_clone_vm3" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "not_clone_vm3"
      vb.linked_clone = false
    end
  end
end
```

在上面的Vagrantfile配置中，在全局的provider语句块内设置了`linked_clone = true`，这会使得本配置文件中其他(Virtualbox)主机都会采用链接克隆的方式去创建虚拟机，除非在某个具体的虚拟机定义语句块内设置了`linked_clone = false`。

因此，在执行`vagrant up`后，上面的配置将创建4个虚拟机，其中一个是链接克隆的模板虚拟机，vm1和vm2是基于该模板虚拟机通过链接克隆方式创建的虚拟机，vm3不是克隆而来的，而是完整的虚拟机。

需了解的是，用于克隆而创建的虚拟机模板只在第一次创建，且不会启动，它也不受vagrant管理(就像是手动安装的virtualbox虚拟机一样)。因此模板虚拟机只能自己手动在VirtualBox上删除。但删除时要确保已经没有其他被克隆而来的虚拟机仍在使用该模板虚拟机。

另外，并非一定要求在全局配置中设置linked_clone，在某个虚拟机的局部配置中也可以设置。

例如，下面的Vagrantfile中，test1和test3中没有设置linked_clone，它们将直接导入box镜像来安装系统，test5中设置`linked_clone=false`也一样如此。而test2和test4则设置了`linked_clone=true`，因此test2和test4将基于一个完整的虚拟机模板以链接克隆的方式创建虚拟机(如果上面示例克隆虚拟机的模板没有删除，下面的示例将直接使用这个模板，vagrant会自动寻找克隆模板，尽量确保只在第一次创建克隆模板)。
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic"

  config.vm.define "test1" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test1"
      vb.cpus = 1
      vb.memory = 512
    end
  end

  config.vm.define "test2" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test2"
      vb.memory = 512
      vb.cpus = 2
      vb.linked_clone = true
    end
  end

  config.vm.define "test3" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test3"
      vb.memory = 512
      vb.cpus = 4
    end
  end

  config.vm.define "test4" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test4"
      vb.memory = 512
      vb.cpus = 4
      vb.linked_clone = true
    end
  end

  config.vm.define "test5" do |t|
    t.vm.provider "virtualbox" do |vb|
      vb.name = "test5"
      vb.memory = 512
      vb.cpus = 4
      vb.linked_clone = false
    end
  end
end
```
