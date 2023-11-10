---
title: SysV /etc/rc.d/init.d/functions脚本源码分析
p: shell/functions.md
date: 2021-03-13 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# SysV /etc/rc.d/init.d/functions脚本源码分析

虽然现在SysV风格的服务启动脚本在主流的操作系统上已经被systemd替代而逐渐退出舞台，但是对于学习写健壮的服务管理脚本以及学习写脚本来说，学习/etc/rc.d/init.d/functions还是很有帮助的。

/etc/rc.d/init.d/functions几乎被/etc/rc.d/init.d/下所有的Sysv服务启动脚本加载，也是学习shell脚本时一个非常不错的材料，在其中使用了不少技巧。

在该文件中提供了几个有用的函数：

- `daemon`：启动一个服务程序。启动前还检查进程是否已在运行。  
- `killproc`：杀掉给定的服务进程。  
- `status`：检查给定进程的运行状态。  
- `success`：显示绿色的"OK"，表示成功。![](E:\onedrive\docs\blog_imgs\733013-20170913223825907-224821005.png)  
- `failure`：显示红色的"FAILED"，表示失败。![](E:\onedrive\docs\blog_imgs\733013-20170913223834641-1191775278.png)  
- `passed`：显示绿色的"PASSED"，表示pass该任务。![](E:\onedrive\docs\blog_imgs\733013-20170913223842610-698811737.png)  
- `warning`：显示绿色的"warning"，表示警告。![](E:\onedrive\docs\blog_imgs\733013-20170913223849328-632306163.png)  
- `action`：根据进程退出状态码自行判断是执行success还是failure。  
- `confirm`：提示`(Y)es/(N)o/(C)ontinue? [Y]`并判断、传递输入的值。  
- `is_true`：`$1`的布尔值代表为真时，返回状态码0，否则返回1。包括t、y、yes和true，不区分大小写。  
- `is_false`：`$1`的布尔值代表为假时，返回状态码0。否则返回1。包括f、n、no和false，不区分大小写。  
- `checkpid`：检查/proc下是否有给定pid对应的目录。给定多个pid时，只要存在一个目录都返回状态码0。  
- `__pids_var_run`：检查pid是否存在，并保存到变量pid中，同时返回几种进程状态码。是functions中重要函数之一。  
- `__pids_pidof`：获取进程pid。  
- `pidfileofproc`：获取进程的pid。但只能获取/var/run下的pid文件中的值。  
- `pidofproc`：获取进程的pid。可获取任意给定pidfile或默认/var/run下pidfile中的值。  

前三个是functions文件最重要的3个函数，还用到了一些额外的辅助函数，稍稍有点复杂。所以由简至繁，先介绍并展示后面几个函数，再回头解释前3个函数。

以下是/etc/init.d/functions文件的开头定义的语句。设置umask值，使得加载该文件的脚本所在shell的umask为22。导出路径变量。但说实话，这个导出的路径变量并不理想，因为要为非rpm包安装的程序设计服务启动脚本时，必须写全路径命令，例如/usr/local/mysql/bin/mysql。因此，可以考虑将/etc/init.d/functions中的语句注释掉。

```bash
umask 022

# Set up a default search path.
PATH="/sbin:/usr/sbin:/bin:/usr/bin"
export PATH
```

PS：本文分析的/etc/init.d/functions文件是CentOS 7上的，和CentOS 6有些许区别，但该有的目的和动作都有。

<a name="blog1"></a>
## 1.几个显示函数

包括echo\_success、success、echo\_failure、failure、echo\_passed、passed、echo\_warning和warning函数。这几个函数的定义方式和使用方法完全一样。

以下是echo\_success和success函数的定义语句。

```bash
echo_success() {
  [ "$BOOTUP" = "color" ] && $MOVE_TO_COL
  echo -n "["
  [ "$BOOTUP" = "color" ] && $SETCOLOR_SUCCESS
  echo -n $"  OK  "
  [ "$BOOTUP" = "color" ] && $SETCOLOR_NORMAL
  echo -n "]"
  echo -ne "\r"
  return 0
}

success() {
  [ "$BOOTUP" != "verbose" -a -z "${LSB:-}" ] && echo_success
  return 0
}
```

很简单，就是不换行带颜色输出`[  OK  ]`字样。

```bash
[root@xuexi ~]# . /etc/init.d/functions

[root@xuexi ~]# success
[root@xuexi ~]#                                            [  OK  ]
[root@xuexi ~]# echo_success
[root@xuexi ~]#                                            [  OK  ]
```

同理，剩余的几个状态显示函数也一样。

```bash
[root@xuexi ~]# echo_failure
[root@xuexi ~]#                                            [FAILED]
[root@xuexi ~]# failure
[root@xuexi ~]#                                            [FAILED]
```

<a name="blog2"></a>
## 2.action函数

这个函数在写脚本时还比较有用，可以根据退出状态码自动判断是执行success还是执行failure函数。

action函数定义语句如下：

```bash
action() {
  local STRING rc

  STRING=$1
  echo -n "$STRING "
  shift
  "$@" && success $"$STRING" || failure $"$STRING"
  rc=$?
  echo
  return $rc
}
```

