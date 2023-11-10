---
title: 熟练使用vagrant(3)：使用vagrant创建虚拟机示例
p: virtual/vagrant/vagrant_create_vms.md
date: 2020-10-01 17:37:32
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------

# 熟练使用vagrant(3)：使用vagrant创建虚拟机示例

## vagrant创建并启动一个CentOS7

随便进入一个目录，例如`V:\vagrant_test\`，然后执行`vagrant init`：

```powershell
$ cd V:\vagrant_test
$ vagrant init
```

执行`vagrant init`后，将会在当前目录创建一个名为`Vagrantfile`文件，这个文件定义了将要创建和启动的虚拟机的配置。

打开`Vagrantfile`文件，修改或添加几项内容：
```
config.vm.box = "centos7"
config.vm.box_url = "http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2002_01.VirtualBox.box"
```

上面表示创建一个虚拟机，这个虚拟机从给定的URL处下载。

然后执行：
```powershell
$ vagrant up
```

上面输出了一大堆东西，每个步骤所做的事都区分的很清晰。可以加上选项`--debug`进入调试模式，会输出更详细的内容，比如可以查看下载的VirtualBox.box文件保存到了哪里去。

如果成功启动了虚拟机，此时可直接使用`vagrant ssh`进入虚拟机：
```powershell
$ vagrant ssh
PS $ vagrant ssh

# 已进入虚拟机
[vagrant@172 ~]$ whoami
vagrant
[vagrant@172 ~]$ pwd
/home/vagrant
```

如果`vagrant up`过程中遇到错误或卡住(这是常态)，对于初学者，我个人建议不要花时间去排错或网上搜索解决方案，而是应该直接抛弃本次安装的虚拟机并更换安装其他的box。我收集了几个vagrant使用过程中遇到的错误，或许可以参考参考：[vagrant诡异错误收集](/virtual/vagrant/vagrant_errors)。

```powershell
# 删除根据当前目录下的Vagrantfile所安装的虚拟机
$ vagrant destroy --force
```

失败后换成哪个版本呢？比如可以换成不同版本的centos/7，换成由不同人打包的centos7或尝试各种版本的ubuntu等，直到找到能成功启动的box。可以去vagrant官方的镜像仓库<https://app.vagrantup.com/>搜索各种版本的虚拟机镜像，使用这里的虚拟机镜像下载时可能会很慢，我给出了几种解决方案：[解决vagrant下载很慢的问题](/virtual/vagrant/vagrant_speedup)。

## vagrant创建并启动一个Ubuntu

现在使用vagrant去安装一个Ubuntu，涉及的过程已经很清晰了。

不过建议在开启了hyperv或wsl2的情况下，不要使用ubuntu系统。

过程已经很清晰了：
```powershell
mkdir test4
cd test4

vagrant init ubuntu1804 http://mirrors.ustc.edu.cn/ubuntu-cloud-images/bionic/20201007/bionic-server-cloudimg-amd64-vagrant.box

vagrant up
vagrant ssh
```

