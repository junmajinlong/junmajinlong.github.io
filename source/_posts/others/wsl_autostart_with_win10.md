---
title: 让wsl中的服务随Win10开机自启动
p: others/wsl_autostart_with_win10.md
date: 2020-10-06 18:20:43
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---

# 让wsl中的服务随Win10开机自启动

比如让sshd/cron等服务或某个命令、脚本在启动win10的时候自动在wsl中启动或执行。

>注：
>
>(1).wsl中无法直接使用systemd系统，因此无法通过systemctl设置自启动服务。
>
>(2).另一方面，使用/etc/init.d/xxx start启动的服务会在关闭wsl终端后自动退出。

**步骤**：


(1).确定要在哪个wsl分发版本上设置自启动：

```powershell
# 在cmd/powershell执行
PS C:\Windows\system32> wsl -l
适用于 Linux 的 Windows 子系统:
Legacy (默认)
Ubuntu-18.04
```

这里列出了两个wsl分发版本(Legacy和Ubuntu-18.04)，如果我想在Ubuntu-18.04上部署开机自启动服务。

注：建议不要使用wsl2来设置服务开机自启动，wsl2是一个完整的虚拟机系统，启动wsl2相比wsl1要慢一些。因此，win10开机就启动wsl2的话，会导致在慢速启动wsl2的过程中一直有一个黑窗口存在。


(2).在wsl Ubuntu-18.04中设置无密码sudo：

```shell
# 在wsl Ubuntu-18.04内执行
sudo bash -c "echo '$USER ALL=(ALL) NOPASSWD: ALL' >/etc/sudoers.d/$USER"
```

(3).在`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`中创建快捷方式，其命令行大致如下：

![](/img/others/1587465418748.png)

(4).重启win10验证。

## 使用wsl-distrod工具安装和配置开机自启动

wsl-distrod工具可以快速安装各种wsl2的系统，并设置开机自启动以及端口暴露。

项目地址：https://github.com/nullpo-head/wsl-distrod。