这个函数定义的很有技巧。先将第一个参数保存并踢掉，再执行后面的命令(`$@`表示执行后面的命令)。所以，当action函数只有一个参数时，action直接返回OK，状态码为0，当超过一个参数时，第一个参数先被打印，再执行从第二个参数开始的命令。

例如：

```bash
[root@xuexi ~]# action
                                                           [  OK  ]
[root@xuexi ~]# action 5
5                                                          [  OK  ]
[root@xuexi ~]# action sleeping sleep 3
sleeping                                                   [  OK  ]
[root@xuexi ~]# action "moving file" mv xxxxxx.sh aaaaa.sh
moving file mv: cannot stat ‘xxxxxx.sh’: No such file or directory
                                                           [FAILED]
```

所以，在脚本中使用action函数时，可以让命令执行成功与否的判断显得更"专业"。算是一个比较有趣的函数。

通常，该函数会结合/bin/true和/bin/false命令使用，它们无条件返回0或1状态码。

```bash
action $"MESSAGES: " /bin/true
action $"MESSAGES: " /bin/false
```

例如，mysqld启动脚本中，判断mysqld已在运行时，直接输出启动ok的消息。(但实际上根本没做任何事)

```bash
if [ $MYSQLDRUNNING = 1 ] && [ $? = 0 ]; then
    # already running, do nothing
    action $"Starting $prog: " /bin/true
    ret=0
```

<a name="blog3"></a>
## 3.is\_true和is\_false函数

这两个函数的作用是转换输入的布尔值为状态码。

```bash
is_true() {
    case "$1" in
    [tT] | [yY] | [yY][eE][sS] | [tT][rR][uU][eE])
    return 0
    ;;
    esac
    return 1
}

is_false() {
    case "$1" in
    [fF] | [nN] | [nN][oO] | [fF][aA][lL][sS][eE])
    return 0
    ;;
    esac
    return 1
}
```

当is\_true函数的第一个参数(后面的参数会忽略掉)为忽略大小写的t、y、yes或true时，返回状态码0，否则返回1。  
当is\_false函数的第一个参数(后面的参数会忽略掉)为忽略大小写的f、n、no或false时，返回状态码0，否则返回1。

<a name="blog4"></a>
## 4.confirm函数

这个函数一般用不上，因为脚本本来就是为了避免交互式的。在CentOS 7的functions中已经删除了该函数定义语句。不过，借鉴下它的处理方法还是不错的。

以下摘自CentOS 6.6的/etc/init.d/functions文件。

```bash
# returns OK if $1 contains $2
strstr() {
  [ "${1#*$2*}" = "$1" ] && return 1   # 参数$1中不包含$2时，返回1，否则返回0
  return 0
}

# Confirm whether we really want to run this service
confirm() {
  [ -x /bin/plymouth ] && /bin/plymouth --hide-splash
  while : ; do 
      echo -n $"Start service $1 (Y)es/(N)o/(C)ontinue? [Y] "
      read answer
      if strstr $"yY" "$answer" || [ "$answer" = "" ] ; then
         return 0
      elif strstr $"cC" "$answer" ; then
     rm -f /var/run/confirm
     [ -x /bin/plymouth ] && /bin/plymouth --show-splash
         return 2
      elif strstr $"nN" "$answer" ; then
         return 1
      fi
  done
}
```

第一个函数strstr的作用是判断第一个参数`$1`中是否包含了`$2`，如果包含了则返回状态码0。这函数也是一个不错的技巧。

第二个函数confirm的作用是根据交互式输入的值返回不同的状态码，如果输入的是y或Y或不输入时，返回0。输入的是c或C时，返回状态码2，输入的是n或N时返回状态码1。

于是可以根据confirm的状态值决定是否要继续执行某个程序。

用法和效果如下：

```bash
[root@xuexi ~]# confirm
Start service  (Y)es/(N)o/(C)ontinue? [Y] Y
[root@xuexi ~]# echo $?
0
[root@xuexi ~]# confirm
Start service  (Y)es/(N)o/(C)ontinue? [Y] 
[root@xuexi ~]# echo $?
0
[root@xuexi ~]# confirm
Start service  (Y)es/(N)o/(C)ontinue? [Y] n
[root@xuexi ~]# echo $?
1
[root@xuexi ~]# confirm
Start service  (Y)es/(N)o/(C)ontinue? [Y] c
[root@xuexi ~]# echo $?
2
```

<a name="blog5"></a>
## 5.pid检测相关函数

启动进程时，pid文件非常重要。不仅可以通过它判断进程是否在运行，还可以从中读取pid号用来杀进程。

<a name="blog5.1"></a>
### 5.1 checkpid、\_\_pids\_var\_run和\_\_pids\_pidof函数

1. pid文件的路径可能为`/var/run/$base.pid`文件(​`$base`表示进程名的basename)，也可能是自定义的路径，例如mysql的pid可以自定义为/mysql/data/mysql01.pid。但无论哪种情况，functions中的`__pids_var_run`函数都可以处理。

2. pid文件中可能有多行，表示多实例。

3. 每个进程都必有一个pid，但并不一定都记录在pid文件中，例如线程的pid。但无论如何，在/proc/目录下，一定会有pid号命名的目录，只要有对应pid号的目录，就表示该进程已经在运行。函数`checkpid`专门检测给定的pid值在/proc下是否有对应的目录存在。

