---
title: Windows调整和小技巧
p: others/win.md
date: 2019-07-06 18:20:41
tags: [Others,windows]
categories: [Others,windows]
---

推荐 win 下一些个人爱用的工具软件 (以及使用心得) 和一些系统调整方法，让 win 下不尽人意的设置发生小小变化，让整天摸着电脑的 ITer 们的生活更有乐趣。

本人酷爱收集一些好用的软件，若各位也对某个或某些软件有所心得，不妨 "划下道来"，分享分享，本人感激不尽。

* * *

<a name="shiftspace"></a>

# 禁用shift+space全半角切换

写代码、写 SQL 语句的同志可能会经常性地误按 "shift+space" 将半角输入切换成全角输入法，但实际生活中，绝大多数情况下根本就不需要全角输入，所以直接将它给禁用掉，免得烦恼。

如果是 win7，打开注册表，把 HKEY_CURRENT_USER\Control Panel\Input Method\Hot Keys\00000011 下的 "Key Modifiers"、"Target IME" 和 "Virtual Key" 的二进制值全改为 0 就可以，或者把下面的注册表修改代码放进一个 reg 文件中，然后双机添加就可以。修改完后，最后重启系统即可。

```
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Control Panel\Input Method\Hot Keys\00000011]
"Key Modifiers"=hex:00,00,00,00
"Target IME"=hex:00,00,00,00
"Virtual Key"=hex:00,00,00,00

```

如果想把禁用半全角切换功能重新启用，把注册表改回来即可。以下是启用切换功能的注册表值：

```
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Control Panel\Input Method\Hot Keys\00000011]
"Key Modifiers"=hex:04,c0,00,00
"Target IME"=hex:00,00,00,00
"Virtual Key"=hex:20,00,00,00

```

如果是 win10，也可以用上面的方法。但 win10 有更简单的方法，直接在微软拼音输入法中禁用即可。如果 win10 版本较低，可能还不支持该功能，可以在注册表 HKEY_CURRENT_USER\Control Panel\Input Method 下添加一个键 "Show Status"，并设置它的值为 1 就可以禁用全半角切换，设置为 0 就可以重新启用全半角切换。

如果是禁用，则把下面的注册表修改代码放进一个 reg 文件，并双击执行。然后重启系统就可以。

```
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Control Panel\Input Method]
"Show Status"="0"

[HKEY_CURRENT_USER\Control Panel\Input Method\Hot Keys]

```

**注意，如果设置无效，请看看是不是自己安装的输入法也有 shift+space 相关的设置项。**

![](/img/referer.jpg)

<a name="pdfbm"></a>

# pdf书签提取

我找过不少 pdf 书签提取的工具，都不尽如人意，这个还算不错。

