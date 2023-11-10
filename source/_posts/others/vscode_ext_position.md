---
title: vscode指定扩展安装位置
p: others/vscode_ext_position.md
date: 2019-07-06 18:20:41
tags: [Others,windows]
categories: [Others,windows]
---

# vscode指定扩展安装位置

默认情况下，(Windows)vscode的安装路径为`C:\Users\用户名\.vscode\extensions`。

如果想要自定义扩展的安装路径，无法直接在vscode中修改。但是，在启动vscode的时候，可以指定扩展路径。

```
CMD$ D:\VSCode\bin\code --help

Usage: code.exe [options] [paths...]

To read output from another program, append '-' (e.g. 'echo Hello World | code -')

Options:
  -d, --diff <file> <file>           Compare two files with each other.
  -a, --add <dir>                    Add folder(s) to the last active window.

Extensions Management:
  --extensions-dir <dir>
      Set the root path for extensions.
  --list-extensions
      List the installed extensions.
```


可以看到，code有个选项`--extensions-dir`，它用来指定扩展安装位置。所以，可以修改vscode的快捷方式，加入code的启动选项。

例如，我想将扩展路径设置为`g:\vscode-extensions`，先创建好这个目录。然后，按下图修改：

![](/img/others/733013-20180725002704483-1271003040.jpg)

然后将原来`C:\Users\用户名\.vscode\extensions`下的扩展剪切到新的扩展路径`g:\vscode-extensions`下。

再用快捷方式打开vscode即可。可能需要重新配置颜色主题，不过这都不是问题。