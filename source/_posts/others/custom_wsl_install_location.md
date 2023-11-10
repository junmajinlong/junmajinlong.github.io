---
title: 自定义wsl安装位置以及多wsl共存
p: others/custom_wsl_install_location.md
date: 2020-03-06 18:20:41
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---


# 自定义wsl安装位置

1. 下载wsl的appx镜像https://docs.microsoft.com/en-au/windows/wsl/install-manual，比如下载的Ubuntu 18.04  
2. 将下载的文件的后缀Appx改为zip，然后解压到你想要安装该wsl的位置。比如像安装到Z盘，则解压到Z盘。比如我当前所在目录是在Z，解压后的目录是Ubuntu18.04onWindows_1804  

    ```shell
    $ tree -L 1 Ubuntu18.04onWindows_1804/
    Ubuntu18.04onWindows_1804/
    ├── AppxBlockMap.xml
    ├── AppxManifest.xml
    ├── AppxMetadata
    ├── AppxSignature.p7x
    ├── Assets
    ├── [Content_Types].xml
    ├── fsserver
    ├── install.tar.gz
    ├── resources.pri
    ├── rootfs
    ├── temp
    └── ubuntu1804.exe
    ```

3. 双击ubuntu1804.exe，它会自动在此目录下安装好ubuntu wsl，等待一会即可  
4. 设置PATH环境变量，添加该wsl的路径。打开powershell  

    ```powershell
    $userenv = [System.Environment]::GetEnvironmentVariable("Path", "User")
    [System.Environment]::SetEnvironmentVariable("PATH", $userenv + ";J:\Ubuntu18.04onWindows_1804", "User")
    ```

所以，这种安装方式相当于绿色版的wsl，解压到哪，就运行(安装)在哪。

那么多版本的wsl也非常容易了，随便在哪个位置装都行。

## 我的迁移方式

我个人采用的迁移方式是：

> 注：如果开启了wsl2
>
> 1.如果待迁移的源子系统是wsl1，最好让迁移后的目标子系统也是wsl1。
>
> 2.如果待迁移的源子系统是wsl2，迁移后的目标子系统也是wsl2，则随意。
>
> 说明：
>
> wsl2以虚拟机方式存在，子系统所有内容均存放在一个vhdx虚拟磁盘中。所以，在win10的资源管理器上无法查看、修改存放在这个虚拟磁盘中的文件。就像无法在win上查看或修改vmware虚拟机里的文件一样。
>
> wsl1子系统的文件则是直接展现在win10的资源管理器中的，可随意查看、修改、删除。
>
> 所以在wsl1->wsl2时，最好设置一下默认子系统版本为wsl1
>
> ```
> wsl --set-default-version 1
> ```

我这里迁移的是非常古老的wsl1，Legacy版本(win10刚出wsl没多久的Ubuntu16系统)。

1. 将待迁移的分发版给导出

    ```
    # wsl --export <distribution> <export_name.tar>
    wsl --export Legacy J:\Legacy.tar
    ```

2. 从wsl注销待迁移的分发版本，以便稍后导入

    ```
    wsl --unregister Legacy
    ```

3. 导入

    ```
    # wsl --import <分发版> <安装位置> <文件名> [选项]
    # 如果不是Legacy，则分发版写对应的版本即可，如果迁移的是Legacy，则找到Legacy对应的分发版本
    # Legacy对应哪个分发版本？可用解压工具打开.tar文件，找到里面的/etc/lsb-release文件查看
    # 例如我系统上的Legacy是Ubuntu-16.04
    wsl --import Ubuntu-16.04 J:\Ubuntu-16.04 J:\Legacy.tar
    ```

4. 导入后的用户信息(家目录)是空的，其它路径的文件是齐全的，所以需要手动将原来的家目录下的内容(`C:\Users\malong\AppData\Local\lxss\root`)拷贝到新系统对应的用户家目录下，注意root用户和/home下的用户家目录最好都拷贝。

1. 的用户家目录最好都拷贝。

## 迁移WSL更简单的方式：LxRunOffline

也可以使用更简便的LxRunOffline工具来做迁移。项目地址：<https://github.com/DDoSolitary/LxRunOffline>。用法非常简单，打开powershell：

```powershell
# 查看当前已经安装的wsl
PS G:\桌面\LxRunOffline-v3.4.0> .\LxRunOffline.exe list
Legacy
Ubuntu-18.04

# 移动指定的wsl
# 比如移动Legacy到Z:\LegacyWSL目录下
PS G:\桌面\LxRunOffline-v3.4.0> .\LxRunOffline.exe move -n Legacy -d 'Z:\LegacyWSL\'
```

