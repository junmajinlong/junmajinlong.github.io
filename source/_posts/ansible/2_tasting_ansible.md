---
title: 2.初入Ansible世界：用法概览和初体验
p: ansible/2_tasting_ansible.md
date: 2021-07-11 10:37:30
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。


# 2.初入Ansible世界：用法概览和初体验

Ansible是一个自动化管理工具，它的用法可以非常简单，只需学几个基本的模块就能完成一些简单的自动化任务，它的用法也可以非常难，不仅需要学习大量Ansible知识，还需要大量的实际应用去熟悉最优化、最完美的自动化管理逻辑。比较悲催的是Ansible的知识体系比较庞大，它的知识板块也比较零散，想要构建一个比较完善的Ansible知识体系确实稍有难度。

但无论如何，千里之行始于足下，学习任何一个新知识，总归要从最基本的用法循序渐进地深入，并辅以逐渐展开的宏观结构，加上学习过程中的不断练习，最终构建出完善的知识体系。

因Ansible的基础知识内容较多，本文先介绍最基本的概念和最基本的用法，让大家对Ansible的用法以及功能有一个基本且系统性的认识。之后的文章再逐步探讨Ansible如何应用在配置管理上。

## 2.1 测试环境并配置ssh主机互信

Ansible的作用是批量控制其它远程主机，并指挥远程主机节点做一些操作、完成一些任务。

所以在这个结构中，分为**控制节点**和**被控制节点**。Ansible是Agentless的软件，只需在控制节点安装Ansible，被控制节点一般不需额外安装任何程序，就像一个普通的命令一样，随装随用。

> Ansible的模块是用Python来执行的，且默认远程连接的方式是ssh，所以控制端和被控制端都需要有Python环境，并且被控制端需要启动sshd服务，但通常这两个条件在安装Linux系统时就已经具备了。所以使用Ansible的安装过程只有一个：在控制端安装Ansible。

在本文中，将配置如下测试环境：包括一个Ansible控制节点和7个被控制节点。后面的文章中如果没有特别说明，也都使用此处的主机环境。

| 主机描述     | IP地址         | 主机名       | 操作系统   | Ansible版本 |
| ------------ | -------------- | ------------ | ---------- | ----------- |
| control_node | 192.168.200.26 | control_node | CentOS 7.2 | Ansible 2.9 |
| node1        | 192.168.200.27 | node1        | CentOS 7.2 | -           |
| node2        | 192.168.200.28 | node2        | CentOS 7.2 | -           |
| node3        | 192.168.200.29 | node3        | CentOS 7.2 | -           |
| node4        | 192.168.200.30 | node4        | CentOS 7.2 | -           |
| node5        | 192.168.200.31 | node5        | CentOS 7.2 | -           |
| node6        | 192.168.200.32 | node6        | CentOS 7.2 | -           |
| node7        | 192.168.200.33 | node7        | CentOS 7.2 | -           |

所有主机上都已启动sshd服务并监听在22端口上。

因为有时候也会使用主机名去控制目标节点，所以这里也在control_node节点上配置了其余7个节点的DNS解析，可在control_node节点的/etc/hosts文件中加入如下内容：
```shell
$ cat >>/etc/hosts<<EOF
192.168.200.27 node1
192.168.200.28 node2
192.168.200.29 node3
192.168.200.30 node4
192.168.200.31 node5
192.168.200.32 node6
192.168.200.33 node7
EOF
```

### 2.1.1 安装最新版的Ansible

Ansible依赖于SSH协议(默认)，只需要在控制节点(即control_node主机上)安装一次Ansible即可。

各种系统下安装最新版Ansible的方式参见官方手册：[安装手册](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)。

对于RHEL系列的系统来说，配置好epel镜像即可安装最新版的Ansible。在我当前写文章的时候，最新版是Ansible 2.9版本。

```shell
$ cat >>/etc/yum.repos.d/epel.repo<<'EOF'
[epel]
name=epel repo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/epel/7/$basearch
enabled=1
gpgcheck=0
EOF
```

然后安装即可：
```shell
$ yum install ansible
```

Ansible每个版本释放出来之后，都首先提交到Pypi，所以任何操作系统，都可以使用pip工具来安装最新版的Ansible。
```shell
$ pip3 install ansible
```

但要注意，使用各系统的包管理工具(如yum)安装Ansible时自动会提供一些配置文件，如`/etc/ansible/ansible.cfg`。而使用pip安装的Ansible默认不提供配置文件。

### 2.1.2 Ansible参数补全功能

从Ansible 2.9版本开始，它支持命令的选项补全功能，它依赖于python的argcomplete插件。

安装argcomplete：
```shell
# CentOS/RHEL
yum -y install python-argcomplete

# 任何系统都可以使用pip工具安装argcomplete
pip3 install argcomplete
```

