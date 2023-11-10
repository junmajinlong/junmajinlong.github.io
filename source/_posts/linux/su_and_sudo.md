---
title: su和sudo
p: linux/su_and_sudo.md
date: 2019-07-06 18:20:42
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------


# su

切换用户或以指定用户运行命令。

使用su可以指定运行命令的身份(user/group/uid/gid)。

为了向后兼容，su默认不会改变当前目录，且仅设置HOME和SHELL这两个环境变量(若目标用户非root，则还设置USER和LOGNAME环境变量)。推荐使用--login选项(即"-"选项)避免环境变量混乱。

```
su [options...] [-] [user [args...]]
选项说明：
-c command：使用-c指定要在shell执行的命令，会为每个su都分配新的会话环境
-, -l, --login：启动shell作为登录的shell，模拟真正的登录环境。它会做下面几件事：
       1.清除除了TERM外的所有环境变量
       2.初始化HOME,SHELL,USER,LOGNAME,PATH环境变量
       3.进入目标用户的家目录
       4.设置argv[0]为"-"以便设置shell作为登录的shell
       使用--login的su是交互式登录。不使用--login的su是非交互式登录(除不带任何参数的su外)
-m, -p, --preserve-environment：
       保留整个环境变量(不会重新设置HOME,SHELL,USER和LOGNAME)，
       保留环境的方法是新用户shell上执行原用户的各配置文件，
       如~/.bashrc当设置了--login时，将忽略该选项
-s SHELL：运行指定的shell而非默认shell，选择shell的顺序优先级如下：
       1.--shell指定的shell
       2.如果使用了--preserve-environment，选择SHELL环境变量的shell
       3.选项目标用户在passwd文件中指定的shell
       4./bin/sh
```

注意：

(1). 若su没有给定任何参数，将默认以root身份运行交互式的shell(交互式，所以需要输入密码)，即切换到root用户，但只改变HOME和SHELL环境变量。  
(2). `su - username`是交互式登录，要求密码，会重置整个环境变量，它实际上是在模拟真实登录环境。  
(3). `su username`是非交互登录，不会重置除HOME/SHELL外的环境变量。  

例如：用户wangwu家目录为/home/wangwu，其shell为/bin/csh。

```
$ head -1 /etc/passwd ; tail -1 /etc/passwd
root:x:0:0:root:/root:/bin/bash
wangwu:x:2002:2002::/home/wangwu:/bin/csh
```

首先su到wangwu上，再执行一个完全不带参数的su。

```
# 使用su - username后，以登录shell的方式模拟登录，会重新设置各环境变量。su - username是交互式登录
$ su - wangwu        
$ env | egrep -i '^home|^shell|^path|^logname|^user'
HOME=/home/wangwu
SHELL=/bin/csh
USER=wangwu
LOGNAME=wangwu
PATH=/usr/lib64/qt-3.3/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin
PWD=/home/wangwu

# 不带任何参数的su，是交互式登录切换回root，但只会改变HOME和SHELL环境变量
$ su
$ env | egrep -i '^home|^shell|^path|^logname|^user|^pwd'
SHELL=/bin/bash
USER=wangwu
PATH=/usr/lib64/qt-3.3/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin
PWD=/home/wangwu
HOME=/root
LOGNAME=wangwu

#  su - 的方式切换回root
$ su  -
Password:
$ env | egrep -i '^home|^shell|^path|^logname|^user|^pwd'
SHELL=/bin/bash
USER=root
PATH=/usr/lib64/qt-3.3/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
PWD=/root
HOME=/root
LOGNAME=root

# 再直接su username，它只会重置SHELL和HOME两个环境变量，其他环境变量保持不变
$ su wangwu
$ env | egrep -i '^home|^shell|^path|^logname|^user|^pwd'
SHELL=/bin/csh
USER=wangwu
PATH=/usr/lib64/qt-3.3/bin:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
PWD=/root
HOME=/home/wangwu
LOGNAME=wangwu
```

在某些环境下或脚本中，可能需要临时切换身份执行命令，注意这时候的环境变量是否会改变，否则很可能报错提示命令找不到。

![](/img/referer.jpg)

# sudo

sudo可以让一个用户以某个身份(如root或其他用户)执行某些命令，它隐含的执行方式是切换到指定用户再执行命令，因为涉及到了用户的切换，所以环境变量是否重置是需要设置的。

sudo支持插件实现安全策略。默认的安全策略插件是sudoers，它是通过/etc/sudoers或LDAP来配置的。

安全策略是控制用户使用sudo命令时具有什么权限，但要注意，安全策略可能需要用户进行身份认证，如密码认证的机制或其他认证机制，如果开启了认证要求，则在指定时间内未完成认证时sudo会退出，默认超时时间为5分钟。

安全策略支持对认证进行缓存，使得在一定时间内该用户无需再次认证就可以执行sudo命令，默认缓存时间为5分钟，sudo -v可以更新认证缓存。

sudo支持日志审核，可以记录下成功或失败的sudo。

![](/img/referer.jpg)

## /etc/sudoers文件

该文件里主要配置sudo命令时指定的用户和对应的权限。

```
$ visudo     # 以下选取的是部分行
## hostname or IP addresses instead.   # 主机别名Host_Alias
# Host_Alias     FILESERVERS = fs1, fs2
# Host_Alias     MAILSERVERS = smtp, smtp2

## User Aliases           # 用户别名User_Alias
# User_Alias ADMINS = jsmith, mikem

## Command Aliases        # 命令别名
# Cmnd_Alias SERVICES = /sbin/service, /sbin/chkconfig
# Cmnd_Alias LOCATE = /usr/bin/updatedb
 
root    ALL=(ALL)       ALL   # sudo权限的配置
# %sys ALL = NETWORKING, SOFTWARE, SERVICES, STORAGE, DELEGATING, PROCESSES, LOCATE, DRIVERS
## Allows people in group wheel to run all commands
# %wheel        ALL=(ALL)       ALL
## Same thing without a password
# %wheel        ALL=(ALL)       NOPASSWD: ALL
```

