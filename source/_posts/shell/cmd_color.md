---
title: 修改shell命令提示符和命令的输入颜色
p: shell/cmd_color.md
date: 2019-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

## 修改命令提示符颜色

修改命令提示符的话，只需修改PS1环境变量即可。

```shell
PS1='\[\033[01;31m\][\u@\h \W]$ \[\033[00m\]'
```

效果如图：

![](/img/shell/1569036664433.jpg)

## 修改命令输入的颜色

修改命令输入的颜色，思路是不关闭PS1的颜色，然后在每次敲下回车键执行命令的时候自动插入颜色终止符。这需要借助trap捕获DEBUG信号来实现。

```shell
PS1='\[\033[01;31m\][\u]$ \[\033[1;30m\]'
trap 'echo -ne "\e[0m"' DEBUG
```

![](/img/shell/1569036796135.jpg)


如果要写入shell配置文件，建议写到`~/.bash_profile`，而不要写入`~/.bashrc`，否则借助ssh类的工具都将因为trap DEBUG信号的特殊性而无限等待，比如scp/rsync等。或者，直接判断是否是交互式登录，是的话就设置，否则不设置：
```
if [ "${-#*i}" != "$-" ];then
    # interactively shell
    PS1='\[\033[01;31m\][\u@\h \W]$ \[\033[1;30m\]'
    trap 'echo -ne "\e[0m"' DEBUG
fi
```
