---
title: 熟练使用vagrant(4)：vagrant创建虚拟机时做了哪些事
p: virtual/vagrant/vagrant_do_what.md
date: 2020-10-01 17:37:33
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(4)：vagrant创建虚拟机时做了哪些事

vagrant安装的虚拟机，会自动创建默认用户vagrant，其密码默认也是`vagrant`，且该vagrant用户可以无密码执行sudo操作。
```bash
# vagrant ssh进入虚拟机后执行
[vagrant@172 ~]$ sudo cat /etc/sudoers.d/vagrant
%vagrant ALL=(ALL) NOPASSWD: ALL
```

vagrant创建的虚拟机默认禁止ssh使用密码认证的方式登录，而是使用公钥认证方式。
```bash
[vagrant@172 ~]$ sudo grep '^Password' /etc/ssh/sshd_config
PasswordAuthentication no
```

`vagrant up`在创建虚拟机时，会自动创建ssh连接所需的公钥和私钥，并将公钥分发到虚拟机vagrant用户，即公钥写入`/home/vagrant/.ssh/authorized_keys`文件中，而且每次启动虚拟机时都会做该操作。因此可以直接使用`vagrant ssh`连接到虚拟机。

`vagrant ssh`连接虚拟机时所使用的ssh配置项可通过`vagrant ssh-config`查看：
```powershell
# 和使用的provider有关
# 下面是hyper-v的虚拟机
$ vagrant ssh-config
Host generic_centos7
  HostName 172.20.249.201
  User vagrant
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile V:/vagrant_test/centos7/.vagrant/machines/generic_centos7/hyperv/private_key
  IdentitiesOnly yes
  LogLevel FATAL

# 下面是virtualbox的虚拟机
# 默认虚拟机放在virtualbox的NAT网络，
# 而vbox的NAT网络不能被宿主机访问到，因此做了端口转发
$ vagrant ssh-config c25
Host generic_centos7
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile V:/vagrant_test/centos7/.vagrant/machines/generic_centos7/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

如果用户想要使用远程连接工具(比如XShell、SecureCRT)或ssh命令连接虚拟机，在不修改sshd_config的情况下，只能使用公钥认证的方式，因此需要指定公钥认证时使用的私钥路径。vagrant生成的私钥路径在`vagrant ssh-config`的输出结果中有显示。

```powershell
# 连接hyper-v下的虚拟机
ssh.exe vagrant@172.20.249.201 -i V:/vagrant_test/centos7/.vagrant/machines/generic_centos7/hyperv/private_key

# 连接virtualbox下的虚拟机
ssh.exe vagrant@10.0.2.15 -i V:/vagrant_test/centos7/.vagrant/machines/generic_centos7/virtualbox/private_key
```

