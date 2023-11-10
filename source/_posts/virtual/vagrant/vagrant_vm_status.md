---
title: 熟练使用vagrant(6)：查看vagrant虚拟机状态
p: virtual/vagrant/vagrant_vm_status.md
date: 2020-10-01 17:37:35
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(6)：查看vagrant虚拟机状态

通过`vagrant status`或`vagrant global-status`命令可查看vagrant所管理的所有虚拟机的状态(poweroff/running等)。

- `vagrant status`只能查看当前目录下.vagrant目录内的虚拟机状态  
- `vagrant global-status`可查看全局所有虚拟机的状态  

```powershell
# 在.vagrant所在目录下执行
$ vagrant status
Current machine states:

default       running (virtualbox)

# 切换目录，将查看不到上面的虚拟机状态
$ cd ..
$ vagrant status

# 但可使用global-status查看
$ vagrant global-status
id       name    provider   state   directory
------------------------------------------------
2f5fb92  default virtualbox running V:/vagrant_test
```

上面显示有一台虚拟机已成功运行，且运行在virtualbox上，该虚拟机的全局ID为2f5fb92，其名称为default。

注意：  
1. running并不代表虚拟机成功启动，比如卡在开机过程中，它也会显示running。
2. 这里显示的虚拟机ID和name是vagrant为了管理、区分各虚拟机而设置的标识符，只有vagrant工具自身可以看到这些标识符ID或name。  
   - 如果没有在Vagrantfile中使用config.vm.define定义虚拟机的name，那么它们的name默认都是`default`。  
   - ID具有唯一性，name可能会重复出现，比如一大堆name为default的虚拟机  

可使用ID或name来引用对应的虚拟机。其中：
- ID可缩写(比如上面2f5fb92可缩写为2f5)，只要能保证唯一性即可，且ID可全局引用  
- name标识符只有在.vagrant所在目录下才能引用  
- 如果不指定ID或name，则vagrant默认操作当前目录内`.vagrant`目录下所安装的default虚拟机  

例如：
```powershell
# 重启2f5fb92这台虚拟机，使用完整的ID
vagrant reload 2f5fb92

# 将2f5fb92这台虚拟机关机，使用缩写的ID
vagrant halt 2f5

# 使用name，以及省略目标参数
# 都只能操作当前.vagrant所在目录下的虚拟机
vagrant ssh default
vagrant ssh   # ssh进入当前目录下.vagrant内的虚拟机

# 切换其他目录后，id可全局使用，name不可使用
cd ..
vagrant up 2f5
vagrant up default  # 错
```

