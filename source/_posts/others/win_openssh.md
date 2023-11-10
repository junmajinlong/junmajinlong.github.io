---
title:  Windows上使用ssh服务以及坑
p: others/win_openssh_.md
date: 2021-01-06 15:20:41
tags: [Others,windows]
categories: [Others,windows]
---

# Windows上使用ssh服务以及坑

Windows上提供了openssh客户端和openssh服务端。需要先安装，需要ssh客户端(即ssh.exe命令)就安装openssh客户端，需要提供ssh服务(即sshd功能)，就安装openssh服务端。当然，可以都装上。

安装前，需要确保Windows Update服务是打开的状态。

![](/img/others/1689389982197.png)

![](/img/others/1689390015338.png)

## openssh客户端

安装好openssh客户端之后，就可以使用ssh.exe命令。它会读取用户目录下的.ssh目录下的配置文件`%userprofile%\.ssh\config`。

ssh.exe命令我使用不多，我主要使用wsl里的ssh命令。

## openssh服务端

虽然Windows提供sshd服务，但是想要正常使用，还需额外配置。

Windows上安装完openssh服务端后，如下事项须知：

- 安装openssh服务端之后，sshd的配置文件路径为`C:\ProgramData\ssh\sshd_config`，等价于Linux上的`/etc/ssh/sshd_config`。
- 安装Openssh服务端之后，会提供一个名为`OpenSSH SSH Server`的服务(真实服务名为sshd)。要想使用ssh服务端，需要启动该服务，并且每次修改完sshd的配置文件后，都要重启sshd服务使新配置生效。
- 其它主机上使用ssh连接Windows的ssh服务时，主机认证和用户认证过程的密钥路径分别为：`%userprofile%\.ssh\known_hosts`和`%userprofile%\.ssh\authorized_keys`。

![](/img/others/1689390495528.png)

启动`OpenSSH SSH Server`服务后，就可以在其它主机上通过ssh和**密码验证**的方式连接到Windows了。

## Windows OpenSSH服务无法通过公钥认证的解决方案

虽然安装OpenSSH服务端并启动`OpenSSH SSH Server`服务之后，就能通过密码认证的方式连接到Windows上了，但想要通过公钥认证方式进行连接将会失败。解决方法看最后面的[总结](#f)。

想要通过公钥认证方式连接Windows，首先要将公钥分发到Windows。

如果通过`ssh-copy-id`发送公钥想要通过公钥方式连接Windows SSH服务会出现异常，即使执行了`ssh-copy-id`之后再使用ssh命令进行连接也仍然无法公钥认证，而是让你继续用密码认证。

例如，我在远程主机Linux上使用该命令将公钥id_rsa.pub发送到Windows主机(192.168.200.1)的malong用户上：

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub malong@192.168.200.1
```

但是随后执行`ssh malong@192.168.200.1`仍然会提示使用密码。

原因在于`ssh-copy-id`想要添加公钥到`C:\Users\malong\.ssh\authorized_keys`文件中，会出现错乱或者乱七八糟的字符`: }`甚至完全没任何内容写入。

所以，**需要手动拷贝公钥内容到`C:\Users\malong\.ssh\authorized_keys`文件中**。

即使已经手动拷贝公钥到`C:\Users\malong\.ssh\authorized_keys`之后，ssh可能也依然无法使用公钥连接。原因在于Windows Openssh服务的配置文件`C:\ProgramData\ssh\sshd_config`中存在如下两行：

```
Match Group administrators
   AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

这两行的作用是：

- 如果远程主机连接的是Windows上的管理员用户(加入了管理员组的用户)，默认会使用`C:\Users\Administrator\.ssh\authorized_keys`中的公钥来做用户身份验证。
- 如果远程主机连接的是非管理员用户，则使用该用户目录下的文件`%userprofile%\.ssh\authorized_keys`中的公钥来做身份验证。

例如，`ssh malong@192.168.200.1`时，malong是一个管理员账户，即便已经手动将公钥写入`C:\Users\malong\.ssh\authorized_keys`中，也依然会通过`C:\Users\Administrator\.ssh\authorized_keys`来做身份认证，因此公钥认证会失败。

解决办法有二：

- 将公钥写入到`C:\Users\Administrator\.ssh\authorized_keys`  
- 将配置文件`C:\ProgramData\ssh\sshd_config`中的那两行注释掉，再重启`OpenSSH SSH Server`服务

<a name="f"></a>

因此，**最终解决方案的总结**是：

- 如果连接的是Windows上的非管理员用户，则手动拷贝公钥到`%userprofile%\.ssh\authorized_keys`文件即可

- 如果连接的是Windows上的管理员用户，则两种方式任选其一：
    - 1.将公钥拷贝到`C:\Users\Administrator\.ssh\authorized_keys`  
    - 2.将公钥拷贝到`%userprofile%\.ssh\authorized_keys`，然后修改sshd配置文件`C:\ProgramData\ssh\sshd_config`(如何修改看上面)，并重启`OpenSSH SSH Server`服务  


