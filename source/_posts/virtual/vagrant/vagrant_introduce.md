---
title: 熟练使用vagrant(1)：vagrant简介
p: virtual/vagrant/vagrant_introduce.md
date: 2020-10-01 17:37:30
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(1)：vagrant简介

## vagrant基本概念

vagrant可方便地管理各种类型的虚拟机，包括virtualbox、hyper-v、docker、vmware、kvm。它是vmware/virtualbox/hyperv等虚拟化管理工具的上层集成式管理工具、虚拟机自动化配置工具、虚拟机批量管理工具。支持Windows、MAC以及Linux。

![](/img/virtual/1603759617223.png)

在vagrant中，virtualbox、hyper-v、vmware(不仅vmware收费，vagrant vmware的插件也收费)、docker等称为provider，不同的provider隐含的是不同的底层虚拟化方式。

vagrant默认使用virtualbox，所以要使用vagrant，要求先安装好virtualbox，再装vagrant。

> 注：在开启了hyperv虚拟化或使用wsl2的win10上，vagrant结合virtualbox会出现一些诡异错误。可参考[vagrant诡异错误收集](/virtual/vagrant/vagrant_errors)。

装好vagrant后，可使用`vagrant`命令执行vagrant的所有功能。

```powershell
$ vagrant --version
Vagrant 2.2.10
```

vagrant命令通过各种子命令实现各种功能，外加一些额外的选项。
```powershell
$ vagrant -h
Usage: vagrant [options] <command> [<args>]

    -h, --help       Print this help.

Common commands:
     autocomplete    manages autocomplete installation on host
     box             manages boxes: installation, removal, etc.
     cloud           manages everything related to Vagrant Cloud
     destroy         stops and deletes all traces of the vagrant machine
     global-status   outputs status Vagrant environments for this user
     halt            stops the vagrant machine
     help            shows the help for a subcommand
     init            initializes a new Vagrant environment by creating a Vagrantfile
     login
     package         packages a running vagrant environment into a box
     plugin          manages plugins: install, uninstall, update, etc.
     port            displays information about guest port mappings
     powershell      connects to machine via powershell remoting
     provision       provisions the vagrant machine
     push            deploys code in this environment to a configured destination
     rdp             connects to machine via RDP
     reload          restarts vagrant machine, loads new Vagrantfile configuration
     resume          resume a suspended vagrant machine
     snapshot        manages snapshots: saving, restoring, etc.
     ssh             connects to machine via SSH
     ssh-config      outputs OpenSSH valid configuration to connect to the machine
     status          outputs status of the vagrant machine
     suspend         suspends the machine
     up              starts and provisions the vagrant environment
     upload          upload to machine via communicator
     validate        validates the Vagrantfile
     version         prints current and latest Vagrant version
     winrm           executes commands on a machine via WinRM
     winrm-config    outputs WinRM configuration to connect to the machine

For help on any individual command run `vagrant COMMAND -h`

Additional subcommands are available, but are either more advanced
or not commonly used. To see all subcommands, run the command
`vagrant list-commands`.
        --[no-]color         Enable or disable color output
        --machine-readable   Enable machine readable output
    -v, --version            Display Vagrant version
        --debug              Enable debug output
        --timestamp          Enable timestamps on log output
        --debug-timestamp    Enable debug output with timestamps
        --no-tty             Enable non-interactive output
```

如果想要看子命令的帮助，可在子命令后使用`-h, --help`选项。例如：
```powershell
$ vagrant box -h
Usage: vagrant box <subcommand> [<args>]

$ vagrant box add -h
Usage: vagrant box add [options] <name, url, or path>
```

## vagrant管理虚拟机常用子命令功能介绍

vagrant的子命令不少，可使用`vagrant -h`列出vagrant默认支持的子命令，使用`vagrant list-commands`查看vagrant支持的所有子命令(包括因安装插件而增加的子命令)。

这里只是简单概括常用子命令的功能而不介绍如何使用，后面涉及到的时候自然就会用了。

子命令         | 功能说明
--------------|----------------------
box           | 管理box镜像(box是创建虚拟机的模板)
init          | 初始化项目目录，将在当前目录下生成Vagrantfile文件
up            | 启动虚拟机，第一次执行将创建并初始化并启动虚拟机
reload        | 重启虚拟机
halt          | 将虚拟机关机
destroy       | 删除虚拟机(包括虚拟机文件)
suspend       | 暂停(休眠、挂起)虚拟机
resume        | 恢复已暂停(休眠、挂起)的虚拟机
snapshot      | 管理虚拟机快照(hyperv中叫检查点)
status        | 列出当前目录(Vagrantfile所在目录)下安装的虚拟机列表及它们的状态
global-status | 列出全局已安装虚拟机列表及它们的状态
ssh           | 通过ssh连接虚拟机
ssh-config    | 输出ssh连接虚拟机时使用的配置项
port          | 查看各虚拟机映射的端口列表(hyperv不支持该功能)