在这个文件里，主要有别名(用户别名，主机别名，命令别名)的配置和sudo权限的配置。

安全策略配置格式为：

> 用户名  主机名=(可切换到的用户身份)  权限和命令
>
> ①      ②         ③             ④

①用户名：可以用组，只需在组名前加个百分号%表示。

②主机名：表示该用户可以在哪些主机上运行sudo，可以用hostname也可以用ip指定。

③可切换的用户身份，即指定执行命令的用户，也可以用组。

④权限和命令：允许执行和不允许执行的命令(多个命令间用逗号分隔)和特殊权限，命令可以带其选项及参数。命令要写绝对路径。不允许执行的命令需要在命令前加上"!"来表示。可以使用标签，如NOPASSWD标签表示切换或以指定用户执行该标签后的命令时不需要输入密码。一行写不下时可使用`\`续行。

标签使用方法：

```
NOPASSWD:/usr/sbin/useradd,PASSWD:/usr/sbin/userdel
```

它表示useradd命令不需要输入密码，而userdel需要输入密码。

对于别名，相当于用户对于用户组。权限配置处都可以使用别名，即①②③④处都能使用别名来配置。

例如，主机别名里设置多个主机，以后在②位置处直接使用主机别名。

```
FILESERVERS = fs1, fs2
```

以下是某设置示例：

```
DEFAULT=/bin/*,\
        /usr/bin/*,\
        /bin/su - [!-]*,\
        !/bin/su - root,\
        !/bin/su root, \
        /sbin/ldconfig,\
        /sbin/ifconfig,\
        /sbin/service,\
        /sbin/chkconfig,\
        /usr/sbin/useradd,\
        /usr/sbin/userdel,\
        /usr/sbin/dmidecode, \
        /usr/sbin/lsof, \
        /usr/bin/passwd [!-]*,\
        !/usr/bin/passwd root,\
        sudoedit /etc/rc.local,\
        sudoedit /etc/hosts,\
        sudoedit /etc/ld.so.conf,\
        sudoedit /etc/exports,\
        sudoedit /etc/profile,\
        sudoedit /etc/bashrc,\
        sudoedit /etc/security/limits.conf,\
        /etc/init.d/*

ABC ALL=(ALL)NOPASSWD:DEFAULT
```

其中上面的`/usr/bin/passwd [!-]*`表示允许修改加参数的密码。`/bin/su - [!-]*`表示允许"su -"到某用户下，但必须给参数。


## sudo和sudoedit命令

当sudo执行指定的command时，它会调用fork函数，并设置命令的执行环境(如某些环境变量)，然后在子进程中执行command，sudo的主进程等待命令执行完毕，然后传递命令的退出状态码给安全策略并退出。

sudoedit等价于sudo -e，它是以sudo的方式执行文件编辑动作。

```
sudo [options] [command]
选项说明：
-b             ：(background)该选项告诉sudo在后台执行指定的命令。
                 注意，如果使用该选项，将无法使用任务计划(job)来控制维护这些后台进程，
                 需要交互的命令应该考虑是否真的要后台，因为可能会失败
-l[l] [command]：当单独使用-l选项时，将列出(list)用户可执行和被禁止的命令。
                 当配合command时，且该command是被允许执行的命令，将列出命令的全路径及该命令参数。
                 如果command是不被允许执行的，则sudo直接以状态码-1退出。
                 可以指定多个字母"l"来显示更详细的格式
-n             ：使得sudo变成非交互模式，但如果安全策略是要求输入密码的，则sudo将报错
-S             ：(stdin)该选项使得sudo从标准输入而非终端设备上读取密码，给定的密码必须在尾部加换行符
-s [command]   ：(shell)指定要切换到的shell，如果给定command，则在此shell上执行该命令
-U user        ：(other user)配合-l选项来指定要列出哪个用户的权限信息
-u user        ：(user)该选项明确指定要以此处指定的用户而非root来运行command。
                 若使用uid的方式指定用户，则需要使用"#uid"，但很多时候可能需要
                 对"#"使用"\"转义，即使用"\#uid"
-E             ：(environment)该选项告诉sudo在执行命令时保留自己的环境变量，
                 保留环境变量的方式是执行环境配置文件。但因为跨了用户，所以很可
                 能某些家目录下的环境配置文件会因为无权限而执行失败，此时sudo将报错   
-k [command]   ：当单独使用-k选项时，sudo将使得用户的认证缓存失效。下次执行sudo命令需要输入密码。
                 当配合command时，-k选项将忽略用户的缓存，所以sudo将要求用户输入密码，但这次输
                 入密码不会更新认证缓存但执行-k选项本身，不需要密码
-K             ：(sure kill)类似于-k选项，但它会完全移除用户的认证缓存，且不会配合command，
                 执行-K本身不需要密码
-v             ：(validate)该选项使得sudo更新用户认证缓存
--             ：暗示sudo命令行参数到此结束
```

在sudo上可以直接设置环境变量，它会传递为command的环境。设置的方式为var=value，如`LD_LIBRARY_PATH=/usr/local/pkg/lib`

由于sudo默认的安全策略插件是sudoers，所以当用户执行sudo时，系统会自动去寻找/etc/sudoers文件(该文件里被root配置了用户对应的权限，也即安全策略)，查看sudo要使用的用户是否有对应的权限，如果有则执行，如果没有权限就失败退出sudo。