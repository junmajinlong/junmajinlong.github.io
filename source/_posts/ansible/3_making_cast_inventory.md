---
title: 3.制定演员表：inventory
p: ansible/3_making_cast_inventory.md
date: 2021-07-11 10:37:31
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。


# 3.制定演员表：inventory

在没有对Ansible做任何配置的时候，Ansible只能通过localhost来控制本机，所以在之前的初体验中，都是让Ansible指挥本机去执行任务，这离批量控制远程主机的目标还有点远。Ansible肯定是支持指挥远程主机执行任务的，如何指定哪些远程主机执行任务呢？

让Ansible发挥强大作用的第一步是配置inventory。inventory表示清单的意思，在计算机领域里往往表示的资源清单，在Ansible中它表示主机节点清单，也是资源的一种。通过配置inventory，就可以定义哪些目标主机是可以被控制的。

这有点像是拍电影前先确定好演员表一样，演员表就是inventory，定好了演员，电影就可以正式开拍，演员则各自就位去完成剧本分派给他们的任务(之后还会发现，使用Ansible跟拍电影在很多方面是相似的)。

## 3.1 inventory文件路径

默认的inventory文件是/etc/ansible/hosts，可以通过Ansible配置文件的`inventory`配置指令去修改路径。
```shell
$ grep '/etc/ansible/hosts' /etc/ansible/ansible.cfg 
#inventory      = /etc/ansible/hosts
```

但通常不会去修改这个配置项，如果在其它地方定义了inventory文件，可以直接在ansible的命令行中使用`-i`选项去指定自定义的inventory文件。
```shell
$ ansible -i /tmp/myinv.ini ...
$ ansible-playbook -i /tmp/myinv.ini ...
```

## 3.2 配置inventory

Ansible inventory文件的书写格式遵循ini配置格式。从Ansible 2.4开始支持其它格式，比如yaml格式的inventory。此处以ini格式为例，循序渐进地介绍inventory的规则。假设所有的规则都定义在/etc/ansible/hosts文件中。

### 3.2.1 一行一主机的定义方式

Ansible默认是基于ssh连接的，所以一般情况下inventory中的每个目标节点都配置主机名或IP地址、sshd监听的端口号、连接的用户名和密码、ssh连接时的参数等等。当然，很多参数有默认值，所以最简单的是直接指定主机名或IP地址即可。

例如，在默认的inventory文件/etc/ansible/hosts添加几个目标主机：
```ini
node1
node2 ansible_host=192.168.200.28
192.168.200.31
192.168.200.32:22
192.168.200.3[2:3] ansible_port=22
```

这样定义之后，Ansible就可以控制任何一个目标主机了：
```shell
$ ansible node1 -m copy -a 'src=/etc/passwd dest=/tmp'
$ ansible 192.168.200.31 -m copy -a 'src=/etc/passwd dest=/tmp'
```

上面的inventory配置中：  

![](/img/ansible/1625968723591.png)

范围展开的方式还支持字母范围。下面都是有效的：
```
范围表示      展开结果
--------------------
a[1:3]  -->  a1,a2,a3
[08:12] -->  08,09,10,11,12
a[a:c]  -->  aa,ab,ac
```

上面示例中使用了两个主机变量`ansible_port`和`ansible_host`，它们直接定义在主机的后面，这些变量都是连接目标主机时的行为控制变量，通常它们都能见名知意。Ansible支持很多个连接时的行为控制变量，而且不同版本的Ansible的行为控制变量名称可能还不同，比如在以前版本中指定端口号的行为变量是`ansible_ssh_port`。

