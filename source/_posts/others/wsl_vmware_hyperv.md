---
title: VMware和HyperV(wsl2)共存后性能大幅下降的解决方案
p: others/wsl_vmware_hyperv.md
date: 2023-10-19 18:20:43
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---

# VMware和HyperV(wsl2)共存后性能大幅下降的解决方案

为了使用WSL2，要求开启HyperV的虚拟化，但开了HyperV之后，VMWare就不能用了。

虽然从VMWare16版本开始vmware已经能和wsl2、hyper-v共存(记忆中好像vmware 16版之前就有几个小版本支持和hyper-v、wsl2共存)，但这个共存是付出了巨大代价的：VMWare的性能大幅下降。

我并没有具体测试性能到底下滑了多少，但是可以对比曾经和现在使用VMWare的流畅度：

- 老电脑是宏碁4750G配置：cpu i5-2410m，显卡GT520，内存8G，vmware里装win7、win server 2008/2012，使用机械硬盘好多年，还开好几个虚机多开DNF，跑Exchange、SQL Server集群等等，换成SATA固态之后体验更丝滑，丝毫不感觉虚拟机有卡顿延迟不流畅的感觉  
- 现在的电脑配置：CPU i7-7700K，显卡GTX1060 6G，内存32G，NVME固态。和老电脑相比，配置带来的提升至少是1到2个数量级。然而开启HyperV之后，VMWare里装Win7，只是装几个软件(qq、微信、百度网盘、迅雷)，虚拟机就卡的要死，流畅度很差，延迟卡顿到恼火。关闭HyperV虚拟化之后，瞬间丝滑了  

所以，WSL2和流畅的VMWare只能二选一。

我本人曾经重度使用wsl2，但现在基本上只使用wsl1。不使用wsl2的原因有多方面，虽然现在(windows10 22H2,2023年)已经改善了wsl2一些不友好的地方，但wsl2依然有一个缺陷让我不得不放弃wsl2：我在Win宿主机上有几十G的数据需要Linux子系统里的程序去读写，而wsl2读写win宿主机上的数据是非常慢的，另一方面，wsl1读写win宿主机上的数据并不慢，性能也不差。

所以，我选择仅使用wsl1并且拥抱流畅的VMWare，忍痛放弃wsl2(虽然一点也不痛)。但是，我也不是完全放弃wsl2，而是以多系统隔离的方式让它们共存了。

## VMware和HyperV以多系统隔离的方式共存

描述一下现在我的两个win系统：

- win10：wsl + wsl2 + vmware，也就是说，这个系统中开启了hyper-v，可以使用wsl2，但是容忍vmware的性能下降 
- win10_nohyperv：wsl + vmware，也就是说，这个系统中关闭了Hyper-V，使用wsl和没有性能损失的vmware

生成这样的多系统隔离很简单，执行几个命令即可。

假如，现在的系统win10是开启了hyper-v虚拟化的(如果使用了wsl2，记得先把所有的wsl2子系统备份或者取消注册，后面操作完成了再重新注册回来)，打开命令提示符(注意，不能是powershell)执行下面的命令，这将复制当前系统生成一个关闭了hyper-v虚拟化功能的新系统：

```
$ bcdedit /copy {current} /d "Win10_nohyperv”
# 输出：已将该项成功复制到 {48294053-70b2-11ee-9c25-9e34ad6831ea}，复制大括号里的内容

$ bcdedit /set {48294053-70b2-11ee-9c25-9e34ad6831ea} hypervisorlaunchtype OFF

# 现在就有两个系统了，一个是开启了HyperV虚拟化的，一个是关闭了HyperV虚拟化的
$ bcdedit
Windows 启动管理器
--------------------
标识符                  {bootmgr}
device                  partition=\Device\HarddiskVolume8
path                    \EFI\MICROSOFT\BOOT\BOOTMGFW.EFI
description             Windows Boot Manager
locale                  zh-CN
inherit                 {globalsettings}
default                 {current}
resumeobject            {48294053-70b2-11ee-9c25-9e34ad6831ea}
displayorder            {48294054-70b2-11ee-9c25-9e34ad6831ea}
                        {current}
toolsdisplayorder       {memdiag}
timeout                 3

Windows 启动加载器
-------------------
标识符                  {48294054-70b2-11ee-9c25-9e34ad6831ea}
device                  partition=C:
path                    \WINDOWS\system32\winload.efi
description             Win10_22H2
locale                  zh-CN
inherit                 {bootloadersettings}
recoverysequence        {48294057-70b2-11ee-9c25-9e34ad6831ea}
displaymessageoverride  Recovery
recoveryenabled         Yes
isolatedcontext         Yes
allowedinmemorysettings 0x15000075
osdevice                partition=C:
systemroot              \WINDOWS
resumeobject            {48294053-70b2-11ee-9c25-9e34ad6831ea}
nx                      OptIn
bootmenupolicy          Standard
hypervisorlaunchtype    Auto

Windows 启动加载器
-------------------
标识符                  {current}
device                  partition=C:
path                    \WINDOWS\system32\winload.efi
description             Win10NoHyperV
locale                  zh-CN
inherit                 {bootloadersettings}
recoverysequence        {48294057-70b2-11ee-9c25-9e34ad6831ea}
displaymessageoverride  Recovery
recoveryenabled         Yes
isolatedcontext         Yes
allowedinmemorysettings 0x15000075
osdevice                partition=C:
systemroot              \WINDOWS
resumeobject            {48294053-70b2-11ee-9c25-9e34ad6831ea}
nx                      OptIn
bootmenupolicy          Standard
hypervisorlaunchtype    Off
```

可以通过`msconfig`命令打开系统引导窗口，修改一下默认启动的系统和等待时间间隔。

现在重启操作系统，就会有两个系统供你选择，在Win10_nohyperv系统中就关闭了HyperV的虚拟化功能，可以流畅地使用VMware和wsl1，但不能使用wsl2。

最后，在开启了HyperV虚拟化的系统里，重新恢复注册wsl2子系统即可。