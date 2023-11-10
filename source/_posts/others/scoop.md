---
title: Windows Scoop包管理器的使用
p: others/scoop.md
date: 2022-08-22 18:20:41
tags: [Others,windows]
categories: [Others,windows]
---

# Scoop包管理器安装、配置和使用

Scoop是Windows下比较好用的包管理器，它安装的软件都是"绿色"的，都集中安装在指定的目录下，卸载时不会有文件残留。

> 安装文档：<https://github.com/ScoopInstaller/Install>.
>
> 使用文档: <https://scoop-docs.vercel.app/docs/>

## 安装Scoop

首先修改PowerShell的执行策略：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

然后安装Scoop并设置环境变量：如果中途出现安装一半失败，去把Scoop目录(比如下面的`D:\Scoop`)删掉重新安装。

```powershell
# 对于非管理员用户(打开非管理员模式的powershell)：
$env:SCOOP='D:\Scoop\User'
$env:SCOOP_GLOBAL='D:\Scoop\Global'
[environment]::setEnvironmentVariable('SCOOP',$env:SCOOP,'User')
[Environment]::SetEnvironmentVariable('SCOOP_GLOBAL', $env:SCOOP_GLOBAL, 'Machine')
irm get.scoop.sh -Proxy 'http://<ip:port>' | iex

# 或者也可以采用如下方式安装：
irm get.scoop.sh -Proxy 'http://<ip:port>' -outfile 'install.ps1'
# 可执行./install.ps1 -?来查看使用方式
.\install.ps1 -ScoopDir 'D:\Scoop\User' -ScoopGlobalDir 'D:\Scoop\Global' -Proxy 'http://<ip:port>'

# 对于管理员用户(打开管理员模式的powershell)：
irm get.scoop.sh -Proxy 'http://<ip:port>' -outfile 'install.ps1'
.\install.ps1 -RunAsAdmin -ScoopDir 'D:\Scoop\User' -ScoopGlobalDir 'D:\Scoop\Global' -Proxy 'http://<ip:port>'
# I don't care about other parameters and want a one-line command
iex "& {$(irm -Proxy 'http://<ip:port>' get.scoop.sh)} -RunAsAdmin"
```

安装之后，执行`scoop help`可以查看scoop的使用帮助。例如，`scoop install sudo`可以在Win下安装sudo程序，从而可以辅助使用`sudo scoop`。

scoop安装之后，scoop自身也是一个被scoop管理的程序，可以通过如下方式更新scoop：

```powershell
# 更新自身
scoop update
scoop update scoop

# 更新某个程序
scoop update sudo
```

卸载scoop：

```powershell
scoop uninstall scoop
```

安装常用工具：

```powershell
# utils
# innounp是用来提取setup.exe文件的
scoop install 7zip curl sudo git openssh coreutils grep sed gawk less innounp

# programming languages
scoop install python ruby go nodejs
```

## 配置Scoop

安装完Scoop之后，做一些基本的配置。scoop的配置文件默认为`C:\Users\<USERNAME>\.config\scoop`。

配置scoop使用代理：

```powershell
scoop config proxy <HOST:PORT>
```

配置Scoop安装软件时的安装目录。在安装Scoop时，指定了安装路径(例如`D:\Scoop\User`)，但这可以通过环境变量的方式改变，包括全局安装的目录也可以改变。

```powershell
# 通过环境变量修改用户的安装目录
$env:SCOOP='C:\scoop'
[environment]::setEnvironmentVariable('SCOOP',$env:SCOOP,'User')

# 通过环境变量修改全局安装目录
$env:SCOOP_GLOBAL='c:\apps'
[Environment]::SetEnvironmentVariable('SCOOP_GLOBAL', $env:SCOOP_GLOBAL, 'Machine')
scoop install -g <app>
```

