---
title: 如何写SysV服务管理脚本
p: shell/sysv_script.md
date: 2021-03-13 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# 如何写SysV服务管理脚本

SysV服务管理脚本和/etc/rc.d/init.d/functions文件中的几个重要函数"关系匪浅"本人已对该文件做了极详细的分析和说明，参考[functions文件详细分析和说明][1]。

<a name="blog1.1"></a>
## SysV脚本的特性

SysV风格的服务启动脚本有以下几个特性：

1. 一般都放在/etc/rc.d/init.d目录下。  
2. 这类脚本要求能接受start、stop、restart、status等参数来管理服务进程。  
3. 基本上都会加载/etc/rc.d/init.d/functions文件，因为该文件中定义了几个对进程管理非常有用的函数。  
4. 基本上都会加载/etc/sysconfig目录下的同名文件。此目录下的服务同名文件一般都是为服务管理脚本提供选项参数的。例如/etc/sysconfig/httpd。  
5. 在脚本的顶端，需要加上`# chkconfig`和`# description`两行。chkconfig行定义的是该脚本被chkconfig工具管理时的主要依据，包括开机和关机时的启动、关闭顺序，以及运行在哪些运行级别。description是该脚本的描述性语句。虽然这两行以`#`开头，但必不可少。

例如，/etc/init.d/httpd脚本的前面几行内容如下：

```bash
#!/bin/bash
#
# httpd        Startup script for the Apache HTTP Server
#
# chkconfig: - 85 15
# description: The Apache HTTP Server is an efficient and extensible  \
#              server implementing the current HTTP standards.
# processname: httpd
# config: /etc/httpd/conf/httpd.conf
# config: /etc/sysconfig/httpd
# pidfile: /var/run/httpd/httpd.pid
#

# Source function library.
. /etc/rc.d/init.d/functions

if [ -f /etc/sysconfig/httpd ]; then     # 判断后再加载
    . /etc/sysconfig/httpd
fi
```

<a name="blog1.2"></a>

## SysV脚本要具备的能力

要使用脚本管理服务进程，该脚本还要求具备以下能力，且处理逻辑越完善，脚本就越完美。

1. 启动进程时：  
    - 要求能够检测进程是否已在运行。这包括检测pid文件是否存在、/proc目录下是否有进程pid值对应的目录。  
    - 应该为程序创建锁文件，路径一般在/var/lock/subsys目录下。  
    - 如果使用daemon函数启动进程，允许`--user`指定程序的运行身份。  
    - 有些进程启动时需要依赖于其他进程，如NFS启动时依赖于rpcbind服务、mountd服务等，所以在NFS脚本中必须能够检测并启动这些依赖服务。  
2. 关闭进程时：  
    - 要求能够检测进程是否已在运行。同样是检测pid文件是否存在，/proc目录下是否有pid对应的目录。要注意，只有/proc目录下没有了对应目录，才表示进程已死，但pid文件仍可能存在，例如`kill -9`就会出现这种问题。  
    - 可以使用functions文件中的killproc函数杀进程，也可以直接使用kill或killall。  
    - 为了让脚本更完善，杀进程时应该多次检测进程是否真的已经杀死。  
    - 杀死进程的最后，必须要删除pid文件和锁文件。  
    - 对于有依赖性的服务，考虑是否也应该杀死它们。  
3. 服务重读配置文件时(reload)：  
    - 对于非终端进程，发送HUP信号的作用是重读配置文件，而不会中断进程。  
    - 为了标准，应该找出"master"进程的pid，并向其发送HUP信号。一般来说，服务的子进程或线程不会也没必要读取配置文件。为了方便，可以直接向所有进程发送HUP信号。  
    - 最好在发送HUP信号前，也检查进程是否已在运行。当然，对于reload来说，这无所谓。  
    - 如果待管理程序支持配置文件的语法检查，在发送HUP信号前，应该检查语法是否错误。  
    - 实在无法实现重读配置文件的功能，应该让其和restart的功能一致，一般这也是"force-reload"的功能。  
4. 重启服务时：  
    - 一般来说，就是先stop，再start。  
