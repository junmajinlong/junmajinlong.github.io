---
title: windows隐藏程序运行窗口
p: others/hide_window.md
date: 2022-12-14 18:20:41
tags: [Others,windows]
categories: [Others,windows]
---

# windows隐藏程序运行窗口

在Windows中，有时候想让某些程序在后台运行，比如某些开机自启动但启动时无法隐藏窗口的程序，包括那些启动后是一些黑(dos)窗口的程序。

## 使用Nircmd工具

Nircmd官方下载页面：<https://www.nirsoft.net/utils/nircmd.html>，拉到页面的最底部，有下载zip文件和exe格式的链接，下载zip文件即可。

nircmd是一个功能很丰富的命令行工具，其`exec`子命令可调用其它程序，并可指定以最大窗口、最小窗口、隐藏窗口的方式运行。例如:

```powershell
# 运行 E:\app\rdesk\rdesk.exe 并隐藏窗口，
# 同时为rdesk指定了选项-c和参数config.json
.\nircmd.exe exec hide E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json

# 最小窗口运行程序
# .\nircmd.exec exec min E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json
# 最大窗口运行程序
# .\nircmd.exec exec max E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json
```

如果要运行的程序需要以管理员权限运行，则使用如下方式：

```powershell
.\nircmd.exe elevatecmd exec hide YOUR-CMD
```

## 使用silentCmd工具

silentCmd是一个开源项目，项目地址：<https://github.com/stbrenner/SilentCMD>。

下载：<https://github.com/stbrenner/SilentCMD/releases/download/v1.4/SilentCMD.zip>。

silentCmd默认隐藏运行窗口，且支持日志和延迟调用功能。

```powershell
.\silentCMD.exe E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json

# 延迟15秒调用rdesk
.\silentCMD.exe E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json /DELAY:15
```

## 使用hideexec工具

下载hidexec: <http://code.kliu.org/misc/hideexec/>

下载后解压，里面有源码目录(src)、32位和64位已编译的二进制程序。我电脑64位，所以使用64位的程序。

```powershell
hideexec-1.2.3-redist\bin.x86-64\hideexec.exe E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json
```

## 其它方法

当然还有很多方法，包括但不限于：

- 使用vbs脚本调用程序  
- 使用[winsw](https://github.com/winsw/winsw)工具制作成服务来运行  
- 使用[CommandTrayHost](https://github.com/rexdf/CommandTrayHost)工具制作成托盘运行方式  
- ...

# Windows开机自启动程序并隐藏窗口

有些程序启动就带有窗口，想要让这类程序开机自启动并在自启动时隐藏窗口，可创建快捷方式，并通过上面的一些工具来调用这些带有窗口的程序。

例如，随便找一个文件创建快捷方式，然后重命名，并在属性的`目标框`中通过nircmd或其它命令来调用程序，同时属性的`起始位置框`是nircmd或其它命令的目录。

例如，在目标框中填写：

```
G:\nircmd\nircmd.exe exec hide E:\app\rdesk\rdesk.exe -c E:\app\rdesk\config.json
```

![](/img/others/1671017175330.png)

制作好了快捷方式，就可以将它放在桌面或其它任何位置双击执行，也可以固定到任务栏、开始菜单，也可以放在开机自启动目录(例如`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`)中使其能够开机自启动。

![](/img/others/1671016441098.png)