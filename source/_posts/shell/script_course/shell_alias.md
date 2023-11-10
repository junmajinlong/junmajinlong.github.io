---
title: Shell脚本深入教程：bash命令别名：alias深入理解
p: shell/script_course/shell_alias.md
date: 2023-07-12 14:17:00
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

--------

# bash命令别名：alias深入理解

可以在bash中为命令定义别名。例如：

```bash
# 下次执行ls的时候，就会默认显示所有隐藏文件
alias ls="ls -A"
```

通常默认情况下就定义了一些别名(但不一定)。可通过alias命令查看当前有效的别名。例如，我定义了很多别名：

```bash
$ alias
alias cls='clear'
alias code='/mnt/v/VSCode-portable/bin/code'
alias curl_http='curl -C - -# --proxy http://127.0.0.1:8118 -O'
alias daten='date +"%F %T.%N"'
alias dates='date -d "" +"%s"'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
```

可通过`unalias`取消别名的定义。

```bash
unalias ls
```

alias命令是临时定义别名，要定义长久生效的别名就将别名定义语句写入`/etc/profile`或`~/.bash_profile`或`~/.bashrc`，第一个对所有用户有效，后面两个对对应用户有效。修改后记得使用source来重新调取这些配置文件。

另外需要说明的是，当别名和命令同名时，将优先执行别名(否则别名就没有意义了)，这可以从which的结果中看出：

```bash
[root@xuexi ~]# which mv
alias mv='mv -i'
        /bin/mv
```

如果定义的命名名称和原始命令同名(例如定义的别名`ls='ls -l'`)，此时如果想要明确使用原始命令，可以`unalias`别名或者使用绝对路径或者使用转义符(例如`\ls`)来还原命令。

## alias的缺陷

alias的定义和使用起来有点模糊，以下面这个命令别名为例，在有的shell脚本的书籍上使用了这样的定义，但却是错误的，原因稍后说明。

```bash
alias rmm='cp $@ ~/backup;rm $@'
```

该别名的目的是删除文件时先备份到一个目录下，然后再删除。**按照man bash里的说明，别名rmm只是第一个cp命令的别名，分号后的rm不是别名的一部分，而是紧跟在别名后的下一行命令。当执行别名rmm时，首先读取别名到分号位置处，然后进行别名扩展，执行完别名命令后，再执行分号后的rm命令。**

之所以说上面的命令是错误的命令，问题出在cp的参数`$@`，该变量本表示提供的所有参数，但由于cp命令后使用分号分隔并定义了另一个命令，这使得执行别名命令时，参数无法传递到cp命令上，而只能传递到最后一个命令rm上，也就是说cp后的`$@`是空值。所以该别名等价于：

```bash
alias rmm='cp ~/backup;rm $@'
```

是否真的如此，使用echo测试一番即可。

```bash
[root@xuexi ~]# alias rmm='echo cp $@ ~/backup;echo rm $@'
[root@xuexi ~]# rmm /etc/fstab /etc/hosts
cp /root/backup
rm /etc/fstab /etc/hosts
```

从上面的结果中看到cp后的`$@`根本就没有进行扩展，而是空值。

那如果别名定义语句中没有使用分号或其他方法定义额外的命令，而是只有一个命令呢？别名一定就能正确工作吗？非也。以下面的例子为例：

```bash
[root@xuexi ~]# alias rmm='echo mv -f $@ ~/backup'

[root@xuexi ~]# rmm /etc/fstab /etc/hosts
mv -f /root/backup /etc/fstab /etc/hosts
```

发现问题了吗？`$@`是扩展在`~/backup`目录之后的，也就是说下面mv的别名想要替代rm，是无法正常工作的：

```bash
alias rm='mv -f $@ ~/backup'
```

之所以无法正常工作，是因为`~/backup`也是`$@`的一部分，且是`$@`中最前面的参数。执行下面的命令就知道了：

```bash
[root@xuexi ~]# echo mv -f "$@" ~/backup /etc/fstab /etc/hosts
mv -f /root/backup /etc/fstab /etc/hosts
```

从上面的分析可以知道，alias是有其缺陷的，它只适合进行简单的命令和参数替换、补全，想要实现复杂的命令替代有点难度。因此man bash中建议尽量使用函数来取代别名(For almost every purpose, aliases are superseded by shell functions)。

## 别名的最佳实现

毫无疑问，写个shell脚本比别名安全、完整多了，这是替代别名的一种方法。而我个人的建议是，在别名的定义语句中使用函数来克服别名的缺陷。

例如，为了让rm安全执行，使用以下两种方法定义别名：

```bash
alias rm='copy1(){ /bin/cp -a $@ ~/backup;rm $@; };copy1 $@'
alias rm='move1(){ /bin/mv -f $@ ~/backup; };move1 $@'
```

因为执行别名时的参数只能传递给最后一个命令即copy1或move1函数，但`$@`代表的参数可以传递给函数，让函数中的`$@`得到正确的扩展，于是整个别名都能合理且正确地执行。

或者直接定义一个shell function替代rm。例如向`/etc/profile.d/rm.sh`文件中写入：

```bash
function rm(){ [ -d ~/rmbackup ] || mkdir ~/rmbackup;/bin/mv -f $@ ~/rmbackup; }
chmod +x /etc/profile.d/rm.sh
source /etc/profile.d/rm.sh
```

如此，执行rm命令时，便会执行此处定义的rm函数，使得rm变得更安全。但注意，这样的函数默认无法直接在脚本中使用，除非使用`export -f function_name`导出函数，使其可以被子shell继承。所以，可在`/etc/profile.d/rm.sh`文件的尾部加上导出语句：

```bash
function rm(){ [ -d ~/rmbackup ] || mkdir ~/rmbackup;/bin/mv -f $@ ~/rmbackup; }
export -f rm
```

如果function名和命令名相同，则默认优先执行function，除非使用command明确指定。例如上面定义了rm函数，如果想执行rm命令，除了使用/bin/rm，还可以如下操作：

```bash
command rm a.txt
```

如果是**在shell脚本里涉及到rm命令，那么更建议在每次rm之前先cd到那个目录下，然后再rm相对路径，这样至少能保证不出现符号"/"**。当然，更重要的是脚本习惯一些编写脚本的规范，印在骨子里那种，就算想出问题也难。