安装完成后，还需激活该插件：
```shell
# 要求bash版本大于等于4.2
sudo activate-global-python-argcomplete

# 如果bash版本低于4.2，则单独为每个ansible命令注册补全功能
eval $(register-python-argcomplete ansible)
eval $(register-python-argcomplete ansible-config)
eval $(register-python-argcomplete ansible-console)
eval $(register-python-argcomplete ansible-doc)
eval $(register-python-argcomplete ansible-galaxy)
eval $(register-python-argcomplete ansible-inventory)
eval $(register-python-argcomplete ansible-playbook)
eval $(register-python-argcomplete ansible-pull)
eval $(register-python-argcomplete ansible-vault)
```

最后，退出当前Shell重新进入，或者简单的直接执行如下命令即可：

```shell
exec $SHELL
```

然后就可以按tab一次或两次补全参数或提示参数。例如，下面选项输入到一半的时候，按一下tab键就会补全得到`ansible --syntax-check`。
```shell
$ ansible --syn
```

### 2.1.3 配置主机互信

因为Ansible默认是基于ssh连接的，所以要控制其它节点首先需要建立好ssh连接，而建立ssh连接要么需要提供密码，要么需要配置好认证方式。为了方便后文的测试，这里先配置好control_node和其它被控节点之间的主机互信。

为了避免配置主机互信过程中的交互式询问，这里使用`ssh-keyscan`工具添加主机认证信息以及`sshpass`工具(安装Ansible时会自动安装sshpass，也可以`yum -y install sshpass`安装)直接指定ssh连接密码。

1. 在control_node节点上生成密钥对：

   ```shell
   $ ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''
   ```

2. 将各节点的主机信息(host key)写入control_node的`~/.ssh/known_hosts`文件：

   ```shell
   for host in 192.168.200.{27..33} node{1..7};do
     ssh-keyscan $host >>~/.ssh/known_hosts 2>/dev/null
   done
   ```

3. 将control_node上的ssh公钥分发给各节点：

   ```shell
   # sshpass -p选项指定的是密码
   for host in 192.168.200.{27..33} node{1..7};do
     sshpass -p'123456' ssh-copy-id root@$host &>/dev/null
   done
   ```

配置好ssh的主机互信之后，就可以开始体验Ansible了。

## 2.2 Ansible初体验

![](/img/ansible/1625968437171.png)

好了，第一个关于模块的概念就先介绍这么多，该是体验一下Ansible是如何执行任务的时候了。

在控制节点上执行：
```shell
$ ansible localhost -m copy -a 'src=/etc/passwd dest=/tmp'
localhost | CHANGED => {
    "changed": true, 
    "checksum": "41a051362a32be1ec4266cc64de2c6e4ad06bc73", 
    "dest": "/tmp/passwd", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "f042f8f7c120afd8c54f437944db1108", 
    "mode": "0644", 
    "owner": "root", 
    "size": 1656, 
    "src": "/root/.ansible/tmp/ansible-tmp-1575571772.47-60786643831265/source",
    "state": "file", 
    "uid": 0
}
```

该命令的作用是拷贝本机的/etc/passwd文件到本机的/tmp目录下。

其中的ansible是一个命令，这个自不需要多做解释。除ansible命令外，后面还会用到ansible-playbook命令。

localhost参数表示ansible要控制的节点，即ansible将指挥本机执行任务。

执行任务主要是执行模块，模块的执行可以还依赖一些模块参数。在ansible命令行中，使用`-m Module`来指定要执行哪个模块，即执行什么任务，使用`-a ARGS`来指定模块运行时的参数。

本示例中的模块为copy模块，传递给copy模块的参数包含两项：  
- `src=/etc/passwd`指定源文件  
- `dest=/tmp`指定拷贝的目标路径  

初学Ansible，可能会有两个疑惑：  
1. Ansible的模块那么多，我如何知道某个功能要找哪个模块  
2. 如何知道某个模块的用法，比如参数有哪些  

初学之时，只需学习一些常用的模块，在本文以及后面的文章中会介绍一些常见模块的功能以及用法。熟悉了Ansible之后，再按需求到官方手册中去寻找：<https://docs.ansible.com/ansible/latest/modules/modules_by_category.html>。

有时候为了方便快速寻找模块，可以使用`ansible-doc -l | grep 'xxx'`命令来筛选模块。例如，想要筛选具有拷贝功能的模块：
```shell
$ ansible-doc -l | grep 'copy'
vsphere_copy          Copy a file to a VMware datastore
win_copy              Copies files to remote locations on windows hosts
bigip_file_copy       Manage files in datastores on a BIG-IP
ec2_ami_copy          copies AMI between AWS regions, return new image id
win_robocopy          Synchronizes the contents of two directories using Robocopy
copy                  Copy files to remote locations
na_ontap_lun_copy     NetApp ONTAP copy LUNs
icx_copy              Transfer files from or to remote Ruckus ICX 7000 series switches
unarchive             Unpacks an archive after (optionally) copying it from the local machine
postgresql_copy       Copy data between a file/program and a PostgreSQL table
ec2_snapshot_copy     copies an EC2 snapshot and returns the new Snapshot ID
nxos_file_copy        Copy a file to a remote NXOS device
netapp_e_volume_copy  NetApp E-Series create volume copy pairs
```

