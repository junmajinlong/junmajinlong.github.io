---
title: 让WSL拥有绝对的Win10管理员权限
p: others/wsl_absolute_admin.md
date: 2020-10-06 18:20:42
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---

# 让WSL拥有绝对的Win10管理员权限

Win10里的管理员经常会权限不足(特别是对C盘的一些文件)，无法删除无法修改某些文件等。

要获取绝对的管理员权限，可通过dism++打开对应程序，例如`cmd`、`powershell`。

有些人对Linux shell会更熟悉一点，想要通过Linux Shell来操作Windows系统里的文件，在【春哥附体】里输入`wsl`即可。

![](/img/others/1601445409952.png)

这样打开的wsl就是具有绝对管理员权限的wsl，可以直接删除任意文件。

如果要指定打开的wsl分发版本，可在【春哥附体】中输入`wsl -d <dis_name>`，如`wsl -d Ubuntu-16.04`。

另外，通过【PowerRun：<https://www.sordum.org/9416/powerrun-v1-4-run-with-highest-privileges/>】这个工具也能让程序获取绝对管理员权限。