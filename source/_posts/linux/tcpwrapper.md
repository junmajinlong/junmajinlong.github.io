---
title: tcp_wrapper网络服务的访问控制
p: linux/tcpwrapper.md
date: 2020-04-15 14:17:18
tags: Linux
categories: Linux
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# tcp\_wrapper网络服务的访问控制

`tcp_wrapper`工作在内核空间和应用程序中间的库层次中。在内核接受到数据包准备传送到用户空间时都会经过库层次，对于部分（只是部分）应用程序会被`tcp_wrapper`库阻挡下来检查，如果允许通过则交给应用程序。

`tcp_wrapper`的使用方式简单，虽然现在应用的不多，但是用来为一些传统的网络服务(比如sshd vsftpd等)做一些简单快速的配置就能达到访问控制的目的也不错。更多控制应通过防火墙来实现，当然也可以配合防火墙来使用。

本文只简单介绍以及基本使用方法，更多内容自行查阅`man hosts_access`。

## 查看是否支持tcp\_wrapper

tcp\_wrapper只会检查tcp数据包，所以称为tcp\_wrapper。但还不是检查所有类型的tcp数据包，例如httpd就不支持。是否支持，可以通过查看应用程序是否依赖于libwrap.so库文件。(路径/lib64/libwrap.so)

```
[root@mail ~]# ldd $(which sshd) | grep wrap
        libwrap.so.0 => /lib64/libwrap.so.0 (0x00007f110efb7000)
[root@mail ~]# ldd  $(which vsftpd) | grep wrap    
        libwrap.so.0 => /lib64/libwrap.so.0 (0x00007f1e73185000)
[root@mail ~]# ldd  $(which httpd) | grep wrap
```

说明sshd和vsftpd都支持tcpwrapper机制，而apache的httpd不支持。

当然上面grep不出结果只能说明不支持这样的动态链接的方式，有些应用程序可能静态编译进程序中了，如旧版本的rpc应用程序portmap。

是否将wrap功能静态编译到应用程序中，可以通过以下方式查看。

```
strings $(which portmap) | grep hosts
```

如果筛选出的结果中有hosts\_access或者/etc/hosts.allow和/etc/hosts.deny这两个文件，则说明是支持的。后两个文件正是tcpwrapper访问控制的文件。

要注意的是，如果超级守护进程xinetd被tcpwrapper控制了，则其下的瞬时守护进程都受tcpwrapper控制。

## 配置文件格式

hosts.allow和hosts.deny两个文件的语法格式是一样的，如下：

```
daemon_list:   client_list  [:options]
```

其中`daemon_list:`部分表示的是网络服务的进程名，通常包含下面几种写法。服务进程名一般和which命令的显示结果名称相同，例如此处的in.telnetd。

```
sshd:
sshd,vsftpd,in.telnetd:
ALL:
daemon@host:
```

最后一项`daemon@host`指定连接IP地址或主机域名，用于对多个IP或主机域名做独立的访问控制。例如，本机有`192.168.100.8`和`172.16.100.1`两个地址，但是只想控制从其中一个ip连接的vsftpd服务，可以写`vsftpd@192.168.100.8:`。

对于`client_list`部分，它表示的要对哪些请求网络服务的客户端做访问控制。也就是定义客户端来源。通常有下面几种写法：

```
单IP：192.168.100.8
网段：有两种写法，"172.16."和"10.0.0.0/255.0.0.0"
主机名或域匹配：fqdn或".a.com"
宏通配符：ALL、KNOWN、UNKNOWN、PARANOID
特殊关键字：EXCEPT
```

- ALL表示所有主机
- LOCAL表示和主机在同一网段的主机
- (UN)KNOWN表示DNS是否可以解析成功的
- PARANOID表示DNS正解反解不匹配的
- EXCEPT翻译为【除了】，但实际上表示的是从前面的大范围内挖一个漏洞放过这个小范围。并且EXCEPT可以嵌套使用，`a EXCEPT b EXCEPT c`将被解析为`(a EXCEPT (b EXCEPT c))`。