5. 查看status时：  
    - 除非有额外的状态显示需求，否则/etc/init.d/functions中的status函数已经足够完美了。  

以上并没有说明，管理多实例服务时的情况。这需要考虑额外的因素，例如程序自身是否支持多实例，支持的话是否应该写多个服务脚本分别管理各程序，配置文件是否要共享，pid文件是否能共享，搜索pid时如何避免搜索出非自身实例的pid，还要注意分配锁文件。这样的脚本写起来可能并不难，但这些因素必须要考虑。本文暂不介绍多实例的SysV脚本，因为和程序自身关联性比较强。

有了以上内容，并理解了functions文件中的函数，再看/etc/init.d/下的服务启动脚本，绝大多数都感觉很简单。因为它们的思路和框架都是一致的。

因为网上以及/etc/init.d/下服务启动脚本示例太多了，所以本文不单独写这类脚本，而是从几个脚本中抽出比较经典的部分，分别介绍start，stop，reload和status的写法。

<a name="blog1.3"></a>
## start函数分析

以httpd的服务管理脚本/etc/init.d/httpd为例。

```
start() {
    echo -n $"Starting $prog: "
    LANG=$HTTPD_LANG daemon --pidfile=${pidfile} $httpd $OPTIONS
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && touch ${lockfile}
    return $RETVAL
}
```

函数首先输出`Starting $prog`信息，再使用daemon启动`$httpd`程序。

在daemon语句中，`--pidfile`是daemon的参数，该参数为daemon检测pid文件是否存在，`$httpd`进程是否已在运行。注意，这个`--pidfile`是写在`$httpd`前面的，表示这是daemon的参数，而非`$httpd`的启动参数。

检测完成后，启动程序。程序的启动命令从`$httpd`参数开始，`$OPTIONS`是`$httpd`的启动选项。一般出现`$OPTIONS`这个字眼，很可能加载了/etc/sysconfig目录下的同名文件，目的是提供程序启动参数。

如果启动成功，则会daemon函数会调用functions中的success函数显示`[  OK  ]`，否则会显示`[  FAILED  ]`。

最后，如果启动成功，则会创建该进程的锁文件`$lockfile`。锁文件一般都在/var/lock/subsys目录下。

很多时候，管理的进程也有`--pidfile`类似的选项。例如下面的启动语句：

```bash
daemon --pidfile $pidfile $processname --pidfile=$pidfile
```

两个`--pidfile`选项，但他们的作用是不一样的。第一个`--pidfile`是daemon函数的参数，以便daemon能够检测该文件中的pid进程是否已在运行。第二个`--pidfile`是`$processname`的启动参数，启动时会创建此文件作为pid文件。

再看一个不使用daemon函数管理进程启动动作的示例。以下是/etc/init.d/sshd中的start函数内容。

```bash
start()
{
        [ -x $SSHD ] || exit 5
        [ -f /etc/ssh/sshd_config ] || exit 6
        # Create keys if necessary
        if [ "x${AUTOCREATE_SERVER_KEYS}" != xNO ]; then
                do_rsa_keygen
                if [ "x${AUTOCREATE_SERVER_KEYS}" != xRSAONLY ]; then
                        do_rsa1_keygen
                        do_dsa_keygen
                fi
        fi

        echo -n $"Starting $prog: "
        $SSHD $OPTIONS && success || failure
        RETVAL=$?
        [ $RETVAL -eq 0 ] && touch $lockfile
        echo
        return $RETVAL
}
```

前面多了一大段，这和服务启动脚本的框架无关，是程序自身要求的，但作用很简单。无非就是判断下程序是否可执行，配置文件是否存在，是否要创建服务端主机验证阶段的密钥对，也就是`/etc/ssh/ssh\_host\_{rsa,dsa}\_key`等几个文件。

再下面才是服务启动脚本中的通用逻辑部分。输出一段信息，然后启动程序，创建锁文件。但这里没有使用daemon函数管理，所以这里配合了success和failure函数以便人性化显示`[  OK  ]`或`[  FAILED  ]`。

<a name="blog1.4"></a>

## stop函数分析

仍然以/etc/init.d/httpd中的stop函数为例。

