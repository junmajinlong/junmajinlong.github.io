---
title: 熟练使用vagrant(2)：设置VAGRANT_HOME
p: virtual/vagrant/vagrant_home.md
date: 2020-10-01 17:37:31
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------

# 熟练使用vagrant(2)：设置VAGRANT_HOME

vagrant在执行子命令box add、init、up等命令时，都可能会去下载所需的虚拟机镜像文件，即Box image。

这些镜像文件默认放在`~/.vagrant.d`目录(Linux)或`C:\USERS\<NAME>\.vagrant.d\`目录(Windows)下，这些镜像文件一般至少都几百兆，大的镜像可能几个G，所以放在家目录或C盘会占用大量空间。

要修改vagrant下载时默认的镜像保存位置，需设置环境变量`VAGRANT_HOME`。

Windows设置`VAGRANT_HOME`环境变量方式：

![](/img/virtual/2020_10_12_1602517288955.png)

设置后重新打开cmd或powershell，如果cmd中`echo %VAGRANT_HOME%`输出的内容或powershell中`$env:VAGRANT_HOME`输出的内容确实是设置的目录，则没问题。

![](/img/virtual/2020_10_12_1602517930370.png)

Linux设置`VAGRANT_HOME`环境变量方式：
```bash
echo 'export VAGRANT_HOME="/data/.vagrant.d"' >>~/.bashrc
exec bash
```

设置`VAGRANT_HOME`环境变量后，vagrant下载的box镜像文件将放在指定的目录下。