4. 为了获取进程名的pid值，此处函数`__pids_pidof`使用的是`pidof`命令。该命令专门设计用来在脚本中取给定进程的pid。它的"-o"选项用于忽略某些进程号，在脚本中应用时常被忽略的是调用pidof的shell的PID，当前shell的PID以及父shell的pid。总之，该函数的目的就是为了获取合理无误的进程pid。

以下是函数`checkpid`、`__pids_var_run`和`__pids_pidof`的定义语句。

```bash
# Check if any of $pid (could be plural) are running
checkpid() {
    local i

    for i in $* ; do          # 检测/proc目录下是否存在给定的进程目录
        [ -d "/proc/$i" ] && return 0
    done
    return 1
}

# __proc_pids {program} [pidfile]
# Set $pid to pids from /var/run* for {program}.  $pid should be declared
# local in the caller.
# Returns LSB exit code for the 'status' action.
__pids_var_run() {                # 通过检测pid判断程序是否已在运行
    local base=${1##*/}           # 获取进程名的basename
    local pid_file=${2:-/var/run/$base.pid}     # 定义pid文件路径

    pid=
    if [ -f "$pid_file" ] ; then        # 给定的pid文件是否存在
            local line p

        [ ! -r "$pid_file" ] && return 4   # "user had insufficient privilege"
        while : ; do                       # 将pid文件中的pid值(可能有多行)赋值给pid变量
            read line
            [ -z "$line" ] && break
            for p in $line ; do
                [ -z "${p//[0-9]/}" ] && [ -d "/proc/$p" ] && pid="$pid $p"
            done
        done < "$pid_file"

            if [ -n "$pid" ]; then  # pid存在，则返回0。否则表示pid文件存在，但/proc下没有对应命令
                    return 0        # 即进程已死，但pid文件却存在，返回状态码1。
            fi
        return 1 # "Program is dead and /var/run pid file exists"
    fi
    return 3 # "Program is not running"    # pid文件不存在时，表示进程未运行，返回状态码3
}

# Output PIDs of matching processes, found using pidof
__pids_pidof() {             # 下面的pidof命令的意义见稍后解释
    pidof -c -m -o $$ -o $PPID -o %PPID -x "$1" || \   # 忽略当前shell的PID，父shell的pid和
                                                       # 调用pidof程序的shell的pid
        pidof -c -m -o $$ -o $PPID -o %PPID -x "${1##*/}"    # 总之就是找出合理的pid
}
```

从`__pidsvar_run`函数的定义语句中可以了解到，只有当pid文件存在，且/proc下有pid对应的目录时，才表示进程在运行(当然，线程没有pid文件)。`__pids_var_run`函数调用方法：

```bash
__pids_var_run program [pidfile]
```

如果不给定pidfile，则默认为`/var/run/$base.pid`文件。函数的执行结果为4种状态码：

- 0：program正在运行。  
- 1：program进程已死。pid文件存在，但/proc目录下没有对应的文件。  
- 3：pid文件不存在。  
- 4：pid文件的权限错误，不可读。  

除了返回状态码，`__pids_var_run`函数还会保存变量pid的结果，以供其他程序引用。

`__pids_pidof`中使用了pidof命令，其中使用了几个"-o"选项，它用于忽略指定的pid。但看上去`$$ $PPID %PPID`不是很好理解。`-o $$`是忽略的是shell进程，大多数时候它会继承父shell的pid，但在脚本中时它代表的是脚本所在shell的pid。`-o $PPID`忽略的是父shell。`-o %PPID`忽略的是调用pidof命令的shell。不是很好理解，可以参考下面的测试语句。

测试脚本：

```bash
#!/bin/bash

echo 'pidof bash: '`pidof bash`
echo 'script shell pid: '`echo $$`
echo 'script parent shell pid: '`echo $PPID`
echo 'pidof -o $$ bash: '`pidof -o $$ bash`
echo 'pidof -o $PPID bash: '`pidof -o $PPID bash`
echo 'pidof -o %PPID bash: '`pidof -o %PPID bash`
echo 'pidof -o $$ -o $PPID -o %PPID bash: '`pidof -o $$ -o $PPID -o %PPID bash`
```

测试语句：

```bash
[root@xuexi ~]# pidof bash
3306 2436 2302
[root@xuexi ~]# (echo 'parent shell: '$$;echo "current bash pid: `pidof bash`";./test.sh)|cat -n
 1  parent shell: 2302
 2  current bash pid: 3745 3306 2436 2302
 3  pidof bash: 3748 3745 3306 2436 2302
 4  script shell pid: 3748
 5  script parent shell pid: 3745
 6  pidof -o $$ bash: 3745 3306 2436 2302
 7  pidof -o $PPID bash: 3748 3306 2436 2302
 8  pidof -o %PPID bash: 3745 3306 2436 2302
 9  pidof -o $$ -o $PPID -o %PPID bash: 3306 2436 2302
```

