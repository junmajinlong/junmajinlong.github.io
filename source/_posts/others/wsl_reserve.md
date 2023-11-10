---
title: 重装win10系统时保留wsl子系统
p: others/wsl_reserve.md
date: 2021-01-06 15:20:41
tags: [Others,windows,wsl]
categories: [Others,windows,wsl]
---

# 重装win10系统时保留wsl子系统

在换win10系统的时候，如果想在新系统中继续使用原有的wsl子系统，可使用工具：LxRunOffline。

LxRunOffline项目地址：<https://github.com/DDoSolitary/LxRunOffline>。

假如，在新系统中，原来wsl的目录为`H:\wsl\ubuntu2004`，那么执行：

```powershell
.\LxRunOffline.exe rg -n ubuntu2004 -d H:\wsl\ubuntu2004
```

就会将这个目录里的wsl子系统注册到当前系统，wsl里面的一切都不会发生变化。