作者原文：[https://blog.csdn.net/yinqingwang/article/details/78736474](https://blog.csdn.net/yinqingwang/article/details/78736474)

下载链接:[https://pan.baidu.com/s/1jIeTsUy](https://pan.baidu.com/s/1jIeTsUy) 密码: i5un

这个工具是 java 写的，因此需要安装好 jre 环境。

解压后，里面有下面几个文件：

```
PDFBOOK.js
PDFBookmark.jar
Run.bat
Run.sh
使用说明.txt
说明 (关于PDFBOOK-js).txt
```

在 unix 环境下，就执行 Run.sh，在 Windows 环境下，就双击 Run.bat。之后会提示选择 pdf 文件。下面是一个示例图。

![](/img/others/733013-20180514153822402-1713715384.jpg)

* * *

<a name="pdfinout"></a>

# pdf书签导入导出

![](/img/referer.jpg)

有时候我们自己做了一个 pdf 文件，想要从另一个 pdf 文件中把书签导入过来，同时每个书签指向的页码位置还是正确的，例如将文字版的 pdf 转换为图片版的 pdf 时就需要导入导出书签。

我用的是 **pdf-XChange** 软件 (请自行搜索下载)。它是一个 pdf 阅读工具，不仅可以复制带页码属性的书签，搜索内容的速度也远比 Adobe Acrobat 类的套件快。不过它功能比较单一，只有阅读，无法编辑 pdf。所以我也就是偶尔在有需求的时候用它来完成一些工作。

例如，我在这里选中全部书签，复制并粘贴到旁边的 2.pdf 中。

![](/img/others/733013-20180330230711715-2088220347.jpg)

![](/img/others/733013-20180330230829305-1633136675.jpg)

在 2.pdf 中，每个书签都是设置好了目标位置的。

一个小缺憾，虽然这个 pdf 软件能在两个 pdf 间复制粘贴书签，但复制的书签却不能粘贴到其他程序中，例如记事本。

![](/img/referer.jpg)

<a name="ts2mp4"></a>

# 加密的m3u8、ts文件合并

见下文：[https://www.junmajinlong.com/others/ts_m3u8](/others/ts_m3u8)

<a name="chrome"></a>

# chrome无法添加扩展程序

现在 chrome 默认不支持外部的扩展程序，直接拖 crx 文件到扩展程序里进去已经失效了。

要想添加外部的扩展程序，需要经过一番设置：

1\. 下载模板文件。
[https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip](https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip)

2\. 解压后，找到 windows/adm/zh-cn/chrome.adm

3.gpedit.msc

4\. 在计算机管理 --> 管理模板 --> 右键新建模板，找到 windows/adm/zh-cn/chrome.adm

5\. 在管理模板 --> 经典管理模板 -->Google-->Google Chrome--> 扩展程序 --> 配置扩展程序白名单

点击启用，并在 "显示" 选择要添加的扩展程序 id。

扩展程序的 id 可以在拖到 chrome 后，并被自动删除前，去 chrome 扩展程序页面查看。

6\. 重启 chrome

<a name="disablekey"></a>

# 禁用笔记本自带键盘

```
sc config i8042prt start= disabled
```

然后重启计算机。

如果想要重新启用自带键盘：

```
sc config i8042prt start= auto
```

<a name="doc2pdf"></a>

# word批量转pdf(带书签)

word 转 pdf 方式很多，批量转为不带书签的 pdf 网上随便一搜索，方法也很简单。

但是要批量将 word 转换为带书签的 pdf 的方法就没那么容易找到，网上有些方法还是借助 c# 来实现的，相当麻烦。

所以写了个 vba 来实现。要求 office 版本高于或等于 2013(2010 应该不行，我没试)。

加入 e:\words \ 目录下有很多 docx 文件。下面的步骤会将这个目录下的所有 docx 文件转换为带书签的 pdf。

1\. 随便打开一个 docx 文件。最好不要是目标目录下临时新建的 word。

2\. 按 alt+f11 插入模块，复制一下代码，保存退出。

```
'例如将d:\a目录下的word转换为pdf，则在非d:\a下新建一个word，打开，alt+f11，插入模块，复制一下代码，按f5，选择D:\a目录就ok
' 只支持docx，如要支持doc，则修改下面对应代码为：fileName = Dir(filePath & "\*.doc")

Sub IAassembleex()

    Dim fileName    As String
    Dim filePath    As String
    Dim wbkThis     As Document
    Dim wbkOpen     As Document
    
    Dim tfil  As Integer
    
    Application.ScreenUpdating = False
    
    Set wbkThis = ThisDocument
    
    tfil = 0
    
    Application.DisplayAlerts = False
    
    With Application.FileDialog(msoFileDialogFolderPicker)
    
        .AllowMultiSelect = False
    
        If .Show = -1 Then
            filePath = .SelectedItems(1)
        End If
    End With
    
    fileName = Dir(filePath & "\*.docx")
    
    Do While fileName <> ""
    
        On Error Resume Next
    
        tfil = tfil + 1
    
        Set wbkOpen = Documents.Open(filePath & "\" & fileName)
    
            ActiveDocument.ExportAsFixedFormat OutputFileName:= _
        filePath &"\"& Left(fileName,InStrRev(fileName,"."))&"pdf", ExportFormat:= _
        wdExportFormatPDF, OpenAfterExport:=False, OptimizeFor:= _
        wdExportOptimizeForPrint, Range:=wdExportAllDocument, From:=1, To:=1, _
        Item:=wdExportDocumentContent, IncludeDocProps:=True, KeepIRM:=True, _
        CreateBookmarks:=wdExportCreateHeadingBookmarks, DocStructureTags:=True, _
        BitmapMissingFonts:=True, UseISO19005_1:=False
    
        wbkOpen.Close False
    
        fileName = Dir
    Loop
 link
    Application.ScreenUpdating = True

    MsgBox ("successfully" & vbCrLf & "total read " & tfil)

End Sub
```

上面的 vba 只支持 docx 文件的转换，如果要支持 doc 文件，将 fileName = Dir(filePath & "\*.docx") 改为 fileName = Dir(filePath & "\*.doc") 就行。

![](/img/others/733013-20180514153308882-942468291.jpg)

3\. 视图 --> 宏 --> 查看宏 --> 运行。然后选择 docx 文件所在目录即可，例如此处是 e:\words 目录。

![](/img/others/733013-20180514153822402-1713715384.jpg)

转换完成后 pdf 文件和 docx 文件在同一目录下。

![](/img/referer.jpg)

<a name="snipaster"></a>

# 屏幕贴图snipaste

官方主页：https://www.snipaste.com/

日常工作必备工具，无论是办公、学习、聊天，只要你需要参照你复制 (截图) 的图片，都可以将复制的图片贴在频幕上，放大、缩小、编辑。当然，除了它强大的贴图功能，还有截图功能。

例如要比较两个 excel 表格 sheet1、sheet2，sheet1 为参照基准，sheet2 是当前正在编辑的，可以将 sheet1 截图下来，贴在频幕上，这样编辑 sheet2 的同时也能看到 sheet1 的内容。

![](/img/others/733013-20181016090829444-257071711.jpg)

<a name="ditto"></a>

# 复制、粘贴神器Ditto

平时我们是复制一次就粘贴一次，Ctrl+C -> Ctrl+V -> Ctrl+C -> Ctrl+V ->Ctrl+C -> Ctrl+V 。有时候想从同一个复制多次，然后在另一个地方依次粘贴，不用来回复制、粘贴。Ditto 神器能很好地解决这个问题。

使用方法：[Ditto使用方法说明](/others/ditto)

项目主页：https://ditto-cp.sourceforge.io/

![](/img/others/733013-20181016091633685-1765108915.jpg)

<a name="vscodeext"></a>

# vscode指定扩展的安装位置

见：[https://www.junmajinlong.com/others/vscode_ext_position](/others/vscode_ext_position)

<a name="blogvirtualdesktop"></a>

# 虚拟桌面神器 (多桌面)

Win10 自带了虚拟桌面的功能，但是功能并不太好，比如桌面 1 打开了某个应用，在桌面 2 打开这个应用，有可能会自动切换回桌面 1 打开的这个应用。

有很多不错的虚拟桌面工具，但是我用的最强大的虚拟桌面是 dexpot，个人使用的话是免费的。

官方站点：**[https://www.dexpot.de](https://www.dexpot.de)**

下面是我的设置：

![](/img/others/733013-20190124205346555-1068184413.jpg)

![](/img/others/733013-20190124205403012-1894725167.jpg)

![](/img/others/733013-20190124205649923-1605788130.jpg)

![](/img/others/733013-20190124210839022-1808162213.jpg)

<a name="wsl"></a>

# 自定义wsl安装位置以及多wsl共存

见：[https://www.junmajinlong.com/others/custom_wsl_install_location](/others/custom_wsl_install_location)

# Win10替换为Win7画图

曾经Windows的画图是不可替代的草图神器，但已经忘记了是从Win10哪个版本开始，Win10的画图变的非常难用，特别是选择较小的范围时。

所以，想把好用方便的Win7画图替换到Win10上。

方式很简单，从一个Win7(或Win8或早期版本的Win10)虚拟机上找到：

```
C:\Windows\system32\mspaint.exe
C:\Windows\system32\zh-CN\mspaint.exe.mui
```

将它们拷贝出来放在一个目录下(比如a目录)，再创建一个a/zh-CN目录，将mui文件丢进去。所以，最终文件结构：

```
a\
├── mspaint.exe
└── zh-CN
    └── mspaint.exe.mui
```

将目录a拷贝到Win10上，然后将画图快捷方式的路径指向这个mspaint即可。

![](/img/others/1593001563719.png)