安装`aria2`之后，scoop就会默认使用aria2来下载。当然，也可以配置使用更多的并发下载，也可以将其禁用等。但需注意，使用aria2之后，scoop自身设置的代理将不会生效于aria2的下载，可以通过配置aria2的选项来设置其下载时的代理，当然也可以禁用aria2从而让scoop使用自身的代理。

```powershell
scoop install aria2

# 配置aria2更大的并发下载连接数(默认5)
scoop config aria2-max-connection-per-server 10

# 配置aria2的代理选项
scoop config aria2-options --all-proxy=<HOST:PORT>

# 禁用aria2下载
scoop config aria2-enabled false
```

scoop默认会以当前用户安装程序，如果想要安装到全局路径或让所有用户都可以使用，那么需要使用`scoop install <app> -g`选项，这需要管理员权限。为了简化授权操作，可安装`sudo`程序，功能类似于Unix下的sudo命令：
```powershell
scoop install sudo

sudo install 7zip -g
sudo update 7zip -g
```

## 管理Bucket

scoop使用Bucket作为软件源，官方的Bucket是main，main中的程序只包含非Gui程序，且上传到main Bucket中要求非常严格，所以main Bucket中的程序并不太多。

因此，有必要添加其它的bucket。最常用的是`Extras Bucket`。

```powershell
# 添加extras bucket
$ scoop bucket add extras
$ scoop bucket add extras https://github.com/ScoopInstaller/Extras.git

# 移除 bucket
$ scoop bucket rm extras

# 列出已添加的bucket
$ scoop bucket list
Name   Source                                   Updated            Manifests
----   ------                                   -------            ---------
extras https://github.com/ScoopInstaller/Extras 2022-08-23 8:38:06      1659
main   https://github.com/ScoopInstaller/Main   2022-08-23 8:37:54      1069

# 列出目前已知的无需指定仓库地址就可以添加的bucket，例如添加extras时无需指定参考地址
$ scoop bucket known
main
extras
versions
nirsoft
php
nerd-fonts
nonportable
java
games
```

添加收集了部分国内软件的Bucket dorado，程序不多，但源在中国，下载较快:

```powershell
scoop bucket add dorado https://github.com/h404bi/dorado
```

添加收集了JetBrain全家桶的Bucket：

```
scoop bucket add JetBrain https://github.com/Ash258/Scoop-JetBrains
# scoop install RubyMine
```

## Scoop别名

scoop别名的功能，可以简化PowerShell或CMD下的较长命令。例如简化`scoop install`命令，建立一个别名：

```powershell
scoop alias add i 'scoop install $args[0]' 'Install App'
```

之后就可以直接使用`scoop i APP`来安装软件了。

## 切换包的版本

类似于Ruby的包管理工具`rbenv`，scoop也可以切换不同的版本环境，例如安装了两个版本的Python，需要从当前版本切换到另一个版本去开发。

Scoop的版本切换依赖于`versions Bucket`，因此需先安装它：
```powershell
scoop install versions
```

然后通过`scoop reset`来切换版本。例如：

```powershell
scoop install python27 python
python --version # -> Python 3.6.2

# switch to python 2.7.x
scoop reset python27
python --version # -> Python 2.7.13

# switch back (to 3.x)
scoop reset python
python --version # -> Python 3.6.2
```

# chocolate

安装在指定的位置：设置系统级的环境变量`ChocolateyInstall` 到目标路径，然后重启powershell终端。

如果不设置该环境变量，则默认安装在`%PROGRAMDATA%\Chocolatey`。

安装Chocolate在默认位置：

```
# powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

如果已经安装好了，想要改变安装路径，则修改系统级的环境变量`ChocolateyInstall` 到目标路径，然后拷贝原来的目录到目标路径，同时修改PATH环境变量。

设置代理：

```
choco config set proxy http://127.0.0.1:8118
```

安装和卸载：

```
# 安装包
choco install -y 7zip curl
# 卸载
choco uninstall -f -y 7zip
```