```bash
# When stopping httpd, a delay (of default 10 second) is required
# before SIGKILLing the httpd parent; this gives enough time for the
# httpd parent to SIGKILL any errant children.
stop() {
    status -p ${pidfile} $httpd > /dev/null
    if [[ $? = 0 ]]; then
        echo -n $"Stopping $prog: "
        killproc -p ${pidfile} -d ${STOP_TIMEOUT} $httpd
    else
        echo -n $"Stopping $prog: "
        success
    fi
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && rm -f ${lockfile} ${pidfile}
}
```

前面加了一段注释，大致意思是说这里杀死进程的行为和httpd自带的apachectl工具停止服务的命令`apachectl -k stop`的行为是不同的。之所以我要把这一段也贴上来，也就是为了说明这一点。有些服务程序自带进程管理工具，亦或是使用functions中的函数，完全由我们自己决定。

再看stop函数的逻辑。首先使用"status"函数检查进程的状态，如果进程已在运行，则使用killproc函数杀掉它，否则表示进程未运行或进程已死，但pid文件还存在。所以，在最后删掉pidfile和lockfile。

需要注意的是，killproc杀进程时，能保证pidfile同时被删除。但它不负责lockfile，而且执行stop之前曾手动执行了`kill -9`杀进程，那么进程虽然已死，但pid文件却存在。因此也仍需手动rm删除pidfile。

killproc的调用方法为：

```bash
killproc [-p $pidfile] -[d $delay] $processname [-signal]
```

它的逻辑和执行过程是这样的：  

1. 根据pidfile找出要杀的pid，如果没有指定pidfile，则默认从`/var/run/$base.pid`读取；  
2. 如果指定了要发送的信号，则killproc通过kill命令发送给定信号。0.5秒后检查/proc目录下是否还有对应目录存在，有则说明进程杀死失败，返回`[  FAILED  ]`信息，否则表示成功，于是删除pid文件。  
3. 如果没有指定要发送的信号，则killproc先发送TERM信号(即`kill -15`)，然后在给定的延迟时间delay内，每隔一秒检查一次/proc下是否有对应目录，如果发现没有，则表示进程杀死成功，于是删除pid文件(其实这种情况不用删，因为TERM信号会自动做收尾动作)。但如果delay都超时了，还发现进程存在，则发送KILL信号强制杀死进程，最后删除pid文件。  

现在再理解`killproc -p ${pidfile} -d ${STOP_TIMEOUT} $httpd`就很简单了。

再看/etc/init.d/sshd脚本中的stop。

```bash
stop()
{
    echo -n $"Stopping $prog: "
    killproc -p $PID_FILE $SSHD
    RETVAL=$?
    # if we are in halt or reboot runlevel kill all running sessions
    # so the TCP connections are closed cleanly
    if [ "x$runlevel" = x0 -o "x$runlevel" = x6 ] ; then
        trap '' TERM
        killall $prog 2>/dev/null
        trap TERM
    fi
    [ $RETVAL -eq 0 ] && rm -f $lockfile
    echo
}
```

更直接，直接就killproc。但是后面还设置了runlevel的判断情况，这就属于程序自身属性了，和服务管理脚本的逻辑框架无关。

最后再看mysqld中的stop函数。

```bash
stop(){
    if [ ! -f "$mypidfile" ]; then
        # not running; per LSB standards this is "ok"
        action $"Stopping $prog: " /bin/true      # pid文件都不存在，直接显示成功
        return 0
    fi
    MYSQLPID=`cat "$mypidfile" 2>/dev/null`       # 读取pidfile中的pid号
    if [ -n "$MYSQLPID" ]; then                   # 如果pid不为空，则
        /bin/kill "$MYSQLPID" >/dev/null 2>&1     # 先发送默认的TERM信号杀一次
        ret=$?
        if [ $ret -eq 0 ]; then         # 如果杀成功了，则执行下面一段。
                                        # 否则直接失败，但这不可能。为了逻辑完整，后面仍写了else
            TIMEOUT="$STOPTIMEOUT"
            while [ $TIMEOUT -gt 0 ]; do   # 在延迟时间内，每隔1秒杀一次
                /bin/kill -0 "$MYSQLPID" >/dev/null 2>&1 || break
                sleep 1
                let TIMEOUT=${TIMEOUT}-1
            done
            if [ $TIMEOUT -eq 0 ]; then    # 如果达到延迟时间边界，则返回杀死进程超时信息
                echo "Timeout error occurred trying to stop MySQL Daemon."
                ret=1
                action $"Stopping $prog: " /bin/false
            else                           # 否则进程杀死成功，删除pidfile和lockfile
                rm -f $lockfile
                rm -f "$socketfile"
                action $"Stopping $prog: " /bin/true
            fi
        else
            action $"Stopping $prog: " /bin/false
        fi
    else                                   # 如果pid为空，则表示未成功读取pidfile。
        # failed to read pidfile, probably insufficient permissions
        action $"Stopping $prog: " /bin/false
        ret=4
    fi
    return $ret
}
```

