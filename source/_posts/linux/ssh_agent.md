---
title: ssh代理ssh-agent
p: linux/ssh_agent.md
date: 2021-03-15 14:17:18
tags: Linux
categories: Linux
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

SSH系列文章：  
[SSH基础：SSH和SSH服务](/linux/ssh)  
[SSH转发代理：ssh-agent用法详解](/linux/ssh_agent)  
[SSH隧道：端口转发功能详解](/ssh_port_forward)  

------

# ssh代理ssh-agent

## 使用ssh-agent之前

使用ssh公钥认证的方式可以免去ssh客户端(如ssh命令、xshell等)连接远端主机sshd时需要输入对方用户密码的问题。

但如果执行ssh命令所在的主机上保存了多套秘钥且将各公钥分发给了不同的远端主机，这时即使使用了公钥认证，也依然需要输入密码，因为ssh客户端不知道要读取哪个私钥去和该远端主机上的公钥配对。

看下面这张图描述的情况：

![](/img/linux/733013-20190306141627208-1059807246.png)


上面描述的情形是这样的：ssh客户端要管理web server群，还要管理mysql server群，ssh客户端要为这两个群内的主机使用不同的密钥对。例如要连接web server群内的主机，使用`~/.ssh/id_rsa_1`这一套秘钥，连接mysql server群内的主机，使用`~/.ssh/id_rsa_2`这一套秘钥。

于是，将`id_rsa_1.pub`分发给web server群内的每个主机，将`id_rsa_2.pub`分发给mysql server群内的每个主机：
```
$ ssh-copy-id -i ~/.ssh/id_rsa_1.pub root@webserver1
$ ssh-copy-id -i ~/.ssh/id_rsa_1.pub root@webserver2
$ ssh-copy-id -i ~/.ssh/id_rsa_1.pub root@webserver3
$ ssh-copy-id -i ~/.ssh/id_rsa_2.pub root@mysqlserver1
$ ssh-copy-id -i ~/.ssh/id_rsa_2.pub root@mysqlserver2
$ ssh-copy-id -i ~/.ssh/id_rsa_2.pub root@mysqlserver3
```

这一切进行的都很欢乐，但是一连接，发现还是要密码：
```
$ ssh root@webserver1
快输入 root@webserver's 密码:？？？

$ ssh root@mysqlserver1
快输入 root@mysqlserver's 密码:？？？
```
这是因为ssh客户端连接webserver1的时候，除了默认会读取的规范私钥文件`id_rsa`(或其它秘钥类型)外，不会自动读取任何一个文件，同理连接mysqlserver1也一样。

正确的连接方式是指定连接时使用哪个私钥去配对：
```
$ ssh -i ~/.ssh/id_rsa_1 root@webserver1
$ ssh -i ~/.ssh/id_rsa_2 root@mysqlserver1
```

好欢乐，终于连上了。但是真恶心，还要指定连接私钥。

不仅如此，如果私钥是加密(passphrase)的，指定私钥的时候还得输入这个passphrase密码。😭

## ssh-agent是干什么的

程序员很不满意这样的连接方式，于是创造了一个中间的私钥管理者ssh-agent：你不是不知道怎么配对吗，我帮你配。而且这个功能对于程序员来说，so easy。

