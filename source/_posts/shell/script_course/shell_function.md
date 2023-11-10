---
title: Shell脚本深入教程：Bash函数
p: shell/script_course/shell_function.md
date: 2020-05-19 14:39:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash函数

在shell中，函数可以被当作命令一样执行，它是命令的组合结构体。

函数的语法结构：

```
[ function ] name () compound-cmd [redirection]
```

![](/img/shell/1589873888364.png) 

例如：定义一个名为rm的函数，该函数会将所有参数指定的文件移动到`~/backup`目录下，目的是替代rm命令，避免误删除的危险操作。

```shell
$ function rm () { [ -d ~/rmbackup ] || mkdir ~/rmbackup;/bin/mv -f $@ ~/rmbackup; } &>/dev/null
```

在调用rm函数时，只需是给rm函数传递参数即可。例如，要移动/tmp/a.log。

```shell
$ rm /tmp/a.log
```

在执行函数时，会将执行可能输出的信息重定向到/dev/null中。

关于函数的几个结论：
- (1).当前shell定义的函数只能在当前shell使用，子shell无法继承父shell的函数定义。除非使用`export -f`将函数导出为全局函数。  
- (2).定义了函数后，可以使用`unset -f`移除当前shell中已定义的函数。  
- (3).除非出现语法错误，或者已经存在一个同名只读函数，否则函数的退出状态码是函数最后执行的一个命令的退出状态码。 
- (4).可使用`typeset -f [func_name]`或`declare -f [func_name]`查看当前shell已定义的函数名和对应的定义语句。使用`typeset -F`或`declare -F`则只显示当前shell中已定义的函数名。  
- (5).使用`type func_name`可查看函数定义内容。  
- (6).函数可以递归，递归层次可以无限。  
- (7).shell函数也接受位置变量`$0、$1、$2...`，但函数的位置参数是调用函数时传递给函数的，而非传递给脚本的参数。所以脚本的位置变量和函数的位置变量是不同的，但是`$0`和脚本的位置变量​`$0`是一致的。另外，函数也接受特殊变量`​$#`，和脚本的`$#`一样，它也表示位置变量的个数。  
- (8).函数体内部可以使用return命令，当函数结构体中执行到return命令时将退出整个函数。return后可以带一个状态码整数，即`return n`，表示函数的退出状态码，不给状态码时，默认状态码为0。  
- (9).函数结构体中可以使用local命令定义本地变量，例如`local i=3`。本地变量只在函数内部(包括子函数)可见，函数外不可见。  
- (10).只有先定义了函数，才可以调用函数。不允许函数调用语句在函数定义语句之前。  

为了让函数在子shell(例如脚本)中也可以使用，使用`export -f`将其导出为全局函数。取消函数的导出则使用`export -n`。

```shell
export -f rm
export -n rm
```

另外需要注意的是，函数支持无限递归。这可能在不经意间出错，导致崩溃。例如，写一个名为"ls"的函数。

```shell
function ls() { ls -l; }
```

这时执行ls命令会卡住，和想象中的`ls -l`效果完全不同，因为函数体中的ls也递归成了函数，这将无限递归下去。