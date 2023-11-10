---
title: Shell脚本深入教程：Bash启动时加载配置文件的顺序和过程
p: shell/script_course/bash_startup.md
date: 2023-07-11 14:17:00
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

--------

# Bash启动时加载配置文件的顺序和过程

当执行bash命令或者用户登录系统时，会加载各种bash配置文件，还会设置或清空一系列变量，有时还会执行一些自定义的命令。这些行为都算是启动bash时的过程。

另外，有些时候登录系统是可以交互的(如正常登录系统)，有些时候是无交互的(如执行一个脚本)，因此总的来说bash启动类型可分为交互式shell和非交互式shell。更细分一层，交互式shell还分为交互式的登录shell和交互式非登录shell，非交互的shell在某些时候可以在bash命令后带上`--login`或短选项`-l`，这时也算是登录式，即非交互的登录式shell。

## 判断是否交互式、是否登录式

判断是否为交互式shell有两种简单的方法：

**方法一：判断变量"-"，如果值中含有字母"i"，表示交互式。**

```bash
[root@xuexi ~]# echo $-
himBH

[root@xuexi ~]# vim a.sh
#!/bin/bash
echo $-

[root@xuexi ~]# bash a.sh
hB
```

**方法二：判断变量PS1，如果值非空，则为交互式，否则为非交互式，因为非交互式会清空该变量。**

```bash
[root@xuexi ~]# echo $PS1
[\u@\h \W]\$
```

**判断是否为登录式的方法也很简单，只需执行`shopt login_shell`即可。值为"on"表示为登录式，否则为非登录式。**

```bash
[root@xuexi ~]# shopt login_shell  
login_shell     on
[root@xuexi ~]# bash

[root@xuexi ~]# shopt login_shell
login_shell     off
```

所以，要判断是交互式以及登录式的情况，可简单使用如下命令：

```bash
echo $PS1;shopt login_shell
# 或者
echo $-;shopt login_shell
```

## 几种常见的bash启动方式

**(1).正常登录(伪终端登录如ssh登录，或虚拟终端登录)时，为交互式登录shell。**

```bash
[root@xuexi ~]# echo $PS1;shopt login_shell 
[\u@\h \W]\$
login_shell     on
```

**(2).su命令，不带`--login`时为交互式、非登录式shell，带有`--login`时，为交互式、登录式shell。**

```bash
[root@xuexi ~]# su root

[root@xuexi ~]# echo $PS1;shopt login_shell 
[\u@\h \W]\$
login_shell     off
[root@xuexi ~]# su -
Last login: Sat Aug 19 13:24:11 CST 2017 on pts/0

[root@xuexi ~]# echo $PS1;shopt login_shell
[\u@\h \W]\$
login_shell     on
```

**(3).执行不带`--login`选项的bash命令时为交互式、非登录式shell。但指定`--login`时，为交互式、登录式shell。**

```bash
[root@xuexi ~]# bash

[root@xuexi ~]# echo $PS1;shopt login_shell
[\u@\h \W]\$
login_shell     off
[root@xuexi ~]# bash -l

[root@xuexi ~]# echo $PS1;shopt login_shell
[\u@\h \W]\$
login_shell     on
```

**(4).使用命令组合(使用括号包围命令列表)以及命令替换进入子shell时，继承父shell的交互和登录属性。**

```bash
[root@xuexi ~]# (echo $BASH_SUBSHELL;echo $PS1;shopt login_shell)
1
[\u@\h \W]\$
login_shell     on
[root@xuexi ~]# su

[root@xuexi ~]# (echo $BASH_SUBSHELL;echo $PS1;shopt login_shell)
1
[\u@\h \W]\$
login_shell     off
```

**(5).ssh执行远程命令，但不登录时，为非交互、非登录式。**

```bash
[root@xuexi ~]# ssh localhost 'echo $PS1;shopt login_shell'
 
login_shell     off
```

**(6).执行shell脚本时，为非交互、非登录式shell。但指定了`--login`时，将为非交互、登录式shell。**

例如，脚本内容如下：

