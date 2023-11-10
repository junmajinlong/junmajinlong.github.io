---
title: wsl安装多个同发行版本子系统
p: others/wsl_same_distro_versions.md
date: 2020-10-06 18:20:41
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---

# wsl安装多个同发行版本子系统

默认情况下，wsl不允许同时安装多个相同发行版本的子系统。

例如，可以同时安装Ubuntu 16、Ubuntu 18、Ubuntu 20，但不能同时安装两个Ubuntu 18。在安装第二个Ubuntu 18时，会自动打开已经安装的Ubuntu 18。

如果确实想要安装多个相同发行版的子系统(例如多个Ubuntu 18)，需导出Ubuntu 18，然后以wsl 1的方式多次导入该子系统。这种方式个人觉得非常不方便。

好在，有位大佬开发了一个工具：LxRunOffline。LxRunOffline可非常方便地管理wsl，包括本文的目标：同时安装两个相同的发行版子系统。

LxRunOffline项目地址：<https://github.com/DDoSolitary/LxRunOffline>。

LxRunOffline用法非常明确：

```
$ .\LxRunOffline.exe
[ERROR] No action is specified.

Supported actions are:
    l, list            List all installed distributions.
    gd, get-default    Get the default distribution, which is used by bash.exe.
    sd, set-default    Set the default distribution, which is used by bash.exe.
    i, install         Install a new distribution.
    ui, uninstall      Uninstall a distribution.
    rg, register       Register an existing installation directory.
    ur, unregister     Unregister a distribution but not delete the installation directory.
    m, move            Move a distribution to a new directory.
    d, duplicate       Duplicate an existing distribution in a new directory.
    e, export          Export a distribution's filesystem to a .tar.gz file, which can be imported by the "install" command.
    r, run             Run a command in a distribution.
    di, get-dir        Get the installation directory of a distribution.
    gv, get-version    Get the filesystem version of a distribution.
    ge, get-env        Get the default environment variables of a distribution.
    se, set-env        Set the default environment variables of a distribution.
    ae, add-env        Add to the default environment variables of a distribution.
    re, remove-env     Remove from the default environment variables of a distribution.
    gu, get-uid        Get the UID of the default user of a distribution.
    su, set-uid        Set the UID of the default user of a distribution.
    gk, get-kernelcmd  Get the default kernel command line of a distribution.
    sk, set-kernelcmd  Set the default kernel command line of a distribution.
    gf, get-flags      Get some flags of a distribution. 
    sf, set-flags      Set some flags of a distribution. 
    s, shortcut        Create a shortcut to launch a distribution.
    ec, export-config  Export configuration of a distribution to an XML file.
    ic, import-config  Import configuration of a distribution from an XML file.
    sm, summary        Get general information of a distribution.
    version            Get version information about this LxRunOffline.exe.
```

例如，列出当前已经安装的发行版：

```
$ .\LxRunOffline.exe l
Ubuntu-18.04
Ubuntu-20.04
Ubuntu-16.04
CentOS7
```

对于本文同时安装多个相同发行版的子系统的目标，要用到的子命令是`d, duplicate`，这将会直接拷贝指定的发行版并进行注册。

直接执行子命令会给出进一步的错误帮助信息：

```
$ .\LxRunOffline.exe d
[ERROR] the option '-N' is required but missing

Options:
  -n arg                Name of the distribution
  -d arg                The directory to copy the distribution to.
  -N arg                Name of the new distribution.
  -c arg                The config file to use. This argument is optional.
  -v arg (=4294967295)  The version of filesystem to use, same as source if not
                        specified.
```

因此，如果我想要再安装一个Ubuntu 18.04，可执行如下命令：

```
$ .\LxRunOffline.exe d  -n Ubuntu-18.04 -d V:\wsl\Ubuntu-18.04-22\ -N Ubuntu-18.04-1

[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\full" of type 0020000.
[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\null" of type 0020000.
[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\ptmx" of type 0020000.
[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\random" of type 0020000.
[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\tty" of type 0020000.
[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\urandom" of type 0020000.
[WARNING] Ignoring an unsupported file "\\?\V:\wsl\Ubuntu-18.04\rootfs\dev\zero" of type 0020000.
```

警告信息可忽略。

现在，有一个Ubuntu-18.04就可以使用了。

```
$ wsl -l -v
  NAME               STATE           VERSION
* Ubuntu-16.04       Stopped         1
  Ubuntu-18.04-1     Stopped         1
  Ubuntu-18.04       Stopped         1
  Ubuntu-20.04       Stopped         2
  CentOS7            Stopped         1
```

