---
title: systemd时代的服务管理
p: linux/systemd/service_manage.md
date: 2020-06-27 18:20:52
tags: Linux
categories: Linux
---

------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Systemd系列文章大纲](/linux/index#systemd)**  
**[回到Shell系列文章大纲](/shell/index)**  

------

# systemd时代的服务管理

使用systemd做服务管理时，需要了解一些基本知识：  
1. 了解systemd可管理哪些服务  
2. 了解systemd所管理服务的状态  
3. 了解systemctl管理服务的基本命令  
4. 学会编写、修改、看懂服务Unit配置文件  

此处介绍前(3)点相关的内容，第(4)点内容较多，后面的文章再详细介绍。

## systemd可管理哪些服务

操作系统使用systemd后，所有用户进程都是systemd的后代进程。

```bash
$ pstree -p
systemd(1)─┬─agetty(1056)
           ├─auditd(737)───{auditd}(738)
           ├─crond(810)
           ├─dbus-daemon(761)
           ├─dhclient(966)
           ├─irqbalance(764)
           ├─lvmetad(573)
           ├─master(1140)─┬─pickup(1141)
           │              └─qmgr(1142)
           ├─mysqld(1068)─┬─{mysqld}(1161)
           │              └─......
           ├─polkitd(763)─┬─{polkitd}(814)
           │              ├─......
           ├─rpcbind(762)
           ├─rsyslogd(1047)─┬─{rsyslogd}(1093)
           │                └─{rsyslogd}(1094)
           ├─sshd(1042)───sshd(1110)─┬─bash(1143)───pstree(2240)
           │                         └─bash(1492)
           ├─systemd-journal(545)
           ├─systemd-logind(773)
           ├─systemd-udevd(576)
           └─tuned(1024)─┬─{tuned}(1377)
                         ├─{tuned}(1378)
                         ├─{tuned}(1380)
                         └─{tuned}(1381)
```

**当某个进程不在某个绑定了终端的Shell进程下，该进程必然是脱离终端脱离Shell的Daemon类进程或其子孙进程**。

虽然从进程树关系来看，所有进程都直接或间接地受到systemd的管理，但是，并非所有systemd的子进程都受Systemd Unit管理单元的管理。只有那些由systemd方式启动的服务进程(比如systemctl命令启动)才受到Systemd Unit管理单元的监控和管理。为了简化描述，后面均直接以『systemd管理』来描述受systemd unit管理单元的管理。

比如，用户可以通过下面两种方式启动Nginx服务进程：

```bash
nginx                    # (1)
systemctl start nginx    # (2)
```

但systemd只能监控、管理第(2)种方式启动的nginx服务。比如第一种方式启动的nginx，无法使用`systemctl stop nginx`来停止。

所以，**systemd下的直系子进程可分为两类：受systemd管理的子进程和不受systemd管理的子进程**。

![](/img/linux/1593509043956.png)

## systemd管理服务的命令

注，下面这些命令可同时操作多个服务，且可对服务名称做模式匹配。

启动、停止服务:
```bash
systemctl start Service_Name1 Service_Name2
systemctl stop Service_Name
```
服务重载、重启相关操作：
```bash
# 重载服务：服务未运行时不做任何事
systemctl reload Service_Name

# 重启服务：服务已运行时重启之，服务未运行时启动之
systemctl restart Service_Name

# 服务已运行时重启之，未运行时不启动之
systemctl try-restart Service_Name

# 服务已运行时，如果支持reload，则reload，如果不支持则restart
# 服务未运行时，启动之
systemctl reload-or-restart Service_Name

# 服务已运行时，如果支持reload，则reload，如果不支持则restart
# 服务未运行时，不做任何事
systemctl reload-or-try-restart Service_Name
```

服务状态查看操作：
```bash
# 查看服务状态
systemctl status Service_Name

# 检查服务是否active: 服务是否已启动
# 至少一个服务active时，返回0，否则返回非0退出状态码
systemctl is-active Service_Name1 Service_Name2
systemctl --quiet is-active Service_Name  # 静默模式

# 检查服务是否failed: 服务启动命令退出状态码非0或启动超时
systemctl is-failed Service_Name
```

## systemd所管理服务的状态说明

当使用systemd管理服务时，有必要了解服务的各种状态信息。

使用`systemctl list-units --type=service`可以列出Service Unit的状态信息：

```
$ systemctl list-units --type service
  UNIT              LOAD   ACTIVE SUB     DESCRIPTION
  auditd.service    loaded active running Security Auditing Service
  cgconfig.service  loaded active exited  Control Group configuration service
  crond.service     loaded active running Command Scheduler
  ......
  mysqld.service    loaded active running MySQL Server
  network.service   loaded active running LSB: Bring up/down networking
● nginx.service     loaded failed failed  The nginx HTTP and reverse proxy server
  polkit.service    loaded active running Authorization Manager
  postfix.service   loaded active running Postfix Mail Transport Agent
  ......

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type
```

使用`systemctl status Service_Name`命令可以查看到服务的状态信息。

```bash
$ systemctl status sshd
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2020-06-30 14:34:58 CST; 1h 59min ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 1042 (sshd)
    Tasks: 1
   Memory: 6.0M
   CGroup: /system.slice/sshd.service
           └─1042 /usr/sbin/sshd -D
```

![](/img/linux/1594543604130.png)

![](/img/linux/1593518387978.png)

**第一项状态信息描述systemd是否在监控服务进程**。可能的状态值有：  
- active  
- inactive  
- activating或deactivating  
- reloading
- failed  

如果一个服务已经启动成功，它将成功加入到systemd监控队列(job queue)，此后它将受到systemd监控和管理，此时该服务的状态为`active`。

如果一个服务在启动过程中失败，比如负责服务启动的命令(ExecStart指令所指定)在启动时以非0退出状态码退出、服务启动过程耗时过久导致超时、进程崩溃等，此时该服务的状态将处于`failed`状态，表示加入监控队列失败，事实上它加入到了某个描述启动失败的队列中。

当一个服务未启动，或者服务已停止时，该服务的状态将处于`inactive`状态。

有些服务启动、停止可能会消耗一点时间，在启动或停止的过程中去查看服务的状态，看到的状态信息可能会是`activating`或`deactivating`或`reloading`。

![](/img/linux/1593518425182.png)

Active状态行的**第二项状态信息描述被监控进程的进程状态**。它可能的状态值有很多(部分机器如Ubuntu可使用`systemctl --state=help`查看支持的值)，且随着版本的更迭，支持的值也会发生变化。如下是ubuntu 18.04 systemd 237版本中查看到的值：

```
dead
start-pre
start
start-post
running
exited
reload
stop
stop-sigabrt
stop-sigterm
stop-sigkill
stop-post
final-sigterm
final-sigkill
failed
auto-restart
```

虽然有很多种状态，但用户通常只需关注其中几项常见的即可。并且这些状态的存在通常都依赖于第一项`active`状态，比如`active(running)`、`active(exited)`。

`running`状态表明被systemd监控的服务进程正在运行。

`dead`状态表明被systemd监控的进程已经死了(终结了)，systemd识别到这种状态后就会将其从监控队列中释放并不再监控它，所以它的第一项状态也会随之变为inactive。以下几种情况都会使得服务进入dead状态：  

![](/img/linux/1594543504416.png)

`exited`表明被systemd监控的服务进程已经退出了或者systemd找不到它本该要监控的进程，但是管理者systemd认为它还没有死，只是认为它暂时退出了，所以通常会结合active状态一起出现，即`active(exited)`，这种状态表明了一种意愿：服务进程已经消失了，但依然认为服务进程是活动的且正被监控。例如，当服务配置文件中配置了`RemainAfterExit=true`时，服务在退出后会进入active(exited)状态，还有多种其它可能会进入这种状态，通常来说意味着服务启动不正常(比如配置文件错误)，但并非一定代表服务是失败的，它仍然可能会正常提供服务。

`auto-restart`表明服务正在被systemd自动重启中，当服务配置文件设置了`Restart`且符合Restart规则时systemd会自动重启服务。