```bash
[root@xuexi ~]# vim b.sh
#!/bin/bash
echo $PS1
shopt login_shell
```

不带`--login`选项时，为非交互、非登录式shell。

```bash
[root@xuexi ~]# bash b.sh
 
login_shell     off
```

带`--login`选项时，为非交互、登录式shell。

```bash
[root@xuexi ~]# bash -l b.sh
 
login_shell     on
```

**(7).在图形界面下打开终端时，为交互式、非登录式shell。**

![](/img/shell/733013-20170823123801246-458943313-1689245232634.png)

但可以设置为使用交互式、登录式shell。

![](/img/shell/733013-20170823123807871-1777221991-1689245260531.png)

## 加载bash环境配置文件

无论是否交互、是否登录，bash总要配置其运行环境。bash环境配置主要通过加载bash环境配置文件来完成。但是否交互、是否登录将会影响加载哪些配置文件，除了交互、登录属性，有些特殊的属性也会影响读取配置文件的方法。

bash环境配置文件主要有：

- /etc/profile
- ~/.bash_profile
- ~/.bashrc
- /etc/bashrc
- /etc/profile.d/*.sh

为了测试各种情形读取哪些配置文件，先分别向这几个配置文件中写入几个echo语句，用以判断该配置文件是否在启动bash时被读取加载了。

```bash
echo "echo '/etc/profile goes'" >>/etc/profile
echo "echo '~/.bash_profile goes'" >>~/.bash_profile
echo "echo '~/.bashrc goes'" >>~/.bashrc
echo "echo '/etc/bashrc goes'" >>/etc/bashrc
echo "echo '/etc/profile.d/test.sh goes'" >>/etc/profile.d/test.sh
chmod +x /etc/profile.d/test.sh
```

**①.交互式登录shell或非交互式但带有`--login`(或短选项`-l`，例如在shell脚本中指定`#!/bin/bash -l`时)的bash启动时，将先读取`/etc/profile`，再依次搜索`~/.bash_profile ~/.bash_login ~/.profile`，并仅加载第一个搜索到且可读的文件。当退出时，将执行`~/.bash_logout`中的命令。**

但要注意，在`/etc/profile`中有一条加载`/etc/profile.d/*.sh`的语句，它会使用source加载`/etc/profile.d/`下所有可执行的sh后缀的脚本。

```bash
[root@xuexi ~]# grep -A 8 \*\.sh /etc/profile  
for i in /etc/profile.d/*.sh ; do
    if [ -r "$i" ]; then
        if [ "${-#*i}" != "$-" ]; then
            . "$i"
        else
            . "$i" >/dev/null 2>&1
        fi
    fi
done
```

内层if语句中的`"${-#*i}" != "$-"`表示将`$-`从左向右模式匹配`*i`并将匹配到的内容删除(即进行变量切分)，如果`$-`切分后的值不等于`$-`，则意味着是交互式shell，于是怎样怎样，否则怎样怎样。

同样的，在`~/.bash_profile`中也一样有加载`~/.bashrc`的命令。

```bash
[root@xuexi ~]# grep -A 1 \~/\.bashrc ~/.bash_profile
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi
```

而`~/.bashrc中`又有加载`/etc/bashrc`的命令。

```bash
[root@xuexi ~]# grep -A 1 /etc/bashrc ~/.bashrc
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi
```

其实`/etc/bashrc`中还有加载`/etc/profile.d/*.sh`的语句，但前提是非登录式shell时才会执行。以下是部分语句：

```bash
if ! shopt -q login_shell ; then   # We're not a login shell
...
    for i in /etc/profile.d/*.sh; do
        if [ -r "$i" ]; then
            if [ "$PS1" ]; then
                . "$i"
            else
                . "$i" >/dev/null 2>&1
            fi
        fi
    done
...
fi
```

从内层if语句和`/etc/profile`中对应的判断语句的作用是一致的，只不过判断方式不同，写法不同。

因此，交互式的登录shell加载bash环境配置文件的实际过程如下图：

![](/img/shell/1689245446283.png)

以下结果验证了结论：

```bash
Last login: Mon Aug 14 04:49:29 2017     # 新开终端登录时
/etc/profile.d/*.sh goes
/etc/profile goes
/etc/bashrc goes
~/.bashrc goes
~/.bash_profile goes

[root@xuexi ~]# ssh localhost        # ssh远程登录时
root@localhost's password:
Last login: Mon Aug 14 05:05:50 2017 from 172.16.10.1
/etc/profile.d/*.sh goes
/etc/profile goes
/etc/bashrc goes
~/.bashrc goes
~/.bash_profile goes

[root@xuexi ~]# bash -l        # 执行带有"--login"选项的login时
/etc/profile.d/*.sh goes
/etc/profile goes
/etc/bashrc goes
~/.bashrc goes
~/.bash_profile goes

[root@xuexi ~]# su -          # su带上"--login"时
/etc/profile.d/*.sh goes
/etc/profile goes
/etc/bashrc goes
~/.bashrc goes
~/.bash_profile goes

[root@xuexi ~]# vim a.sh    # 执行shell脚本时带有"--login"时
#!/bin/bash -l
echo haha

[root@xuexi ~]# ./a.sh 
/etc/profile goes
/etc/bashrc goes
~/.bashrc goes
~/.bash_profile goes
haha
```

之所以执行shell脚本时没有显示执行`/etc/profile.d/*.sh`，是因为它是非交互式的，根据`/etc/profile`中的`if [ "${-#*i}" != "$-" ]`判断，它将会把`/etc/profile.d/*.sh`的执行结果重定向到`/dev/null`中。也就是说，即使是shell脚本(带`--login`选项)，它也加载了所有bash环境配置文件。

**②.交互式非登录shell的bash启动时，将读取`~/.bashrc`，不会读取`/etc/profile ~/.bash_profile ~/.bash_login ~/.profile`。**

因此，交互式非登录shell加载bash环境配置文件的实际过程为下图内方框中所示：

![](/img/shell/1689245612964.png)

例如，执行不带`--login`的bash命令或su命令时。

```bash
[root@xuexi ~]# bash
/etc/profile.d/*.sh goes
/etc/bashrc goes
~/.bashrc goes

[root@xuexi ~]# su
/etc/profile.d/*.sh goes
/etc/bashrc goes
~/.bashrc goes
```

**③.非交互式、非登录式shell启动bash时，不会加载前面所说的任何bash环境配置文件，但会搜索变量`BASH_ENV`，如果搜索到了，则加载其所指定的文件。但并非所有非交互式、非登录式shell启动时都会如此，见情况④。**

它就像是这样的语句：

```bash
if [ -n "$BASH_ENV" ];then
    . "$BASH_ENV"
fi
```

**几乎执行所有的shell脚本都不会特意带上`--login`选项，因此shell脚本不会加载任何bash环境配置文件，除非手动配置了变量`BASH_ENV`。**

**④.远程shell方式启动的bash，它虽然属于非交互、非登录式，但会加载`~/.bashrc`，所以还会加载`/etc/bashrc`，由于是非登录式，所以最终还会加载`/etc/profile.d/*.sh`，只不过因为是非交互式而使得执行的结果全部重定向到了`/dev/null`中。**

如果了解rsync，就知道它有一种远程shell连接方式。所谓的远程shell方式，是指通过网络的方式启动bash并将bash的标准输出关联起来，就像它连接了一个远程的shell守护进程一样。一般由sshd实现这样的连接方式，老版的rshd也一样支持。

事实也确实如此，使用ssh连接但不登录远程主机时(例如只为了执行远程命令)，就是远程shell的方式，但它却是非交互、非登录式的shell。

```bash
[root@xuexi ~]# ssh localhost echo haha
root@localhost's password:
/etc/bashrc goes
~/.bashrc goes
haha
```

正如上文所说，它同样加载了`/etc/profile.d/*.sh`，只不过`/etc/bashrc`中的if判断语句`if [ "$PS1" ]; then`使得非交互式的shell要将执行结果重定向到`/dev/null`中。