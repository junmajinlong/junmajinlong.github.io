---
title: Linux查询端口是否被占用的几种方法
p: linux/ports_busy.md
date: 2020-07-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux查询端口是否被占用的几种方法

一个面试题，使用三种不同的方法查看8080被哪个进程占用了。通常比较熟悉的方法是netstat和lsof两种，但还有什么方法呢。

**1.netstat或ss命令**

```
netstat -anlp | grep 80
```

**2.lsof命令**

这个命令是查看进程占用哪些文件的

```
lsof -i:80
```

**3.fuser命令**

fuser命令和lsof正好相反，是查看某个文件被哪个进程占用的。Linux中，万物皆文件，所以可以查看普通文件、套接字文件、文件系统。而套接字文件就包含了端口号。比如查看22端口。

```
fuser 22/tcp -v
                     USER        PID ACCESS COMMAND
22/tcp:              root       1329 F.... sshd
                     root       1606 f.... sshd
```

**4.nmap工具**

nmap默认总是会扫描端口，要扫描本机端口，很方便。

```
nmap localhost

Starting Nmap 5.51 ( http://nmap.org ) at 2018-03-03 18:00 CST
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000020s latency).
Other addresses for localhost (not scanned): 127.0.0.1
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp

Nmap done: 1 IP address (1 host up) scanned in 0.06 seconds
```