完整的连接行为控制变量参见官方手册：[Connecting to hosts: behavioral inventory parameters](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#connecting-to-hosts-behavioral-inventory-parameters)。下面解释几个常见的行为变量。

| Inventory变量名                  | 含义                                                         |
| :------------------------------- | :----------------------------------------------------------- |
| **ansible_host**                 | ansible连接节点时的IP地址                                    |
| **ansible_port**                 | 连接对方的端口号，ssh连接时默认为22                          |
| **ansible_user**                 | 连接对方主机时使用的主机名。不指定时，将使用执行ansible或ansible-playbook命令的用户 |
| **ansible_password**             | 连接时的用户密码                                             |
| **ansible_connection**           | 连接类型，有效值包括smart、ssh、paramiko、local、docker、winrm，默认为smart。smart表示智能选择ssh和paramiko，当SSH支持ControlPersist(即持久连接)时使用ssh，否则使用paramiko。local和docker是非基于ssh连接的方式，winrm是连接windows的插件 |
| **ansible_ssh_private_key_file** | 指定密钥认证ssh连接时的私钥文件                              |
| **ansible_ssh_common_args**      | 提供给ssh、sftp、scp命令的额外参数                           |
| **ansible_become**               | 允许进行权限提升                                             |
| **ansible_become_method**        | 指定提升权限的方式，例如可使用sudo/su/runas等方式            |
| **ansible_become_user**          | 提升为哪个用户的权限，默认提升为root                         |
| **ansible_become_password**      | 提升为指定用户权限时的密码                                   |


### 3.2.2 inventory中的普通变量

在定义inventory时，除了可以指定连接的行为控制变量，也可以指定Ansible的普通变量，以便在ansible执行任务时使用。

例如：
```ini
node1 node1_var="hello world"
node2 ansible_host=192.168.200.28
```

在ansible执行任务时可以引用普通变量：
```shell
$ ansible node1 -m debug -a 'var=node1_var'
node1 | SUCCESS => {
    "node1_var": "hello world"
}
```

### 3.2.3 主机分组

上面的示例中是每行单独定义一个主机，这样的方式虽然简单，但是极其不方便管理多个节点。

为此，Inventory支持对主机进行分组，每个组内可以定义多个主机，每个主机都可以定义在任何一个或多个主机组内。

例如：

```ini
node1 node1_var="hello world"
node2 ansible_host=192.168.200.28
192.168.200.31
192.168.200.32:22
192.168.200.3[2:3] ansible_port=22

[nginx]
192.168.200.27
192.168.200.28 ansible_password='123456'
192.168.200.29

[apache]
192.168.200.3[0:3]

[mysql]
192.168.200.27
192.168.200.29
```

这里定义了3个主机组：nginx主机组、apache主机组和mysql主机组。nginx组包含3个节点，apache主机组包含4个节点，mysql主机组包含2个节点。

需要注意的是，mysql组中的节点也同时存在于nginx组内，一个主机同时存在于多个组内是允许也是必要的功能，只有这样才能更为灵活的对各个节点进行分类管理。

有了主机组，就可以让ansible控制一个组，从而让该组内所有主机执行任务：

```shell
$ ansible apache -m copy -a 'src=/etc/passwd dest=/tmp'
```

Ansible默认预定义了两个主机组：`all`分组和`ungrouped`分组。
- `all`分组中包含所有分组内的节点  
- `ungrouped`分组包含所有不在分组内的节点  
- 这两个分组都不包含localhost这个特殊的节点  

定义了inventory之后，可以使用`ansible --list`或`ansible--playbook --list`命令来查看主机组的信息，还可以使用更为专业的`ansible-inventory`命令来查看主机组信息。

```shell
# 使用ansible或ansible-playbook列出所有主机
$ ansible -i /etc/ansible/hosts nginx --list
  hosts (3):
    192.168.200.27
    192.168.200.28
    192.168.200.29

# 使用ansible-inventory列出nginx组中的主机
$ ansible-inventory -i /etc/ansible/hosts nginx --graph
@nginx:
  |--192.168.200.27
  |--192.168.200.28
  |--192.168.200.29

# 使用ansible-inventory列出nginx组中的主机，同时带上变量
$ ansible-inventory nginx --graph --vars
@nginx:
  |--192.168.200.27
  |  |--{ansible_password = 123456}
  |  |--{ansible_port = 22}
  |--192.168.200.28
  |  |--{ansible_password = 123456}
  |  |--{ansible_port = 22}
  |--192.168.200.29
  |  |--{ansible_password = 123456}
  |  |--{ansible_port = 22}
  |--{ansible_password = 123456}

# 使用ansible-inventory列出all组内的主机
$ ansible-inventory --graph all
@all:
  |--@apache:
  |  |--192.168.200.30
  |  |--192.168.200.31
  |  |--192.168.200.32
  |  |--192.168.200.33
  |--@mysql:
  |  |--192.168.200.27
  |  |--192.168.200.29
  |--@nginx:
  |  |--192.168.200.27
  |  |--192.168.200.28
  |  |--192.168.200.29
  |--@ungrouped:
  |  |--node1
  |  |--node2

# 使用ansible-inventory以json格式列出所有主机的信息
$ ansible-inventory --list
```

### 3.2.4 主机组变量

有了主机组之后，可以直接为主机组定义变量，这样组内的所有主机都具有该变量。

```ini
[nginx]
192.168.200.27
192.168.200.28 ansible_password=123456
192.168.200.29

[nginx:vars]
ansible_password='123456'

[all:vars]
ansible_port=22

[ungrouped:vars]
ansible_port=22
```

上面`[nginx:vars]`表示为nginx组内所有主机定义变量`ansible_password='123456'`。而`[all:vars]`和`[ungrouped:vars]`分别表示为all和ungrouped这两个特殊的主机组内的所有主机定义变量。

### 3.2.5 组嵌套

Inventory还支持主机组的分组嵌套，可以通过`[GROUP:children]`的方式定义一个主机组，并在其中包含子组。

例如：

```ini
[nginx]
192.168.200.27
192.168.200.28
192.168.200.29

[apache]
192.168.200.3[0:3]

[webservers:children]
nginx
apache
```

其中webservers主机组中包含了nginx组合apache组内的所有主机。

甚至，还可以递归嵌套。例如下面的示例中，centos7分组中的webservers仍然是一个包含子组的分组。

```ini
[nginx]
192.168.200.27
192.168.200.28
192.168.200.29

[apache]
192.168.200.3[0:3]

[mysql]
192.168.200.27
192.168.200.30

[webservers:children]
nginx
apache

[centos7:children]
webservers
mysql
```

### 3.2.6 多个inventory文件

当Ansible要管理的节点非常多时，仅靠分组的逻辑可能也不足够方便管理，这个时候可以定义多个inventory文件并放在一个目录下，并按一定的命名规则为每个inventory命名，以便见名知意。

例如，创建一个名为/etc/ansible/inventorys的目录，在其中定义a和b两个inventory文件：

```
/etc/ansible/inventorys/
├── a
└── b
```

内容分别如下：

```ini
# /etc/ansible/inventorys/a的内容：
[nginx]
192.168.200.27
192.168.200.28 ansible_password='123456'
192.168.200.29

[apache]
192.168.200.3[0:3]

# /etc/ansible/inventorys/b的内容：
[mysql]
192.168.200.27
192.168.200.29

[web:children]
apache
nginx

[os:children]
web
mysql
```

现在要使用多个inventory的功能，需要将inventory指定为目录路径。

例如，Ansible配置文件将`inventory`指令设置为对应的目录：

```
inventory      = /etc/ansible/inventorys
```

或者，ansible或ansible-playbook命令使用`-i INVENTORY`选项指定的路径应当为目录。

```shell
$ ansible-inventory -i /etc/ansible/inventorys --graph all
```

执行下面的命令将列出所有主机：

```shell
$ ansible-inventory -i /etc/ansible/inventorys --graph all
```

inventory指定为目录时，inventory文件最好不要带有后缀，就像示例中的a和b文件。因为Ansible当使用目录作为inventory时，默认将忽略一些后缀的文件不去解析。需要修改配置文件中的`inventory_ignore_extensions`项来禁止忽略指定后缀(如ini后缀)的文件。

```
#inventory_ignore_extensions = ~, .orig, .bak, .ini, .cfg, .retry, .pyc, .pyo
inventory_ignore_extensions = ~, .orig, .bak, .cfg, .retry, .pyc, .pyo
```

## 3.3 动态inventory和临时添加主机

在前面介绍inventory的一大段内容中，所有的主机连接信息都是按照Ansible要求的格式直接一步到位定义在文件中，这种定义方式称为**静态的inventory**(static inventory)。

Ansible还支持更为灵活的主机收集方式：**动态inventory**。它可以将其它程序或已经存在于一个文件中(比如不是inventory支持的ini格式的文件)的主机信息动态收集起来。

不仅如此，Ansible还可以通过特殊的模块`add_host`来临时添加主机(也就是临时找来的演员😄)，通过`group_by`来临时设置主机组。这种方式添加的主机或主机组都只在内存中，只在Ansible运行时生效，Ansible退出之后就消失。

但无论是动态inventory还是`add_host`或`group_by`都不适合在此处进行演示，在后面的文章中涉及到的时候我会介绍其用法。

## 3.4 本文使用的inventory文件

下面是本文使用的inventory文件/etc/ansible/hosts的内容，在后面的文章中如果没有特别说明，也将使用此处的inventory配置。

```ini
[nginx]
192.168.200.27
192.168.200.28
192.168.200.29

[apache]
192.168.200.3[0:3]

[webservers:children]
nginx
apache

[mysql]
192.168.200.33
```