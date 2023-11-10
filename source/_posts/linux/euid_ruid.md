---
title: 理解Effective UID(EUID)和Real UID(RUID)
p: linux/euid_ruid.md
date: 2020-09-27 18:20:42
tags: Linux
categories: Linux
---

# 理解Effective UID(EUID)和Real UID(RUID)


每个文件都有其所有者和所属组以及权限位属性：

```
$ ls -l /etc/hosts
-rw-r--r-- 1 root root 619 Oct  1 08:51 /etc/hosts
```

文件上的用户信息和权限信息是文件属性，是静态的，用来限制谁能对该文件执行什么操作。

而对文件执行操作最终是由进程完成的，所以进程执行操作时拥有的权限才是决定能否对文件执行操作的标准。

Effective UID(EUID)称为有效用户ID，Real UID称为实际用户ID。还有对应的EGID和RGID。它们都是针对进程而非针对文件的，也就是说，**它们是进程的属性，而非文件的属性**。

**进程在执行某些涉及权限的操作时，内核将使用进程的【有效用户/组ID】作为凭证来判断是否有权限执行对应操作**。例如，rm命令删除文件a.txt时，内核将使用rm进程的有效用户ID和有效组ID和a.txt文件的用户ID和组ID做比较，进而再比较对应的权限位，从而判断rm进程是否有权限删除a.txt。

既然涉及到权限的操作都以有效ID作为依据，那**实际ID有什么作用**？**RUID和RGID用于确定进程所属的用户和组，主要用于判断是否有权限向进程发送信号：对于非特权用户，如果发送信号的进程A的RUID和目标进程B的RUID，则进程A可以发送信号。由于子进程会继承父进程的实际ID，所以父子进程的RUID相等，父子进程可互相发信号**。


用户执行程序时，默认将当前用户或指定的用户(如sudo方式)设置为该进程的有效用户ID和实际用户ID。此时它们是相等的。

进程内部可通过代码修改进程的有效用户ID。例如，以root身份运行的程序，初始时其有效用户ID为0(即root)，具有特权，之后在程序内部修改其有效用户ID为nobody，修改后该进程将失去大量权限。

也可以在程序文件上设置suid，使得该程序在执行时，其有效用户ID被设置为程序文件的UID。

```
# /bin/passwd设置了SUID，执行passwd时，passwd进程的有效用户ID将设置为root
$ ls -l /bin/passwd  
-rwsr-xr-x 1 root root 68208 Apr 16 20:36 /bin/passwd
```

举个例子来分析。用户user1执行/bin/app删除a.txt文件时的身份设置过程：只考虑用户不考虑组。
- 1.获取/bin/app权限信息，判断user1是否有权限执行该程序，若有权，则exec加载成功。
- 2.如果app文件未设置SUID，则设置进程的有效用户ID和实际用户ID都为user1。
- 3.如果app文件设置了SUID，则设置进程的有效用户ID为app文件的用户ID。
- 4.app程序内部可能也会通过系统调用修改有效用户ID。
- 5.app进程执行删除a.txt文件代码时，内核将比较此时app进程的有效ID和a.txt文件的用户ID，从而判断权限位

此外还有保存的UID(Saved UID)和保存的GID(Saved GID)，保存的ID是保存有效ID的。在程序启动的时候，会将保存的ID设置为有效ID相同的值，之后如果修改有效ID，保存ID将不会改变。

通过ps命令可以查看进程的euid、ruid和suid。

```bash
$ ps -o pid,euid,ruid,suid,command 
    PID  EUID  RUID  SUID COMMAND
  26481  1000  1000  1000 -bash
  29120  1000  1000  1000 ps -o pid,euid,ruid,suid,command
```

## 服务类程序权限配置的惯用方式

对于服务类程序，经常允许用户去配置以哪个User和哪个Group(通常配置为非特权身份以求安全)去运行服务。对于这类服务进程，惯用的手段是：

- 程序启动时，先执行一些需要特权才能执行的操作(如创建/var/run/xx.pid)。  
- 然后备份当前具有特权的EUID。  
- 修改有效GID、实际GID和辅助组(除了有效ID，辅助组也会影响内核检查权限)。  
  - 之所以在修改EUID之前修改组，因为修改EUID之后就丢失了权限，无法再修改组。  
- 修改EUID为非特权用户(比如nobody，一般可在配置文件中指定)。  
  - 修改EUID后，进程将丢失大量权限，可保证进程是安全的。  
- 在进程退出时，恢复EUID为具有特权的用户身份。  
  - 之所以要恢复特权EUID，是可能需要做一些要求特权的清理善后操作，例如删除/var/run/xx.pid。

例如，下面是一段perl实现服务类程序的部分代码：

```perl
use constant PIDPATH => '/var/run'; # 只有root可操作

sub init_server{
  my ($user, $group);
  ($pidfile, $user, $group) = @_;
  $pidfile = shift || getpidfilename();
  my $fh = open_pid_file($pidfile);   # 只有root可创建、删除、打开/var/run下的pid文件
  become_daemon();
  print $fh $$;
  close $fh;
  init_log();   
  # 在change_privileges之前先执行一些需要特权的操作，之后修改为非特权EUID
  change_privileges($user, $group) if defined $user && defined $group;
  return $pid = $$;
}

sub change_privileges{
  my ($user, $group) = @_;
  my $uid = getpwnam($user) or die "Can't get uid for $user\n";
  my $gid = getgrnam($group) or die "Can't get gid for $group\n";
  # 和UID/GID相关的几个perl变量
  #   $>: 有效UID
  #   $): 有效GID
  #   $<: 实际UID
  #   $(: 实际GID
  $) = "$gid $gid";  # 修改EGID，同时将辅助组也设置为EGID，表示辅助组设置为空
  $( = $gid;      # 修改RGID
  $> = $uid;      # 修改EUID，从此开始将丢失权限
}

END{
  $> = $<; # 恢复特权EUID，以便删除要求权限的pid文件
  unlink $pidfile if defined $pid and $$ == $pid
}
```

**参考资料**：

- 《Linux-UNIX系统编程手册（上、下册）》第九章 进程凭证。
- 《Perl网络编程》第十四章 安全的服务器