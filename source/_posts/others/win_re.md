---
title: 重建Windows的恢复分区(RE)
p: others/win_re.md
date: 2023-10-16 18:20:41
tags: [Others,windows]
categories: [Others,windows]
---

# 重建Windows的恢复分区(RE)

Re环境被disabled了，说明恢复分区不存在。

```
reagentc /info

Windows 恢复环境(Windows RE)和系统初始化配置
信息:

    Windows RE 状态:           Disabled
    Windows RE 位置:
    引导配置数据(BCD)标识符:   19285d63-6ff2-11ee-9e01-f7bfb87067d9
    恢复映像位置:
    自定义映像位置:
    自定义映像索引:            0

REAGENTC.EXE: 操作成功。
```

操作步骤：

1.下载一个Windows安装镜像并解压
2.拷贝"解压目录\sources\install.wim"到某个位置(如D:\win10\install.wim)
3.执行下面的命令解压释放install.wim中的内容到某个目录(如D:\re_install)

```
dism /mount-wim /wimfile:D:\win10\install.wim /index:1 /mountdir:D:\re_install
```
4.从释放目录(D:\re_install\Windows\System32\Recovery)中找到Winre.wim拷贝到C:\Recovery,C:\Recovery可能是隐藏的，也可能不存在，不存在则创建它。拷贝后，卸载镜像：
```
dism /Unmount-Wim /Mountdir:D:\re_install /discard
```
5.执行下面的命令注册WinRe
```
reagentc /setreimage /path C:\Recovery
reagentc /enable
```
6.查看
```
reagentc /info
Windows 恢复环境(Windows RE)和系统初始化配置
信息:

    Windows RE 状态:           Enabled
    Windows RE 位置:           \\?\GLOBALROOT\device\harddisk0\partition3\Recovery\WindowsRE
    引导配置数据(BCD)标识符:   19285d65-6ff2-11ee-9e01-f7bfb87067d9
    恢复映像位置:
    恢复映像索引:              0
    自定义映像位置:
    自定义映像索引:            0

REAGENTC.EXE: 操作成功。
```

7.如果需要下次启动进入RE环境，执行`reagentc /boottore`命令即可。或者通过Windows的`设置->更新和安全->恢复->高级启动->立即重新启动`来立即进入RE环境。