虽然有点长，但有了前面[SysV脚本要具备的能力](#blog1.2)的概念，stop函数的逻辑都一样好简单。

<a name="blog1.5"></a>
## reload函数分析

关于reload函数，主要有两点：(1).语法检查；(2).发送HUP信号给"master"进程。其中语法检查要程序自身能支持，例如`httpd -t`，`nginx -t`。

以下是`/etc/init.d/{httpd,nginx}`两个脚本中的reload函数。

```bash
## reload() in /etc/rc.d/init.d/httpd
reload() {
    echo -n $"Reloading $prog: "
    if ! LANG=$HTTPD_LANG $httpd $OPTIONS -t >&/dev/null; then  # 语法检查
        RETVAL=6
        echo $"not reloading due to configuration syntax error"
        failure $"not reloading $httpd due to configuration syntax error"
    else
        # Force LSB behaviour from killproc        # 语法检查通过，发送HUP信号
        LSB=1 killproc -p ${pidfile} $httpd -HUP
        RETVAL=$?
        if [ $RETVAL -eq 7 ]; then             # 注意reload失败时退出状态码为7
            failure $"httpd shutdown"
        fi
    fi
    echo
}

## reload() in /etc/rc.d/init.d/nginx
reload() {
    configtest_q || return 6           # 语法检查
    echo -n $"Reloading $prog: "
    killproc -p $pidfile $prog -HUP    # 发送HUP信号
    echo
}

configtest_q() {
    $nginx -t -q -c $NGINX_CONF_FILE
}


case "$1" in
    reload)
        rh_status_q || exit 7        # reload失败时，退出状态码7
        $1
        ;;
```

唯一需要注意的是，reload失败时，退出状态码为7。这大概已经约定俗成了吧。

再看/etc/init.d/sshd中的reload。

```bash
reload()
{
    echo -n $"Reloading $prog: "
    killproc -p $PID_FILE $SSHD -HUP
    RETVAL=$?
    echo
}

case "$1" in
    reload)
        rh_status_q || exit 7
        reload
        ;;
```

有意思的是mysqld的reload。它直接退出不做任何动作。

```bash
case "$1" in
  reload)
    exit 3
    ;;
```


如果不使用killproc函数，而是使用kill命令，那么应该找出"master" pid。可以使用functions中的pidofproc函数。例如：

```bash
pid=$(pidofprco -p pidfile $processname)
action "Reloading $prog: " kill -HUP $pid
```

<a name="blog1.6"></a>
## status、restart、force-reload等

- status：就是为了获取进程状态的，一般直接调用functions中的status函数`status -p "$pidfile" $prog`。  
- restart：一般直接stop再start即可。  
- force-reload：其实就是restart。  
- condrestart：称为条件式重启。所谓的条件一般是判断锁文件是否存在，存在则重启，否则忽略该动作。"try-restart"也是一样的行为。  

<a name="blog1.7"></a>
## 结束语

其实SysV服务启动脚本大多都很简单，至少它们的逻辑几乎都一样。在了解了functions中的几个函数后，再把脚本的各参数(如start、stop)应该要具备的能力搞搞清楚，这类脚本完全是小菜一两碟。




[1]: https://www.junmajinlong.com/shell/functions "functions文件详细分析和说明"