第一个pidof命令：说明当前已有3个bash，pid为：3306、2436和2302。  
第二个命令：  
- 行1说明括号的父shell为2302。  
- 行5说明脚本的父shell为3745。即括号的父shell为当前bash环境，脚本的父shell为括号所在shell。  
- 行2减第一个命令的结果说明括号所在子shell的pid为3745。  
- 行3减行2说明shell脚本所在子shell的pid为3748。  
- `-o $$`忽略的是当前shell，即脚本所在shell的pid，因为在shell脚本中时，`​$$`不继承父shell的pid。  
- `-o $PPID`忽略的是pidof所在父shell，即括号所在shell。  
- `-o %PPID`忽略的是调用调用pidof程序所在的shell，即脚本所在shell。  

<a name="blog5.2"></a>

### 5.2 pidfileofproc和pidofproc函数

除了以上3个pid相关函数，functions文件中，还提供了两个函数`pidfileofproc`和`pidofproc`，均用于获取给定程序的pid值。

以下是pidfileofproc函数的定义语句。注意，该函数不是获取pidfile，而是获取pid值。

```bash
# A function to find the pid of a program. Looks *only* at the pidfile
pidfileofproc() {
    local pid

    # Test syntax.
    if [ "$#" = 0 ] ; then
        echo $"Usage: pidfileofproc {program}"
        return 1
    fi

    __pids_var_run "$1"          # 不提供pidfile，因此认为是/var/run/$base.pid
    [ -n "$pid" ] && echo $pid
    return 0
}
```

因此，`pidfileofproc`函数只能获取/var/run下的pid。

以下是pidofproc函数的定义语句：

```bash
# A function to find the pid of a program.
pidofproc() {
    local RC pid pid_file=

    # Test syntax.
    if [ "$#" = 0 ]; then
        echo $"Usage: pidofproc [-p pidfile] {program}"
        return 1
    fi
    if [ "$1" = "-p" ]; then    # 既可以获取/var/run/$base.pid中的pid，
        pid_file=$2             # 也可以获取自给定pid文件中的pid
        shift 2
    fi
    fail_code=3 # "Program is not running"

    # First try "/var/run/*.pid" files
    __pids_var_run "$1" "$pid_file"
    RC=$?
    if [ -n "$pid" ]; then     # $pid不为空时，输出program的pid值
        echo $pid
        return 0
    fi

    [ -n "$pid_file" ] && return $RC   # $pid为空，但使用了"-p"指定pidfile时，返回$RC。
    __pids_pidof "$1" || return $RC    # $pid为空，且$pidfile为空时，获取进程号pid并输出
}
```

