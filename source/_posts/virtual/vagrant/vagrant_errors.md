---
title: 熟练使用vagrant(18)：vagrant诡异错误收集
p: virtual/vagrant/vagrant_errors.md
date: 2020-10-01 17:37:47
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(18)：vagrant诡异错误收集

所有win10+virtualbox的诡异问题，如果在hyperv下安装相同的box没问题，几乎可以肯定是wsl2或win10开启了虚拟化(hyperv)导致的问题。据我个人使用经验，在Win10下，vagrant+hyperv可以减少很多问题，而开启了虚拟化的win10下的vagrant+virtualbox，可能会遇到各种疑难杂症。

这种问题没有合理的解决方案，但可以尝试cmd或powershell下执行：
```
bcdedit /set hypervisorlaunchtype off
```
然后重启。

这可能会解决问题，但这个操作会导致wsl2不可用。如果想要重新开启：
```
bcdedit /set hypervisorlaunchtype auto
```

使用无参数的bcdedit命令可以查看hypervisorlaunchtype的状态。

## 可放心使用的centos7和ubuntu box

如果win10没有开启虚拟化或没有wsl2，那么任何版本的centos7、ubuntu应该都不会有问题。

另外，在win10下，任何版本的centos7、ubuntu在hyperv下或在没有开启hyperv虚拟化的virtualbox上都没有问题。

在开启了hyperv虚拟化(包括使用了wsl2)的win10下，结合virtualbox时：
- 下面的box是我测试过没有问题的：generic/centos7  
- 下面的box是我测试过有问题的：  
  - 官方box镜像：centos/7  
  - 各种ubuntu版本，在`apt update`时故障  

## virtualbox下安装CentOS7开机时卡住

vagrant命令行在安装官方centos/7镜像时，开机过程中卡在`SSH auth method: private key`，打开virtualbox的显示界面，卡在开机进度条上(一段时间后自动进入救援模式)或者卡在【tsc: Refined TSC clocksource calibration】。

![](/img/virtual/2020_10_14_1602641758660.png)

可执行本节开头的bcdedit命令解决：
```
bcdedit /set hypervisorlaunchtype off
# 执行完后重启
```

另外，可进入虚拟机centos7的bios关闭虚拟化功能，也可解决。

上面两种方法虽然都能解决问题，但是都不友好，无法非交互式完成。

据我个人测试，在hyperv下无此问题，且换个centos7的box版本而不是使用官方的centos/7也无此问题，比如generic/centos7可正常开机。

## 安装ubuntu时apt失败

vagrant安装ubuntu时，启动虚拟机后进入ubuntu虚拟机，执行`apt update`或`apt install`时出现类似如下的错误：
```
Err:6 http://security.ubuntu.com/ubuntu focal-security/main amd64 Packages
  Hash Sum mismatch
  Hashes of expected file:
   - Filesize:1291458 [weak]
   - SHA256:407e1e5f8188d9a38dec308c87203aadb16cb4ac173b729005525c4d824317c9
   - SHA1:47f50537ae094c725a60e90bf5e72d563a73abb2 [weak]
   - MD5Sum:05500c05300ff2f3be66040cd9d4c24b [weak]
  Hashes of received file:
   - SHA256:fc9c8ae3385f3b2cdb7c52c8e4ee5fa5074c843155b7ce4786b3ce3c97fe8c3f
   - SHA1:59b7721865c7a9c1b313ce11f11aefa4c7951fd4 [weak]
   - MD5Sum:05500c05300ff2f3be66040cd9d4c24b [weak]
   - Filesize:1291458 [weak]
```

尽管这个报错是在虚拟机内部，但这是win10开启了虚拟化导致的，好在这个问题可以在ubuntu内部直接解决，而无需执行本节开头的bcdedit命令。

```bash
sudo mkdir /etc/gcrypt
sudo bash -c 'echo all >>/etc/gcrypt/hwf.deny'
```

然后`sudo apt update`即可正常工作。

参考：<https://stackoverflow.com/a/64157969>。