根据描述，大概找出是否有想要的模块。

找到模块后，想要看看它的功能描述以及用法，可以继续使用`ansible-doc`命令。
```shell
# 详细的模块描述手册
$ ansible-doc copy

# 只包含模块参数用法的模块描述手册
$ ansible-doc -s copy
```

再来一个示例，通过Ansible删除本地文件/tmp/passwd。需要使用的模块是`file`模块，file模块的主要作用是创建或删除文件/目录。
```shell
$ ansible localhost -m file -a 'path=/tmp/passwd state=absent'
```

参数`path=`指定要操作的文件路径，`state=`参数指定执行何种操作，此处指定为absent表示删除操作。

Ansible的很多模块都提供了一个state参数，它是一个非常重要的参数。它的值一般都会包含present和absent两种状态值(并非一定)，不同模块的present和absent状态表示的含义不同，但通常来说，present状态表示肯定、存在、会、成功等含义，absent则相反，表示否定、不存在、不会、失败等含义。

例如这里的file模块，absent状态表示递归删除文件/目录，类似于`rm -r`命令，touch状态和touch命令的功能一样，directory状态表示递归创建目录，类似于`mkdir -p`命令。

所以，在本地创建文件、创建目录的命令如下：
```shell
# 创建文件
$ ansible localhost -m file -a 'path=/tmp/a.log state=touch'

# 创建目录
$ ansible localhost -m file -a 'path=/tmp/dir1/dir2 state=directory'
```

![](/img/ansible/1625968599551.png)

例如，输出"hello world"，需要使用msg参数：
```shell
$ ansible localhost -m debug -a 'msg="hello world"'
localhost | SUCCESS => {
    "msg": "hello world"
}
```

Ansible中也支持使用变量，这里仅演示最简单的设置变量和引用变量的方式。ansible命令的`-e`选项或`--extra-vars`选项可以设置变量，设置的方式为`-e 'var1="aaa" var2="bbb"'`。

例如，设置变量后使用debug的msg参数输出：
```shell
$ ansible localhost -e 'str=world' -m debug -a 'msg="hello {{str}}"'
localhost | SUCCESS => {
    "msg": "hello world"
}
```

注意上面示例中的{% raw %} msg="hello {{str}}" {% endraw %}，Ansible的字符串是可以**不用引号**去包围的，例如`msg=hello`是允许的，但如果字符串中包含了特殊符号，则可能需要使用引号去包围，例如此处的示例出现了会产生歧义的空格。此外，要区分变量名和普通的字符串，需要在变量名上加一点标注：用{% raw %} {{}} {% endraw %}包围Ansible的变量，这其实是Jinja2模板(如果不知道，先别管这是什么)的语法。其实不难理解，它的用法和Shell下引用变量使用`$`符号或`${}`是一样的，例如`echo "hello ${var}"`。

debug模块除了msg参数，还有一个var参数，它只能用来输出变量(还包括以后要介绍的Jinja2表达式)，而且var参数引用变量的时候，不能使用{% raw %} {{}} {% endraw %}包围，因为var参数已经隐式地包围了一层{% raw %} {{}} {% endraw %}。例如：
```shell
$ ansible localhost -e 'str="hello world"' -m debug -a 'var=str'
localhost | SUCCESS => {
    "str": "hello world"
}
```

体验完Ansible模块的基本用法后，下面将简单说说Ansible中的配置文件。

## 2.3 Ansible配置文件

通过操作系统自带的包管理器(比如yum、dnf、apt)安装的Ansible一般都会提供好Ansible的配置文件/etc/ansible/ansible.cfg。

这是Ansible默认的全局配置文件。实际上，Ansible支持4种方式指定配置文件，它们的解析顺序从上到下：  
- `ANSIBLE_CFG`: 环境变量指定的配置文件  
- `ansible.cfg`: 当前目录下的ansible.cfg  
- `~/.ansible.cfg`: 家目录下的ansible.cfg  
- `/etc/ansible/ansible.cfg`：默认的全局配置文件  

Ansible配置文件采用ini风格进行配置，每一项配置都使用`key=value`的方式进行配置。

例如，下面是我从默认的/etc/ansible/ansible.cfg中截取的`[defaults]`段落和`[inventory]`段落的部分配置信息。
```ini
[defaults]
# some basic default values...
#inventory      = /etc/ansible/hosts
#library        = /usr/share/my_modules/
#module_utils   = /usr/share/my_module_utils/
#remote_tmp     = ~/.ansible/tmp
#local_tmp      = ~/.ansible/tmp
#plugin_filters_cfg = /etc/ansible/plugin_filters.yml
#forks          = 5

[inventory]
#enable_plugins = host_list, virtualbox, yaml, constructed
#ignore_extensions = .pyc, .pyo, .swp, .bak, ~, .rpm, .md, .txt, ~, .orig, .ini, .cfg, .retry
#ignore_patterns=
#unparsed_is_failed=False
```

目前没必要去了解配置文件里的每项指令的含义，在后面需要的时候，我会介绍涉及到的相关配置项的含义。