这两个函数的区别在于pidfileofproc只能搜索/var/run下的pid，而pidofproc可以搜索自给定的pidfile或/var/run/下的pid。而前面的`__pids_pidof`函数，只有在获取bash进程时更精确(因为它会忽略父shell进程)。至于3个选哪个的问题，见[文末总结](#blog9.1)。

这两个函数用的比较少，但确实有使用它的脚本。如crond启动脚本中借助pidfileofproc来杀进程：

```bash
echo -n $"Stopping $prog: "
if [ -n "`pidfileofproc $exec`" ]; then
        killproc $exec
        RETVAL=3
else
        failure $"Stopping $prog"
fi
```

dnsbind的named服务启动脚本中借助pidofproc来判断进程是否已在运行。

```bash
pidofnamed() {
        pidofproc -p "$ROOTDIR$PIDFILE" "$named";
}

if [ -n "`pidofnamed`" ]; then
  echo -n $"named: already running"
  success
  echo
  exit 0;
fi;
```

<a name="blog6"></a>
## 6.重头戏(一)：daemon函数

daemon函数用于启动一个程序，并根据结果输出success或failure。

定义语句如下：

```bash
# A function to start a program.
daemon() {
    # Test syntax.
    local gotbase= force= nicelevel corelimit    # 定义一大堆变量
    local pid base= user= nice= bg= pid_file=
    local cgroup=
    nicelevel=0
    while [ "$1" != "${1##[-+]}" ]; do   # 当参数$1以"-"或"+"开头时进入循环，但$1为空时也满足
      case $1 in
        '')    echo $"$0: Usage: daemon [+/-nicelevel] {program}" "[arg1]..."
               return 1;;
        --check)                 # daemon接受"--arg value"和"--arg=value"两种格式的参数
           base=$2
           gotbase="yes"
           shift 2
           ;;
        --check=?*)
               base=${1#--check=}
           gotbase="yes"
           shift
           ;;
        --user)
           user=$2
           shift 2
           ;;
        --user=?*)
               user=${1#--user=}
           shift
           ;;
        --pidfile)
           pid_file=$2
           shift 2
           ;;
        --pidfile=?*)
           pid_file=${1#--pidfile=}
           shift
           ;;
        --force)
               force="force"
           shift
           ;;
        [-+][0-9]*)
               nice="nice -n $1"
               shift
           ;;
        *)     echo $"$0: Usage: daemon [+/-nicelevel] {program}" "[arg1]..."
               return 1;;
      esac
    done

        # Save basename.
        [ -z "$gotbase" ] && base=${1##*/}   # 若未传递"--check"，则此处获取bashname

        # See if it's already running. Look *only* at the pid file.
    __pids_var_run "$base" "$pid_file"

    [ -n "$pid" -a -z "$force" ] && return    # 如进程已在运行(已检测出pid)，且没有使用force
                                              # 强制启动，则退出daemon函数

    # make sure it doesn't core dump anywhere unless requested   
    corelimit="ulimit -S -c ${DAEMON_COREFILE_LIMIT:-0}"  # corelimit、cgroup和资源控制有关，忽略它

    # if they set NICELEVEL in /etc/sysconfig/foo, honor it
    [ -n "${NICELEVEL:-}" ] && nice="nice -n $NICELEVEL"
    
    # if they set CGROUP_DAEMON in /etc/sysconfig/foo, honor it
    if [ -n "${CGROUP_DAEMON}" ]; then
        if [ ! -x /bin/cgexec ]; then
            echo -n "Cgroups not installed"; warning
            echo
        else
            cgroup="/bin/cgexec";
            for i in $CGROUP_DAEMON; do
                cgroup="$cgroup -g $i";
            done
        fi
    fi

    # Echo daemon
        [ "${BOOTUP:-}" = "verbose" -a -z "${LSB:-}" ] && echo -n " $base"

    # And start it up.      # 启动程序。runuser的"-s"指定执行程序的shell，$user指定运行的身份
                            # "$*"是剔除掉daemon选项后程序的启动指令。
    if [ -z "$user" ]; then
       $cgroup $nice /bin/bash -c "$corelimit >/dev/null 2>&1 ; $*"
    else
       $cgroup $nice runuser -s /bin/bash $user -c "$corelimit >/dev/null 2>&1 ; $*"
    fi

    [ "$?" -eq 0 ] && success $"$base startup" || failure $"$base startup"
}
```

daemon函数调用方法：

```bash
daemon [--check=servicename] [--user=USER] [--pidfile=PIDFILE] [--force] program [prog_args]
```

需要注意的是：

1. 只有`--user`可以用来控制program启动的环境。  
2. `--check`和`--pidfile`都是用来检查是否已运行的，不是用来启动的，如果提供了`--check`，则检查的是名为servicename的进程，否则检查的是program名称的进程。  
3. `--force`则表示进程已存在时仍启动。  
4. prog\_args是向program传递它的运行参数，一般会从/etc/sysconfig/$base文件中获取。

例如httpd的启动脚本中。

```bash
echo -n $"Starting $prog: "
daemon --pidfile=${pidfile} $httpd $OPTIONS
```

这样的语句的执行结果大致如下：

```bash
[root@xuexi ~]# service httpd start
Starting httpd:                            [  OK  ]
```

还需注意，通常program的运行参数可能也是`--`开头的，要和program前面的选项区分。例如：
```bash
daemon --pidfile $pidfile --check $servicename $processname --pid-file=$pidfile
```
第二个`--pid-file`是`$processname`的运行参数，第一个`--pidfile`是daemon检测`$processname`是否已运行的选项。由于提供了`--check $servicename`，所以函数调用语句`__pids_var_run $base [pidfile]`中的`$base`等于`​$servicename`，即表示检查`$servicename`进程是否允许。如果没有提供该选项，则检查的是`​$processname`。

至此，daemon函数已经分析完成。实际上很简单，就是为daemon提供几个选项，再提供要执行的命令，并为该命令提供启动参数。

<a name="blog7"></a>
## 7.重头戏(二)：killproc函数

killproc函数的作用是根据给定程序名杀进程。中间它会获取程序名对应的pid号，且保证/proc目录下没有pid对应的目录才表示进程关闭成功。
```bash
# A function to stop a program.
killproc() {
    local RC killlevel= base pid pid_file= delay try
    
    RC=0; delay=3; try=0
    # Test syntax.
    if [ "$#" -eq 0 ]; then
        echo $"Usage: killproc [-p pidfile] [ -d delay] {program} [-signal]"
        return 1
    fi
    if [ "$1" = "-p" ]; then  # 指定pid_file。不给"-p"时，"__pids_var_run"将检查/var/run下的文件
        pid_file=$2
        shift 2
    fi
    if [ "$1" = "-d" ]; then  # awk的多目运算符。delay的有效值单位为d(天)、时(h)、分(m)、秒(s)。
                              # 不写单位时默认为秒。该语句将所给时间转换成秒，接受小数，做四舍五入计算
        delay=$(echo $2 | awk -v RS=' ' -v IGNORECASE=1 '{if($1!~/^[0-9.]+[smhd]?$/) exit 1;d=$1~/s$|^[0-9.]*$/?1:$1~/m$/?60:$1~/h$/?60*60:$1~/d$/?24*60*60:-1;if(d==-1) exit 1;delay+=d*$1} END {printf("%d",delay+0.5)}')
        if [ "$?" -eq 1 ]; then
            echo $"Usage: killproc [-p pidfile] [ -d delay] {program} [-signal]
            return 1
        fi
        shift 2
    fi
    
    # check for second arg to be kill level
    [ -n "${2:-}" ] && killlevel=​$2     # 获取稍后的kill程序将要发送的信号
    
    # Save basename.
    base=${1##*/}
    
    # Find pid.                       # 获取program的pid号，以让kill程序杀掉
    __pids_var_run "$1" "$pid_file"   # 检查program是否已有对应pid文件，并返回pidfile中所有pid值
    RC=$?
    if [ -z "$pid" ]; then
        if [ -z "$pid_file" ]; then
            pid="$(__pids_pidof "$1")"  # pid为空，且没有pidfile时，获取program的pid
        else
            [ "$RC" = "4" ] && { failure $"$base shutdown" ; return $RC ;}
        fi
    fi
    
    # Kill it.    # 根据pid，杀掉已存在的进程
    if [ -n "$pid" ] ; then    # 如果进程pid存在，则杀死它
        [ "$BOOTUP" = "verbose" -a -z "${LSB:-}" ] && echo -n "$base "
        if [ -z "$killlevel" ] ; then       # 没有指定要传递的信号时
            if checkpid $pid 2>&1; then  # 给定pid在/proc目录中是否有对应目录
                # TERM first, then KILL if not dead
                kill -TERM $pid >/dev/null 2>&1   # 先发送TERM信号
                usleep 50000
                if checkpid $pid ; then           # 0.5秒后还没死透，则
                    try=0
                    while [ $try -lt $delay ] ; do   # 在给定delay时间内不断检测是否已死
                        checkpid $pid || break
                        sleep 1
                        let try+=1
                    done
                    if checkpid $pid ; then          # 超出delay后，发送KILL信号强制杀死
                        kill -KILL $pid >/dev/null 2>&1
                        usleep 50000
                    fi
                fi
            fi
            checkpid $pid   # 若/proc下还有pid对应的目录，则进程关闭失败
            RC=$?
            [ "$RC" -eq 0 ] && failure $"$base shutdown" || success $"$base shutdown"
            RC=$((! $RC))
        # use specified level only
        else                            # 使用指定的信号杀进程
            if checkpid $pid; then
                kill $killlevel $pid >/dev/null 2>&1
                RC=$?
                [ "$RC" -eq 0 ] && success $"$base $killlevel" || failure $"$base $killlevel"
            elif [ -n "${LSB:-}" ]; then
                RC=7 # Program is not running
            fi
        fi
    else                              # 如果进程pid不存在，表示未运行
        if [ -n "${LSB:-}" -a -n "$killlevel" ]; then
            RC=7 # Program is not running
        else
            failure $"$base shutdown"
            RC=0
        fi
    fi
    
    # Remove pid file if any.
    if [ -z "$killlevel" ]; then    # 未给定信号时，可能KILL信号强杀时使得pid文件还存在，手动移除它
            rm -f "${pid_file:-/var/run/$base.pid}"
    fi
    return $RC
}
```

根据此脚本，可以知道关闭进程时，需要再三确定pid文件是否存在，/proc下是否有和pid对应的目录。**直到/proc下已经没有了和pid对应的目录时，才表示进程真正杀死了。但此时pid文件仍可能存在**，因此还要保证pid文件已被移除。

该函数的调用方法：

```bash
killproc [-p pidfile] [ -d delay] {program} [-signal]
```

<a name="blog8"></a>
## 8.重头戏(三)：status函数

status函数用于获取进程的运行状态，有以下几种状态：
```bash
${base} (pid $pid) is running...  
${base} dead but pid file exists  
${base} status unknown due to insufficient privileges.  
${base} dead but subsys locked  
${base} is stopped  
```
以下的status函数定义语句。注意，此为CentOS 7上语句，比CentOS 6多了一段systemctl的处理，用于Sysv的status状态向systemd的status状态转换。

```bash
status() {
    local base pid lock_file= pid_file=

    # Test syntax.
    if [ "$#" = 0 ] ; then
        echo $"Usage: status [-p pidfile] {program}"
        return 1
    fi
    if [ "$1" = "-p" ]; then
        pid_file=$2           # 指定pidfile
        shift 2
    fi
    if [ "$1" = "-l" ]; then
        lock_file=$2          # 指定lockfile
        shift 2
    fi
    base=${1##*/}

    if [ "$_use_systemctl" = "1" ]; then
        systemctl status ${0##*/}.service
        ret=$?
        # LSB daemons that dies abnormally in systemd looks alive in 
        # systemd's eyes due to RemainAfterExit=yes
        # lets adjust the reality a little bit
        if systemctl show -p ActiveState ${0##*/}.service | grep -q '=active$' && \
        systemctl show -p SubState ${0##*/}.service | grep -q '=exited$' ; then
            ret=3
        fi
        return $ret
    fi

    # First try "pidof"
    __pids_var_run "$1" "$pid_file"   # 根据给定的pidfile获取program的pid，并返回pid值
    RC=$?
    if [ -z "$pid_file" -a -z "$pid" ]; then   # pid为空，且没有pidfile时，获取program的pid
        pid="$(__pids_pidof "$1")"
    fi
    if [ -n "$pid" ]; then             # pid存在，则返回程序正在运行
        echo $"${base} (pid $pid) is running..."
        return 0
    fi

    case "$RC" in
        0)
            echo $"${base} (pid $pid) is running..."
            return 0
            ;;
        1)               # program进程已死。pid文件存在，但/proc目录下没有对应的文件。
            echo $"${base} dead but pid file exists"
            return 1
            ;;
        4)               # pid文件不可读，错误
            echo $"${base} status unknown due to insufficient privileges."
            return 4
            ;;
    esac
    if [ -z "${lock_file}" ]; then
        lock_file=${base}
    fi
    # See if /var/lock/subsys/${lock_file} exists  
    if [ -f /var/lock/subsys/${lock_file} ]; then   # 检查/var/lock/subsys下是否有lockfile
        echo $"${base} dead but subsys locked"      # pid不存在，但锁文件存在时
        return 2
    fi
    echo $"${base} is stopped"   # 以上都不满足时，表示程序未运行
    return 3
}
```

函数调用方法：

```bash
status [-p pidfile] [-l lockfile] program
```

由于函数定义原因，如果同时提供"-p"和"-l"选项，"-l"选项必须放在"-p"的后面。

<a name="blog9"></a>
## 9.几个重要函数的总结和使用说明

functions文件重要的东西差不多都介绍了，还有些无所谓的东西就忽略它们好了。看完这么多分析，肯定会晕头转向，所以给个总结。至于前面几个简单的函数`echo_success`、`echo_failure`、`echo_passed`、`echo_warning`、`success`、`failure`、`passed`、`warning`、`action`、`confirm`、`is_true`、`is_false`就懒的总结了，用法都很简单。

<a name="blog9.1"></a>
### 9.1 pid相关

- `checkpid`：检查/proc下是否有给定pid对应的目录，无论给定多少个pid，只要有一个有目录，都返回0。  

调用方法：`checkpid pid_list`

```bash
[root@xuexi ~]# source /etc/init.d/functions
[root@xuexi ~]# sleep 10 & a="$!";sleep 10 & a="$a $!";sleep 10 & a="$a $!";checkpid $a
[root@xuexi ~]# echo $?
0
```

- `__pids_var_run`：检查pid是否存在，并保存到变量pid中，同时返回几种进程状态码。  

这个函数非常重要，不仅从pidfile中获取并保存pid号码，还根据情况返回几种状态码，这几个状态码是status函数的重要依据。在SysV服务启动脚本中使用非常广泛。

调用方法：`__pids_var_run program [pidfile]`

以下是httpd进程的测试结果。分别是指定pid文件和不指定pid文件的情况。

```bash
[root@xuexi ~]# service httpd start
[root@xuexi ~]# __pids_var_run httpd /var/run/httpd/httpd.pid 
[root@xuexi ~]# echo $?
0
[root@xuexi ~]# echo $pid
4863
[root@xuexi ~]# __pids_var_run httpd  # 不指定pidfile时，将搜索/var/run/httpd.pid
[root@xuexi ~]# echo $?
3
[root@xuexi ~]# echo $pid      # 每次调用该函数Pid会重置

[root@xuexi ~]# 
```

- `__pids_pidof`：获取进程pid。  
- `pidfileofproc`：获取进程的pid。但只能获取/var/run下的pid文件中的值。  
- `pidofproc`：获取进程的pid。可获取任意给定pidfile或默认/var/run下pidfile中的值。  

都是获取进程pid，第一个函数和后两个的区别主要在于获取的pid是bash进程时更精确，第二个和第三个函数的区别在于第2个函数只能获取/var/run下pid文件中的pid值。

```bash
[root@xuexi ~]# service httpd restart
[root@xuexi ~]# pidfileofproc httpd
[root@xuexi ~]# pidofproc httpd     
4872 4871 4870 4869 4868 4867 4866 4865 4863
[root@xuexi ~]# __pids_pidof httpd
4872 4871 4870 4869 4868 4867 4866 4865 4863
```

上面pidfileofproc命令没有任何结果，因为httpd的pid文件为/var/run/httpd/httpd.pid，而非/var/run/httpd.pid。

如果将httpd的pid路径修改为/var/run/httpd.pid，再看它们的结果。

```bash
[root@xuexi ~]# service httpd stop
[root@xuexi ~]# sed -i "s%^PidFile.*%PidFile /var/run/httpd.pid%" /etc/httpd/conf/httpd.conf 
[root@xuexi ~]# sed -i 's%^#PIDFILE.*%PIDFILE=/var/run/httpd.pid%' /etc/sysconfig/httpd  
[root@xuexi ~]# service httpd start

[root@xuexi ~]# ls /var/run/httpd*
/var/run/httpd.pid

/var/run/httpd:
```

再看它们搜索到的pid以及进程列表中httpd的pid和pid文件中的pid。

```bash
[root@xuexi ~]# __pids_pidof httpd
6235 6234 6233 6232 6231 6230 6229 6228 6226
[root@xuexi ~]# pidofproc httpd
6226
[root@xuexi ~]# pidfileofproc httpd
6226

[root@xuexi ~]# ps aux | grep http[d]
root       6226  0.0  0.3 177844  3892 ?        Ss   12:14   0:00 /usr/sbin/httpd
apache     6228  0.0  0.2 177844  2532 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6229  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6230  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6231  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6232  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6233  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6234  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
apache     6235  0.0  0.2 177844  2508 ?        S    12:14   0:00 /usr/sbin/httpd
[root@xuexi ~]# cat /var/run/httpd.pid 
6226
```

所以，要使用这3个函数中的哪一个？**如果要找出进程的"master"进程号，例如要向主进程发送HUP信号reload配置文件时，应该用pidofproc并使用"-p"指定pid文件。其余时候用`__pids_pidof`准没错，也正是如此，在daemon和killproc函数中都使用了它。另外，在多实例的情况下，也可以考虑使用pidofproc来根据pidfile搜索对应实例的pid。**

<a name="blog9.2"></a>
### 9.2 daemon的使用

- `daemon`：启动一个服务程序。在启动前还检查是否已在运行。  


调用方式：

```bash
daemon [--check=servicename] [--user=USER] [--pidfile=PIDFILE] [--force] program [prog_args]
```

`--user`用于指定进程运行身份，`--check`和`--pidfile`用于指定检查进程是否已在运行，`--force`表示即使在运行也同样再启动一个程序。prog_args用于为program程序提供启动参数。

一般daemon会配合以下几个语句同时执行，这正是SysV脚本的一个特点。

```bash
echo -n $"Starting $prog: "
daemon --pidfile=${pidfile} $prog $OPTIONS
RETVAL=$?
[ $RETVAL = 0 ] && touch ${lockfile}
return $RETVAL
```

注意，daemon函数启动程序时，自身就会调用success或failure函数，所以就不需再使用action函数了。如果不使用daemon函数启动服务，通常会配合action函数。例如：


```bash
$prog $OPTIONS
RETVAL=$?
[ $RETVAL -eq 0 ] && action "Starting $prog" /bin/true && touch ${lockfile}
```


<a name="blog9.3"></a>
### 9.3 killproc的使用

- `killproc`：杀掉给定的服务进程。  

函数调用方式：

```bash
killproc [-p pidfile] [-d delay] program [-signal]
```

- `-p pidfile`：选项用于指定从此文件中获取进程的pid号，不指定时默认从`/var/run/$base.pid`中获取。  
- `-signal`：用于指定kill发送的信号。如果不指定，则默认先发送TERM信号，在`-d delay`时间段内仍不断检测是否进程已经被杀死，如果还未死透，则delay超时后发送KILL信号强制杀死。  
- `-d delay`：指定未使用`-signal`时的延迟检测时间。有效单位为秒、分、时、日("smhd")，不写时默认为秒。

需要明确的是，只有/proc目录下没有了pid对应的目录才算是杀死了。

一般来说，killproc前会判断进程是否已在运行，最后还要删除pid文件和lock文件。当然，killproc函数可以保证pid文件被删除。所以，killproc函数大致会同时配合以下语句用来杀进程：

```bash
    status -p ${pidfile} $prog > /dev/null
    if [[ $? = 0 ]]; then
            echo -n $"Stopping $prog: "
            killproc -p ${pidfile} -d ${STOP_TIMEOUT} $httpd
    else
            echo -n $"Stopping $prog: "
            success
    fi
    RETVAL=$?
    [ $RETVAL -eq 0 ] && rm -f ${lockfile} ${pidfile}
```

同样注意，killproc中已经自带success和failure函数。如果不使用killproc杀进程，则通常会配合action函数或者success、failure。大致如下：

```bash
killall $prog ； usleep 50000 ； killall $prog
RETVAL=$?
if [ "RETVAL" -ne 0 ];then
    action $"Stopping $prog: " /bin/true
    rm -rf ${lockfile} ${pidfile}
else
    action $"Stoping $prog: " /bin/false
fi
```

以上由于采用的是killall命令，如果采用的是kill命令，则需要先获取进程的pid，在此之前还要检查pid文件是否存在。

<a name="blog9.4"></a>
### 9.4 status的使用
- `status`：检查给定进程的运行状态。  

用于返回进程状态。调用方式：注意"-p"必须在"-l"前面

```bash
status [-p pidfile] [-l lockfile] program
```

共有以下几种状态：
```bash
${base} (pid $pid) is running...  
${base} dead but pid file exists  
${base} status unknown due to insufficient privileges.  
${base} dead but subsys locked  
${base} is stopped  
```

<a name="blog10"></a>
## 10.memcached服务启动脚本示例

以下是memcached服务启动脚本的示例，是一个非常简单但却非常通用的Sysv服务启动脚本。

```bash
#!/bin/bash
#
# chkconfig: - 86 14
# description: Distributed memory caching daemon

## Default variables
PORT="11211"
USER="nobody"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS=""

RETVAL=0
prog="/usr/local/memcached/bin/memcached"
desc="Distributed memory caching"
lockfile="/var/lock/subsys/memcached"

. /etc/rc.d/init.d/functions
[ -f /etc/sysconfig/memcached ] && source /etc/sysconfig/memcached

start() {
        echo -n $"Starting $desc (memcached): "
        daemon $prog -d -p $PORT -u $USER -c $MAXCONN -m $CACHESIZE "$OPTIONS"
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && touch $lockfile
        return $RETVAL
}

stop() {
        echo -n $"Shutting down $desc (memcached): "
        killproc $prog
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && rm -f $lockfile
        return $RETVAL
}

restart() {
        stop
        start
}

reload() {
        echo -n $"Reloading $desc ($prog): "
        killproc $prog -HUP
        RETVAL=$?
        echo
        return $RETVAL
}

case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  restart)
        restart
        ;;
  condrestart)
        [ -e $lockfile ] && restart
        RETVAL=$?
        ;;       
  reload)
        reload
        ;;
  status)
        status $prog
        RETVAL=$?
        ;;
   *)
        echo $"Usage: $0 {start|stop|restart|reload|condrestart|status}"
        RETVAL=1
esac

exit $RETVAL
```

另请参考[如何写SysV服务管理脚本][1]。

[1]: /shell/sysv_script  "如何写SysV服务管理脚本"