其中宏通配符和EXCEPT也可以用于`domain_list`部分。详细语法可以查看`man hosts_access`。

对于`options`部分，这部分是可选的，它可以是`spawn allow deny`：

- spawn：用来定义符合此规则的请求被检查时要执行的命令，必须使用绝对路径，或者明确通过`PATH`指定环境变量    
- allow：表示符合该规则的总是被允许访问  
- deny:表示符合该规则的总是被拒绝访问  

tcpwrapper涉及两个文件，这两个文件都用来定义访问控制规则：

- /etc/hosts.allow：定义允许访问的控制规则，即白名单
- /etc/hosts.deny：定义允许访问的控制规则，即黑名单  

tcpwrapper的检查顺序：`hosts.allow --> hosts.deny --> 允许(默认规则)`。

一旦检查到符合条件的规则，将停止继续搜索。因此：

- 如果在hosts.allow定义所有网络服务允许所有客户端访问，将表示最开放的访问控制
- 如果在hosts.deny中定义所有网络服务不允许所有客户端访问，将表示最严格的访问控制，定义在hosts.allow中允许的除外
- 如果在hosts.allow和hosts.deny中都没有匹配到规则，则默认被允许访问。

例如sshd仅允许172.16网段主机访问。

```
hosts.allow:
    sshd: 172.16.

hosts.deny:
    sshd: ALL
```

例如，telnet服务不允许172.16网段访问但允许172.16.100.200访问。有几种表达方式：

表达方式一：

```
hosts.allow:
    in.telnetd: 172.16.100.200

hosts.deny:
    in.telnetd: 172.16.
```

表达方式二：

```
hosts.deny:
    in.telnetd: 172.16. EXCEPT 172.16.100.200
```

不能写成如下方式，因为EXCEPT表示的是从大范围里放过小范围，而不是从小范围放过大范围。

```
hosts.allow：
    in.telnetd: 172.16.100.200 EXCEPT 172.16.
```

表达方式三：

```
hosts.allow:
    in.telnetd: ALL EXCEPT 172.16. EXCEPT 172.16.100.200
    
hosts.deny:
    in.telnetd: ALL
```

EXCEPT的最形象描述是【在其前面的范围内挖一个洞，在洞范围内的都不匹配】。所以hosts.allow中，ALL内有一个172.16的大洞，在大洞172.16中又有小洞172.16.100.200，小洞是排除在大洞之外的，大洞是排除在ALL之外的。也就是说，ALL里有个大洞是被放过留到后面继续检查(即到hosts.deny做检查)的，但这个大洞里有个小洞是不被放过的。

举个例子，EXCEPT就像挑选，没被挑选中的进入下一步挑选。例如，所有男生跟我走，但男生中体重160斤以上的留下别跟我走，但其中175斤的王麻子也跟我走。所以跟我走的是所有160斤以下的男生和175斤的王麻子，其余的男生留到下一轮做拒绝检查。

注意：被EXCEPT匹配到的仅表示不符合此条规则的检查，将会进入下一步规则的检查，而不是在hosts.allow中表示拒绝，在hosts.deny中表示允许(虽然deny中被放过之后，默认规则确实也是允许)。例如在allow中指明一个EXCEPT，当被EXCEPT的范围匹配到，表示的不是被拒绝，而是跳过allow检测进入deny的检测。

对于【:options】部分，它允许三种关键字，并且可以多次使用：

```
:ALLOW
:DENY
:spawn
```

ALLOW和DENY既可以写在allow文件，也可以写在deny文件，因此可以实现在allow文件中直接拒绝，也可以实现在deny文件中直接允许。

例如allow文件中做如下定义，表示当`example.com`域中的客户端连接到ssh服务时，将直接拒绝其访问，并记录一条日志记录。

```
sshd: .example.com  \
    : spawn /bin/echo `/bin/date` access denied>>/var/log/sshd.log \
    : deny
```