我在[ssh身份认证阶段](/linux/ssh#ssh_auth)中解释过，ssh认证的过程其实是客户端(ssh命令端)读取自己的私钥并推导出指纹发送给服务端(sshd端)，服务端也使用自己保存的公钥推导出指纹进行对比，如果指纹相同说明服务端的公钥和客户端的私钥是配对的。如下是公钥、私钥的指纹计算方式：
```
$ ssh-keygen -l -f ~/.ssh/id_rsa_2
2048 2c:e9:70:a8:f5:8d:87:9f:8c:de:cf:cf:14:f4:40:52  root@xuexi.longshuai.com (RSA)
$ ssh-keygen -l -f ~/.ssh/id_rsa_2.pub 
2048 2c:e9:70:a8:f5:8d:87:9f:8c:de:cf:cf:14:f4:40:52  root@xuexi.longshuai.com (RSA)
```

这个密钥身份认证的过程是理解ssh-agent的关键。

ssh-agent的主功能大概如下图描述：

![](/img/linux/733013-20190306154615731-170262604.png)

使用ssh-agent后，可以通过ssh-add命令向ssh-agent注册本机的私钥，ssh-agent会自动推导出这个私钥的指纹(实际上是ssh-add计算的)保存在自己的小本本里(内存)，以后只要ssh连接某主机(某用户)，将自动转发给ssh-agent，ssh-agent将自动从它的小本本里查找私钥的指纹并将其发送给服务端(sshd端)。如此一来，ssh客户端就无需再指定使用哪个私钥文件去连接。

总的看上去，ssh-agent的角色就是帮忙存储、查找并发送对应的指纹而已，也就是说它是一个连接的转发人，扮演的是一个代理的角色。

ssh并非一定支持ssh-agent转发的连接。要使用ssh-agent的转发功能，需要在sshd_config中开启`AllowAgentForwarding`选项，在ssh_config中开启`ForwardAgent`选项。

## 使用ssh-agent和ssh-add

先将ssh-agent运行起来：
```
$ ssh-agent
SSH_AUTH_SOCK=/tmp/ssh-GiORRAqMXEFt/agent.28161; export SSH_AUTH_SOCK;
SSH_AGENT_PID=28162; export SSH_AGENT_PID;
echo Agent pid 28162;
```
输出结果中明确说明了导出了几个环境变量，同时也可以知道ssh-agent使用Unix Domain套接字的方式监听在本机上以及其pid为28162。到时候可以直接使用`kill 28162`杀掉这个进程，更直接的方式是使用`ssh-agent -k`，它会根据当前的环境变量`SSH_AGENT_PID`来杀进程。

但是很不幸，上面的ssh-agent尽管运行成功了，但是那两个环境变量并没有导出，上面的结果只是显示给用户看的。所以更多时候，会使用eval来执行ssh-agent，使得这些环境变量也被导出：
```
# 先杀掉刚才的ssh-agent
$ kill 28162

$ eval `ssh-agent`
Agent pid 28173
```

ssh客户端上运行ssh-agent后，就可以使用ssh-add命令向ssh-agent添加私钥(如果私钥使用了passhprase密码，则要求输入一次密码)。
```
$ ssh-add
$ ssh-add ~/.ssh/id_rsa_1
$ ssh-add ~/.ssh/id_rsa_2
```

默认会添加`~/.ssh/`下的所有私钥类文件(`~/.ssh/id_rsa, .ssh/id_dsa, ~/.ssh/id_ecdsa, ~/.ssh/id_ed25519, ~/.ssh/identity`)，也可以指定要添加的文件。正如上面给出的示例。

ssh-add命令是查找当前环境变量`SSH_AUTH_SOCK`的值并发送添加请求给对应套接字的，所以这个套接字环境变量非常重要。

之后再使用公钥认证就可以直接连接到目标主机：
```
$ ssh root@webserver1
$ ssh root@webserver2
$ ssh root@webserver3

$ ssh root@mysqlserver1
$ ssh root@mysqlserver2
$ ssh root@mysqlserver3
```

## ssh-agent的痛点和解决方案

ssh-agent的工作是依赖于环境变量`SSH_AUTH_SOCK`和`SSH_AGENT_PID`的，**不同用户，不同终端，只要没有和这两个环境变量配对的ssh-agent，这个agent进程就不可使用。要想使用某个agent，就必须在自己的shell中先设置好这两个环境变量**。

另外需要注意的是，以``eval `ssh-agent` ``启动的方式会直接让ssh-agent工作在后台，它自己会独立成自己的进程组，其父进程或终端退出后它仍然会挂靠在pid=1的init/systemd下。而ssh-agent的工作是依赖于环境变量`SSH_AUTH_SOCK`和`SSH_AGENT_PID`的，shell或终端退出后这两个环境变量就消失了，使得之前运行的ssh-agent被多余地保留在后台。

也就是说，每次都要拿个小本本把这两个环境变量记下来，然后在每个shell下设置好这两个环境变量才能使用这个ssh-agent，这种恶心的行为在多个终端之间交互是非常痛苦的。

其实，我们自己完全可以根据已有的ssh-agent推导出这两个环境变量。如果ssh-agent进程还存在，那么在/tmp/目录下一定有对应的套接字文件(除非启动ssh-agent时自定义了套接字路径)。
```
$ tree /tmp/ssh* 
/tmp/ssh-q3tM0FzpCcdU
└── agent.28629
/tmp/ssh-SkKrrkK6qLDq
└── agent.28817
```

于是环境变量`SSH_AUTH_SOCK`的值已经得到了。再根据`agent.<ppid>`中的ppid值，将其加上1，就是其子进程ssh-agent进程的PID值。例如，想要使用上面`agent.28629`对应的ssh-agent，那么设置：
```
export SSH_AUTH_SOCK=/tmp/ssh-q3tM0FzpCcdU/agent.28629
export SSH_AGENT_PID=28630
```

我们完全可以自己写一个脚本来自动化获取并设置这些环境变量。

不过已经有人用shell脚本提供好了更完善的解决方案：<https://github.com/wwalker/ssh-find-agent>。

下载好其中的ssh-find-agent.sh脚本，并放到某个目录下。例如直接放在`/etc/profile.d`目录下，然后给执行权限：
```
$ wget https://raw.githubusercontent.com/wwalker/ssh-find-agent/master/ssh-find-agent.sh -O /etc/profile.d/ssh-find-agent.sh
$ chmod +x /etc/profile.d/ssh-find-agent.sh
```

以后，只要在新的终端上，或者和ssh-agent进程失联的shell中执行（如果在旧终端上，需要先source该shell脚本）：
```
$ ssh-find-agent -a
```

它会自动寻找到第一个ssh-agent进程并配置好相关环境变量：
```
$ ssh-find-agent -a
$ echo $SSH_AUTH_SOCK
/tmp/ssh-SkKrrkK6qLDq/agent.28817
$ echo $SSH_AGENT_PID
28818
```

不给任何参数的ssh-find-agent函数会列出当前能找到的ssh-agent进程以及相关环境变量和编号。
```
$ ssh-find-agent
export SSH_AUTH_SOCK=/tmp/ssh-q3tM0FzpCcdU/agent.28629  #1)         
export SSH_AUTH_SOCK=/tmp/ssh-SkKrrkK6qLDq/agent.28817  #2)  
```

使用ssh-find-agent的-c选项可以手动选择使用哪个ssh-agent进程。

如果不知道当前是否有ssh-agent，可以使用下面的方式：
```
$ ssh-find-agent -a || eval $(ssh-agent) > /dev/null
```

这样就会在找不到的时候自动开启一个ssh-agent。

除了上面的方法，还可以直接使用另一种管理工具：keychain。
```
$ yum install -y keychain
```

keychain能很好的管理ssh-agent，可以去查看下如何使用。


## 管理ssh-agent中的私钥

ssh-agent命令的选项：
```
-a bind_address
指定ssh-agent运行时绑定的Unix Domain套接字路径，默认是`$TMPDIR/ssh-xxx/agent.<ppid>`

-c
-s
指定ssh-agent运行时输出的内容(那些环境变量)是csh还是bash格式的语句

-d
调试模式

-k
杀掉`SSH_AGENT_PID`环境变量指定的pid进程

-t life
指定ssh-agent中私钥(指纹)的有效期。默认单位为秒，可以指定m(分钟)、h(小时)、d(天)、w(周)。如果不指定，则永久有效。该有效期可以被ssh-add指定的有效期选项覆盖
```

ssh-add命令的选项（部分选项）：
```
-D
删除ssh-agent中所有私钥(指纹)

-d key_file
删除指定私钥

-L
列出agent当前主机上所有公钥参数，即公钥文件中的内容

-l
列出agent当前已保存的指纹信息

-t
设置私钥(指纹)的有效期。默认单位为秒，可以指定m(分钟)、h(小时)、d(天)、w(周)

-x
使用一个密码将agent锁起来(lock)，锁起来的agent将不再提供任何服务

-X
解锁agent
```

