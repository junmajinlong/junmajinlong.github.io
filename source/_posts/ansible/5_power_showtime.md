---
title: 5.Ansible力量初显：批量初始化服务器
p: ansible/5_power_showtime.md
date: 2021-07-11 10:37:33
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。

# 5.Ansible力量初显：批量初始化服务器

## 5.1 批量初始化服务器案例

既然前面已经学完了必要理论基础知识：inventory和playbook，现在或许是自己上手来试试Ansible如何进行批量配置管理的好机会。

在本文中，将借助批量初始化服务器的案例来演示如何使用Ansible playbook，以便让各位熟悉playbook的编写。

本文要实现的初始化配置目标包括：  
- 1.Ansible配置SSH密钥认证  
- 2.Ansible远程配置主机名  
- 3.Ansible控制远程主机互相添加DNS解析记录  
- 4.Ansible配置远程主机上的yum镜像源以及安装一些软件  
- 5.Ansible配置远程主机上的时间同步  
- 6.Ansible关闭远程主机上selinux  
- 7.Ansible配置远程主机上的防火墙  
- 8.Ansible远程修改sshd配置文件并重启sshd，使其更安全  

在本文最后，我会将所有这些案例的playbook集合到单个playbook文件中，同时还会藉此引入Ansible组织多个任务文件的概念，为下一篇文章做个简单的铺垫。

本文会介绍一些涉及到的模块用法，但模块用法不是本文的重点，也不是学习Ansible的重点，因为学习模块的用法很简单，甚至我觉得学会如何去找模块比学会模块的用法还重要，所以各位在学习Ansible的时候千万不要迷失在大量学习Ansible模块中。本文的重点是借助这些案例来解释Ansible提供的一些重要机制，从而学会如何解决一类问题，这才是真正精通Ansible必不可少的能力。

因为本文不仅仅只是写这几个案例就完事(其实这些案例的playbook内容都很短)，期间会介绍很多相关的重要知识点，所以篇幅较大，希望各位记好并初步整理好自己的笔记。之所以说是初步整理，是因为这里涉及的知识点都来自于案例，这对理解它们的用法很有帮助，但缺点是出现的比较突兀、比较零散，不方便系统性整理，难免会觉得东一榔头，西一棒子，很凌乱。不过不用担心，在后面我会专门写一篇进阶型的文章，到时候会系统性地介绍它们。

废话说了一大堆，进入正题吧。

下面是在/etc/ansible/hosts文件中新添加的两个节点，主机组名为new，它们将作为本文的测试节点：
```ini
[new]
192.168.200.34
192.168.200.35
```

## 5.2 Ansible配置SSH密钥认证

当要使用Ansible去控制管理一个新节点时，第一步肯定是配置控制端(即Ansible端)和这个被控制节点之间的SSH密钥认证，让它们之间互信，不用在建立ssh连接的时候还需要交互输入或手动去指定连接密码。

在之前的Ansible初体验文章中已经给出了一种比较简单的方案：  
- 使用`ssh-keyscan`命令解决主机认证的交互问题(即不用输入yes/no)  
- 使用`sshpass -pPASSWORD`命令解决`ssh-copy-id`分发公钥时要求输入密码的问题  

下面是整个过程：
```shell
# 在control_node节点上生成密钥对
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''

for host in 192.168.200.{27..33};do
  # 将各节点的主机信息(host key)写入control_node的
  # `~/.ssh/known_hosts`文件
  ssh-keyscan $host >>~/.ssh/known_hosts 2>/dev/null

  # 将control_node上的ssh公钥分发给各节点：
  sshpass -p'123456' ssh-copy-id root@$host &>/dev/null
done
```

也可以使用Ansible去实现这样的功能。

### 5.2.1 方案一：命令行改写为playbook

上面使用的是shell命令方案，也可以直接使用Ansible playbook来实现完全相同的步骤，而且写进playbook对统一初始化配置的操作是很值得的。

playbook内容如下：
```yaml
---
- name: configure ssh connection
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: configure ssh connection
      shell: |
        ssh-keyscan {{inventory_hostname}} >>~/.ssh/known_hosts
        sshpass -p'123456' ssh-copy-id root@{{inventory_hostname}}
```

执行该playbook，然后测试。
```shell
$ ansible-playbook sshkey.yaml
$ ssh 192.168.200.35
```

这个方案是我临时加进来的，目的是为了给各位介绍`connection: local`指令的作用，顺带着介绍一下涉及到的`shell`模块。所以，请各位开始记笔记。

首先要解释的是{% raw %} {{inventory_hostname}} {% endraw %}，其中{% raw %} {{}} {% endraw %}在前面的文章中已经解释过了，它可以用来包围变量，在解析时会进行变量替换。而这里引用的变量为`inventory_hostname`，该变量表示的是当前正在执行任务的目标节点在inventory中定义的主机名。例如，对于本示例中两个执行任务的节点来说，该变量的值分别是`192.168.200.34`和`192.168.200.35`。

然后再分别介绍相关模块和`connection: local`。

#### command、shell、raw和script模块

Ansible用于配置管理，但是它最基本的功能得提供：在目标主机上执行命令。

command、shell、raw和script这四个模块的作用和用法都类似，都用于远程执行命令或脚本：  

![](/img/ansible/1625969018806.png)

例如：
```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks: 
    - name: use command module
      command: date +"%F %T"

    - name: use shell module
      shell: date +"%F %T" | cat >/tmp/date.log

    - name: use raw module
      raw: date +"%F %T"
```

如果想要看到命令的输出结果，可在执行playbook的时候加上一个`-v`选项：
```shell
$ ansible-playbook -v module.yaml
```

如果要执行的命令有多行，根据之前文章中介绍的YAML语法，可以换行书写。例如：
```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks: 
    - name: use shell module
      shell: |
        date +"%F %T" | cat >/tmp/date.log
        date +"%F %T" | cat >>/tmp/date.log
```

如果要执行的是一个远程主机上已经存在的脚本文件，可以使用shell、command或raw模块，但有时候脚本是写在Ansible控制端的，可以先将它拷贝到目标主机上再使用shell模块去执行这个脚本，但更佳的方式是使用script模块，script模块默认就是将Ansible控制端的脚本传到目标主机去执行的。此外，script模块和raw模块一样，不要求目标主机上已经装好python。

例如，Ansible端有一个shell脚本文件/tmp/hello.sh，它还接收一个参数，内容如下：
```shell
#!/bin/bash

echo hello world: $1
```

playbook内容如下：
```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks: 
    - name: use script module
      script: /tmp/hello.sh HELLOWORLD
```

Ansible中绝大多数的模块都具有幂等特性，意味着执行一次或多次不会产生副作用。但是shell、command、raw、script这四个模块**默认不满足幂等性**，所以操作会重复执行，但有些操作是不允许重复执行的。例如mysql的初始化命令`mysql_install_db`，逻辑上它应该只在第一次配置的过程中初始化一次，其他任何时候都不应该再执行。所以，每当使用这4个模块的时候都要在心中想一想，重复执行这个命令会不会产生负面影响。

当然，除了raw模块外，其它三个模块也提供了实现幂等性的参数，即creates和removes：  
- `creates`参数: 当指定的文件或目录存在时，则不执行命令  
- `removes`参数: 当指定的文件或目录不存在时，则不执行命令  

例如：
```yaml
---
- name: use some module
  hosts: new
  gather_facts: false
  tasks: 
    # 网卡配置文件不存在时不执行
    - name: use command module
      command: ifup eth0
      args: 
        removes: /etc/sysconfig/network-scripts/ifcfg-eth0

    # mysql配置文件已存在时不执行，避免覆盖
    - name: use shell module
      shell: cp /tmp/my.cnf /etc/my.cnf
      args: 
        creates: /etc/my.cnf
```

显然，使用`removes`或`creates`参数之后，就可以实现幂等性，保证命令不会重复执行。

最后，这四个模块都不限于执行shell命令或shell脚本，可以通过`executable`参数指定其它解释器，比如expect执行expect脚本、perl解释器执行perl脚本等等：
```yaml
- name: Run a perl script
  script: /some/local/script.pl
  args:
    executable: perl
```

#### connection和delegate_to指令

默认情况下，Ansible使用ssh去连接远程主机，但实际上它提供了多种插件来丰富连接方式：smart、ssh、paramiko、local、docker、winrm，默认为smart。

smart表示智能选择ssh和paramiko(paramiko是Python的一个ssh协议模块)，当Ansible端安装的ssh支持ControlPersist(即持久连接)时自动使用ssh，否则使用paramiko。local和docker是非基于ssh连接的方式，winrm是连接Windows的插件。

可以在命令行选项中使用`-c`或`--connection`选项来指定连接方式：
```shell
$ ansible          nginx -c smart -m XXX -a 'YYY'
$ ansible-playbook nginx -c smart -m XXX -a 'YYY'
```

或者在playbook的play级别或task级别上指定连接方式，定义在task上时表示该task使用所指定的连接方式：
```yaml
---
- hosts: nginx
  connection: smart
  ...
  tasks: 
    - copy: src=/etc/passwd dest=/tmp
      connection: local
```

此外，inventory中也可以通过连接的行为变量`ansible_connection`指定连接类型：
```
192.168.200.29 ansible_connection="smart"
```

通常不需要关注连接类型，但是一种特殊的连接方式是`local`，它表示在Ansible端本地执行任务。例如：
```yaml
---
- name: exec task at local
  hosts: new
  gather_facts: false
  connection: local
  tasks: 
    - name: task1
      debug: 
        msg: "{{inventory_hostname}} is executing task"
    - name: task2
      shell: touch /tmp/task1.txt
      args:
        creates: /tmp/task1.txt
```

执行该playbook，注意观察输出结果：
```shell
$ ansible-playbook conn.yaml
......
ok: [192.168.200.34] => {
    "msg": "192.168.200.34 is executing task"
}
ok: [192.168.200.35] => {
    "msg": "192.168.200.35 is executing task"
}
......
```

不难发现/tmp/task1.txt文件是创建在Ansible端本地，而不是在目标主机上创建。

![](/img/ansible/1625969119566.png)

例如：

```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks: 
    - name: task1
      debug: 
        msg: "{{inventory_hostname}} is executing task"
      delegate_to: 192.168.200.35
```

也许这里会有一个更深的疑惑，为什么要使用`connection: local`或`delegate_to`，而不直接在被委托节点上执行任务？

主要原因在于委托执行时获取了目标节点的上下文环境。正如委托给Ansible端本地执行时，它是可以获取到目标节点信息的，而如果直接通过`hosts: localhost`指定在本地执行，则除了localhost外没有任何其它目标节点，也就无法获取到这些节点的信息，比如无法获取它们的IP地址。

花了这么大的篇幅，我想告诉各位的是，`delegate_to`是一个很重要的指令，在后面的文章中将会看到它在不少场景下都大放异彩，比如负载均衡(比如haproxy)管理后端节点的场景。

现在，再回头来看看配置SSH主机互信的playbook，我想应该很容易理解了。
```yaml
---
- name: configure ssh connection
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: configure ssh connection
      shell: |
        ssh-keyscan {{inventory_hostname}} >>~/.ssh/known_hosts
        sshpass -p'123456' ssh-copy-id root@{{inventory_hostname}}
```

### 5.2.2 方案二：使用authorized_key模块

Ansible提供了一个`authorized_key`模块，它可以用来分发SSH公钥。

但需注意，`authorized_key`模块并不负责主机认证的阶段，所以需要额外去处理主机认证阶段，有三种处理方式：  
1. 使用`ssh-keyscan`命令将主机信息添加到Ansible端的`~/.ssh/known_hosts`中  
2. 使用Ansible提供的`known_hosts`模块添加主机信息  
3. 禁止主机认证的阶段  

禁止主机认证阶段的方式有多种，比如去ssh或Ansible的配置文件中禁止主机认证(两种配置都可以，因为Ansible默认是基于ssh进行连接的)，以Ansible配置文件为例，设置如下项即可：
```
host_key_checking = False
```

再比如下面三种单次禁用的方式：
```
# 1.在ansible或ansible-playbook命令行中指定跳过主机认证阶段
ansible --ssh-extra-args '-o StrictHostKeyChecking=no' XXX
ansible-playbook --ssh-extra-args '-o StrictHostKeyChecking=no' XXX

# 2.在inventory中通过ansible_ssh_extra_args设置ssh连接选项
192.168.200.34 ansible_ssh_extra_args="-o StrictHostKeyChecking=no"

# 3.使用环境变量的方式指定跳过主机认证
ANSIBLE_HOST_KEY_CHECKING=False ansible XXX
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook XXX
```

最方便的当然是配置`host_key_checking = False`或使用`ssh-keyscan`命令，这样的操作是一劳永逸的。使用Ansible的`known_hosts`模块也可以，但我个人不习惯它，各位如果有Ansible基础，可以去了解下如何使用该模块。

解决了主机认证的问题，现在使用`authorized_key`分发公钥到目标主机上。由于这一次分发的任务仍然是要建立SSH连接的，所以需要提供SSH连接的密码，当这次分发完成之后，以后Ansible就不再需要连接密码了。

ssh连接密码可以指定在inventory文件中，比如定义在`all`组中：
```
[all:vars]
ansible_password="123456"
```

这样所有目标主机都使用该密码去连接，当分发完成后，再手动将inventory中的该变量删除。

不过，由于密码仅在此处分发密钥时使用一次，如果所有分发目标的连接密码都相同，我想更方便的是在执行playbook时使用`-k`选项要求单次交互式输入密码，而不是将其定义在inventory中并在分发完成后删除。稍后将看到相关用法。

准备工作都完成后，开始写使用`authorized_key`模块分发公钥的playbook，内容如下：
```yaml
---
- name: configure ssh connection
  hosts: new
  gather_facts: false
  tasks:
    - authorized_key:
        key: "{{lookup('file','~/.ssh/id_rsa.pub')}}"
        state: present
        user: root
```

执行该playbook，主机加上了`-k`选项，它会提示用户输入ssh连接密码。如果所有目标主机的密码都相同，则只需输入一次即可：

```shell
$ ansible-playbook -k anth_key.yml
SSH password: 

PLAY [configure ssh connection] ***********

TASK [authorized_key] *********************
changed: [192.168.200.34]
changed: [192.168.200.35]
......
```

然后验证：
```shell
$ ansible new -a 'echo haha'
```

对于上面的playbook，需要说明的是`authorized_key`模块的参数。

user参数和key参数是必须的，key指定公钥字符串。user参数表示将密钥分发给目标主机上的哪个用户，默认会将公钥写入目标主机的`/home/USERNAME/.ssh/authorized_keys`文件中，默认情况下会创建缺失的目录(比如`.ssh`目录)并设置好权限。

这里还使用了`state`参数，state参数值为present时，表示如果对方文件中已有完全相同的公钥信息，则不写入，否则写入。如果值为absent，则表示删除目标节点上与本次待分发公钥完全相同的数据。总结起来就是：  
- `present`：保证目标节点上会保存Ansible端本次分发的公钥  
- `absent`：保证目标节点上没有Ansible端本次分发的公钥  

最后需要说明的是在key参数中使用的`lookup()`。

#### lookup()或query()读取外部数据

lookup()在Ansible中使用频率非常高，几乎稍微复杂一点的playbook都可能会用上它。

`lookup()`是Ansible的一个插件，可用于从外部读取数据，这里的"外部"含义非常广泛，比如：  

- 从磁盘文件读取(file插件)  
- 从redis中读取(redis插件)  
- 从etcd中读取(etcd插件)  
- 从命令执行结果读取(pipe插件)  
- 从Ansible变量中读取(vars插件)  
- 从Ansible列表中读取(list插件)  
- 从Ansible字典中读取(dict插件)  
- ...

具体可以从哪些"外部"读取以及如何读取，取决于Ansible是否提供了相关的读取插件。官方手册：<https://docs.ansible.com/ansible/latest/plugins/lookup.html#plugin-list>中列出了所有支持的插件，在我写本文的时刻支持的插件数量有63个。

要学完所有这些插件显然是不可能的，但只需掌握常见的插件即可。在这里我准备介绍`file`和`fileglob`、`pipe`这三个插件，其它插件在需要时去手册上查找即可，它们的用法也都非常之简单。

下面是lookup()的语法：

```
lookup('<plugin_name>', 'plugin_argument')
```

插件相关的参数可查询官方手册。

首先介绍`fileglob`插件，它使用通配符来通配**Ansible本地端**的文件名。
```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks: 
    - name: task1
      debug:
        msg: "filenames: {{lookup('fileglob','/etc/*.conf')}}"
```

需注意的是，fileglob查询的是Ansible端文件，且只能通配文件而不能通配目录，且不会递归通配。如果想要查询目标主机上的文件，可以使用`find`模块。

再介绍`file`插件，这个插件用的应该是最多的，它用来读取Ansible本地端文件。例如：
```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks: 
    - name: task1
      debug:
        msg: "file content: {{lookup('file','/etc/hosts')}}"
```

最后介绍`pipe`插件，它从Ansible端的一个命令执行结果中读取数据：
```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks: 
    - name: task1
      debug:
        msg: "command res: {{lookup('pipe','cat /etc/hosts')}}"
```

再者需要说明的是，如果`lookup()`查询出来的结果包含多项，则默认以逗号分隔各项的字符串方式返回，如果想要以列表方式返回，则传递一个lookup的参数`wantlist=True`。例如，fileglob通配出来的文件如果有多个，加上`wantlist=True`：
```yaml
---
- name: play1
  hosts: new
  gather_facts: false
  tasks: 
    - name: task1
      debug:
        msg: "filenames: {{lookup('fileglob','/etc/*.conf',wantlist=True)}}"
```

在Ansible 2.5中添加了一个新的功能`query()`或`q()`，后者是前者的等价缩写形式。`query()`在写法和功能上和lookup一致，其实它会自动调用lookup插件，并且总是以列表方式返回，而不需要手动加上`wantlist=True`参数。例如：
```yaml
- name: task1
  debug:
    msg: "{{q('fileglob','/etc/*.conf')}}"
```

## 5.3 配置主机名

拿到新的主机之后，很可能需要配置这些主机的主机名，特别是现在云主机刚创建出来时的主机名通常都是一大串乱七八糟的字符，所以配置主机名是一件迫不及待的事情。

配置主机名非常简单，可以使用shell模块在远程执行相关命令，也可以使用Ansible提供的`hostname`模块。建议使用`hostname`模块，它支持多种操作系统。

当然，要使用Ansible去设置多个主机名，要求目标主机和目标名称已经关联好，否则多个主机和多个主机名之间无法对应去设置。

例如，在new主机组中有两个节点：192.168.200.34和192.168.200.35，分别设置它们的主机名为new1和new2。

playbook内容如下：
```yaml
---
- name: set hostname
  hosts: new
  gather_facts: false
  vars:
    hostnames:
      - host: 192.168.200.34
        name: new1
      - host: 192.168.200.35
        name: new2
  tasks: 
    - name: set hostname
      hostname: 
        name: "{{item.name}}"
      when: item.host == inventory_hostname
      loop: "{{hostnames}}"
```

执行：
```shell
$ ansible-playbook hostname.yaml
```

在上面的playbook中，除了hostname模块中涉及到的`when`和`loop`指令需要介绍，还需要介绍`vars`指令，这些都是非常重要且常见的指令，playbook离不开它们。

### 5.3.1 vars设置变量

`vars`指令可用于设置变量，可以设置一个或多个变量。下面的设置方式都是合理的：

```yaml
# 设置单个变量
vars: 
  var1: value1

vars: 
  - var1: value1

# 设置多个变量：
vars: 
  var1: value1
  var2: value2

vars: 
  - var1: value1
  - var2: value2
```

`vars`可以设置在play级别，也可以设置在task级别： 

![](/img/ansible/1625969206067.png)

例如：
```yaml
---
- name: play1
  hosts: localhost
  gather_facts: False
  vars: 
    - var1: "value1"
  tasks:
    - name: access var1
      debug: 
        msg: "var1's value: {{var1}}"

- name: play2
  hosts: localhost
  gather_facts: false
  tasks: 
    - name: can't access vars from play1
      debug: 
        var: var1

    - name: set and access var2 in this task
      debug: 
        var: var2
      vars: 
        var2: "value2"

    - name: can't access var2
      debug: 
        var: var2
```

执行结果：
```
PLAY [play1] *****************************

TASK [access var1] ***********************
ok: [localhost] => {
    "msg": "var1's value: value1"
}

PLAY [play2] *****************************

TASK [can't access vars from play1] ******
ok: [localhost] => {
    "var1": "VARIABLE IS NOT DEFINED!"
}

TASK [set and access var2 in this task] **
ok: [localhost] => {
    "var2": "value2"
}

TASK [can't access var2] *****************
ok: [localhost] => {
    "var2": "VARIABLE IS NOT DEFINED!"
}
```

回到设置主机名示例中设置`vars`指令部分：
```yaml
vars:
  hostnames:
    - host: 192.168.200.34
      name: new1
    - host: 192.168.200.35
      name: new2
```

这里只设置了一个变量`hostnames`，但这个变量的值是一个数组结构，数组的两个元素又都是对象(字典/hash)结构。

所以想要访问主机名new1和它的IP地址192.168.200.34，可以：
```
tasks: 
  - debug: 
      var: hostnames[0].name
  - debug: 
      var: hostnames[0].host
```

在Ansible中，变量既是重点，也是难点，目前先了解`vars`可以设置变量即可，后面的文章中会统一解释。

### 5.3.2 when条件判断

Ansible作为一个编排、协调任务的配置管理工具，它必不可少的功能是提供流程控制功能，比如条件判断、循环、退出等。

在Ansible中，提供的唯一一个通用的条件判断是`when`指令，当when指令的值为true时，则该任务执行，否则不执行该任务。

例如：
```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  vars: 
    - myname: "junmajinlong"
  tasks:
    - name: task will skip
      debug:
        msg: "myname is: {{myname}}"
      when: myname == "junma"
    
    - name: task will execute
      debug: 
        msg: "myname is: {{myname}}"
      when: myname == "junmajinlong"
```

该playbook中设置了变量`myname`的值为`junmajinlong`。第一个任务因为when的判断条件是`myname == "junma"`，所以判断结果为false，该任务不执行，即被跳过。同理可知，第二个任务因为when的值为true，所以执行了。

该playbook的执行结果：
```
TASK [task will skip] **************
skipping: [localhost]

TASK [task will execute] ***********
ok: [localhost] => {
    "msg": "myname is: junmajinlong"
}
```

需要注意的是，`when`指令因为已经明确是做条件判断，所以它的值必定是一个表达式，它会自动隐式地包围一层{% raw %} {{}} {% endraw %}，所以在写when指令的条件判断时，不要再手动加上{% raw %} {{}} {% endraw %}。正如上面示例中的写法`when: myname == "junma"`，如果改写成{% raw %} when: {{myname == "junma"}} {% endraw %}，这表示表达式的嵌套，通常来说不会出错，但在某些场景下是错误的，而且Ansible会给出警告。

```
TASK [set hostname] *********************
[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: {{item.host == inventory_hostname}}

[WARNING]: conditional statements should not include jinja2 templating delimiters such as {{ }} or {% %}. Found: {{item.host == inventory_hostname}}
```

`when`一个比较常见的应用场景是实现跳过某个主机不执行任务或者只有满足条件的主机执行任务。例如：

```yaml
when: inventory_hostname == "web1"
```

这表示只有inventory中设置的主机web1才执行任务，其它主机都不执行。

在上面设置主机名的示例中也用到了这一功能，稍后介绍loop循环的时候还会再稍作解释。

最后，虽然`when`指令的逻辑很简单：值为true则执行任务，否则不执行任务。但是，它的用法并不简单，概因`when`指令的值可以是Jinja2的表达式，很多内置在Jinja2中的Python的语法都可以用在when指令中，而这需要掌握Python的基本语法。如果不具备这些知识，那么想要实现某种判断功能可能会感觉到较大的局限性，而且别人写的脚本可能看不懂。但是我的建议是，**别强求**，掌握常用的条件判断方式，以后有需求再去网上搜索对应用法。

### 5.3.3 loop循环

除条件判断外，另一种分支控制结构是循环结构。

Ansible提供了很多种循环结构，一般都命名为`with_xxx`，例如`with_items`、`with_list`、`with_file`等，使用最多的是`with_items`。事实上`with_<lookup>`结构是对应lookup插件的，关于`with_xxx`这些循环结构在以后的文章中再统一介绍。

在这里仅介绍`loop`循环，它是在Ansible 2.5版本中新添加的循环结构，等价于`with_list`。大多数时候，`with_xxx`的循环都可以通过一定的手段转换成loop循环，所以从Ansible 2.5版本之后，原来经常使用的`with_items`循环都可以尝试转换成`loop`。

有了循环结构，生活就美妙多了。例如，在localhost上创建两个目录`/tmp/test{1,2}`，不用循环结构的playbook写法：
```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks: 
    - name: create /tmp/test1
      file: 
        path: /tmp/test1
        state: directory

    - name: create /tmp/test2
      file: 
        path: /tmp/test2
        state: directory
```

使用loop循环的写法：
```yaml
---
- name: play1
  hosts: localhost
  gather_facts: false
  tasks: 
    - name: create directories
      file: 
        path: "{{item}}"
        state: directory
      loop:
        - /tmp/test1
        - /tmp/test2
```

显然，循环可以将多个任务组织成单个任务并实现相同的功能。

解释下上面的`loop`和{% raw %} {{item}} {% endraw %}。

`loop`等价于`with_list`，从名字上可以知道它是遍历数组(列表)的，所以在`loop`指令中，每个元素都以列表的方式去定义。列表有多少个元素，就循环执行file模块多少次，每轮循环中，都会将本次迭代的列表元素保存在控制变量`item`中。

或许，使用伪代码去解释这个loop结构会更容易理解，这里我使用shell伪代码来演示：
```
for item in /tmp/test1 /tmp/test2;do
  mkdir ${item}
done
```

回头再来看设置主机名案例的playbook：
```yaml
---
- name: set hostname
  hosts: new
  gather_facts: false
  vars:
    hostnames:
      - host: 192.168.200.34
        name: new1
      - host: 192.168.200.35
        name: new2
  tasks: 
    - name: set hostname
      hostname: 
        name: "{{item.name}}"
      when: item.host == inventory_hostname
      loop: "{{hostnames}}"
```

在这个示例中，是对{% raw %} {{hostnames} {% endraw %}}进行循环遍历，hostnames中包含两个元素，每个元素都是一个key/value的对象结构。所以，第一次迭代时，item变量的值是：
```
{
  host: "192.168.200.34",
  name: "new1"
}
```

所以`item.host`和`item.name`分别对应于`192.168.200.34`和`new1`。

这里需要注意两点：  

![](/img/ansible/1625969257851.png)

## 5.4 互相添加DNS解析记录

如果新增的机器之间需要通过主机名进行通信，且没有DNS服务器，那么可能需要为每个节点都添加其它节点的DNS解析记录。

例如，nginx主机组中有3个新节点A、B、C，A需要添加B和C的DNS记录，B需要添加A和C的，C需要添加A和B的。

下面是playbook的内容：
```yaml
---
- name: add DNS for each
  hosts: nginx
  gather_facts: true
  tasks: 
    - name: add DNS
      lineinfile: 
        path: "/etc/hosts"
        line: "{{item}} {{hostvars[item].ansible_hostname}}"
      when: item != inventory_hostname
      loop: "{{ play_hosts }}"
```

执行结果：
```
$ ansible-playbook add_dns.yaml

PLAY [add DNS for each] ****************************

TASK [Gathering Facts] *****************************
ok: [192.168.200.27]
ok: [192.168.200.28]
ok: [192.168.200.29]

TASK [add DNS] *************************************
skipping: [192.168.200.27] => (item=192.168.200.27)
changed:  [192.168.200.27] => (item=192.168.200.28)
changed:  [192.168.200.27] => (item=192.168.200.29)
changed:  [192.168.200.28] => (item=192.168.200.27)
skipping: [192.168.200.28] => (item=192.168.200.28)
changed:  [192.168.200.28] => (item=192.168.200.29)
changed:  [192.168.200.29] => (item=192.168.200.27)
changed:  [192.168.200.29] => (item=192.168.200.28)
skipping: [192.168.200.29] => (item=192.168.200.29)
```

在这个playbook中使用了一个模块`lineinfile`，同时使用了when和loop以及几个特殊的变量。我一一介绍。

### 5.4.1 lineinfile模块

`lineinfile`模块用于在源文件中插入、删除、替换行，和sed命令的功能类似，也支持正则表达式匹配和替换。

要完整介绍其用法，会花掉大量篇幅，所以这里介绍一些典型的需求，之后再去查看官方手册便更容易理解。

测试文件a.txt的内容为：
```
paragraph 1
first line in paragraph 1
second line in paragraph 1
paragraph 2
first line in paragraph 2
second line in paragraph 2
```

注意在每次测试后，都将该测试文件还原回原始数据状态。

第一个用法是保证某行的存在：
```yaml
---
- name: test inlinefile
  hosts: localhost
  gather_facts: false
  tasks: 
    - lineinfile:
        path: "a.txt"
        line: "this line must in"
```

执行：
```shell
$  ansible-playbook lineinfile.yaml
```

执行完后，"this line must in"将被追加在a.txt的最后一行。

如果再次执行，则不会再次追加此行。因为lineinfile模块的state参数默认值为present，它能保证幂等性，当要插入的行已经存在时则不会再插入。

如果要移除某行，则设置state参数值为absent即可。下面的任务会移除a.txt中所有的"this line must not in"行(如果多行则全部移除)。
```
- lineinfile:
    path: "a.txt"
    line: "this line must not in"
    state: absent
```

如果想要在某行前、某行后插入指定数据行，则结合`insertbefore`和`insertafter`。例如：

```yaml
- lineinfile:
    path: "a.txt"
    line: "LINE1"
    insertbefore: '^para.* 2'

- lineinfile:
    path: "a.txt"
    line: "LINE2"
    insertafter: '^para.* 2'
```
这会将`LINE1`和`LINE2`分别插入在`paragraph 2`行的前后。结果如下：
```
paragraph 1
first line in paragraph 1
second line in paragraph 1
LINE1
paragraph 2
LINE2
first line in paragraph 2
second line in paragraph 2
```

注意，insertbefore和insertafter指定的正则表达式如果匹配了多行，则默认选中最后一个匹配行，然后在被选中的行前、行后插入。如果明确要指定选中第一次匹配的行，则指定参数`firstmatch=yes`：
```yaml
- lineinfile:
    path: "a.txt"
    line: "LINE1"
    insertbefore: '^para.* 2'
    firstmatch: yes
```

此外，对于`insertbefore`，如果它的正则表达式匹配失败，则会插入到文件的尾部。

lineinfile还可以替换行，使用`regexp`参数指定要匹配并被替换的行即可，默认替换最后一个匹配成功的行。
```yaml
- lineinfile:
    path: "a.txt"
    line: "LINE1"
    regexp: '^para.* 2'
```

结果：
```
paragraph 1
first line in paragraph 1
second line in paragraph 1
LINE1
first line in paragraph 2
second line in paragraph 2
```

如果`regexp`指定的正则表达式匹配失败，则行将插入在文件尾部。

`regexp`结合`state=absent`时，表示删除所有匹配的行。

```yaml
- lineinfile:
    path: "a.txt"
    regexp: '^para.* 2'
    state: absent
```

lineinfile最后一个比较常用的功能是`regexp`结合`insertbefore`或结合`insertafter`。这时候的行将根据insertXXX的位置来插入，而`regexp`参数则充当幂等性判断参数：只有regexp匹配失败时，insertXXX才会插入行。

例如：
```yaml
- lineinfile:
    path: "a.txt"
    line: "hello line"
    regexp: '^hello'
    insertbefore: '^para.* 2'
```

这表示将"hello line"插入在`paragraph 2`行的前面，但如果再次执行，则不会再次插入，因为regexp参数指定的正则表达式已经能够已经存在的"hello line"行。

所以，当`regexp`结合insertXXX使用时，regexp的参数通常都会设置为能够匹配插入之后的行的正则表达式，以便实现幂等性。

### 5.4.2 play_hosts和hostvars变量

`inventory_hostname`变量已经使用过几次了，它表示当前正在执行任务的主机在inventory中定义的名称。在此示例中，inventory中的主机名都是IP地址。

`play_hosts`和`hostvars`是Ansible的预定义变量，执行任务时可以直接拿来使用，不过在Ansible中预定义变量有专门的称呼：魔法变量(magic variables)。Ansible支持不少魔法变量，详细信息参见官方手册：[magic variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#magic)，这里只介绍这两个，在后面的一篇进阶型文章里会统一介绍这些变量，所以各位不必急着打开官方手册去了解各个魔法变量的含义。

首先是`play_hosts`变量，它**存储当前play所涉及的所有主机列表**，但连接失败或执行任务失败的节点不会留在此变量中。

例如，某play指定了`hosts: nginx`，那么执行这个play时，`play_hosts`中就以列表的方式存储了nginx组中的所有主机名。
```json
play_hosts: [
  "192.168.200.27",
  "192.168.200.28",
  "192.168.200.29"
]
```

有时候会借助这个变量来做判断。比如只让目标主机中的其中一个节点执行任务，其它节点不执行，那么可以：
```yaml
tasks:
  - name: xxx
    ping:
    when: inventory_hostname == play_hosts[0]
```

`hostvars`变量用于保存**所有和主机相关的变量，通常包括inventory中定义的主机变量和gather_facts收集到的主机信息变量**。`hostvars`是一个key/value格式的字典(即hash结构、对象)，key是每个节点的主机名，value是该主机对应的变量数据。

例如，在inventory中有一行：
```
192.168.200.27 myvar="hello world"
```

如果要通过`hostvars`来访问该主机变量，则使用`hostvars['192.168.200.27'].myvar`。

因为`gather_facts`收集的主机信息也会保存在hostvars变量中，所以也可以通过hostvars去访问收集到的信息。`gather_facts`中收集了非常多的信息，目前只需记住此处所涉及的`ansible_hostname`即可，它保存的是收集目标主机信息而来的主机名，不是定义在Ansible端inventory中的主机名(因为可能是IP地址)。

> **注意：ansible_hostname和ansible_fqdn**
> 如果目标节点是fqdn格式的主机名，比如www.junmajinlong.com，ansible_fqdn保存的是www.junmajinlong.com，而ansible_hostname保存的是www。

再者，由于`hostvars`中保存了所有目标主机的主机变量，所以任何一个节点都可以通过它去**访问其它节点的主机变量**。比如示例中`hostvars[item].ansible_hostname`访问的是某个主机的`ansible_hostname`变量。

再来看互相添加DNS解析记录的示例：
```yaml
- name: add DNS
  lineinfile: 
    path: "/etc/hosts"
    line: "{{item}} {{hostvars[item].ansible_hostname}}"
  when: item != inventory_hostname
  loop: "{{ play_hosts }}"
```

在这里循环迭代nginx组中的三个节点(假设简称为A、B、C)，当A节点执行这个任务的时候，如果迭代到`play_hosts`中的A节点，则本轮循环跳过，当迭代到B节点或C节点时，则向/etc/hosts文件中添加它们的IP地址和主机名，由于这时候是A节点在执行任务，所以要访问B的主机名变量，只能通过`hostvars[B].ansible_hostname`来访问，同理迭代到C节点也是如此。

## 5.5 配置yum镜像源并安装软件

配置yum源并安装一些常用软件，也是初始化配置服务器应有之义。

现在的需求是：  
1. 备份原有yum镜像源文件，并配置清华大学的yum镜像源：OS源和epel源  
2. 安装常用软件，包括lrzsz、dos2unix、wget、curl、vim等  

playbook如下：
```yaml
---
- name: config yum repo and install software
  hosts: new
  gather_facts: false
  tasks: 
    - name: backup origin yum repos
      shell: 
        cmd: "mkdir bak; mv *.repo bak"
        chdir: /etc/yum.repos.d
        creates: /etc/yum.repos.d/bak

    - name: add os repo and epel repo
      yum_repository: 
        name: "{{item.name}}"
        description: "{{item.name}} repo"
        baseurl: "{{item.baseurl}}"
        file: "{{item.name}}"
        enabled: 1
        gpgcheck: 0
        reposdir: /etc/yum.repos.d
      loop:
        - name: os
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch"
        - name: epel
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/$basearch"

    - name: install pkgs
      yum: 
        name: lrzsz,vim,dos2unix,wget,curl
        state: present
```

第一个任务是将/etc/yum.repos.d下的所有repo文件备份到bak目录中，使用了两个参数，chdir参数表示执行shell模块之前先切换到/etc/yum.repos.d目录下，creates参数表示bak目录存在时则不执行shell模块。

第二个任务是使用`yum_repository`模块配置yum源，该模块可添加或移除yum源。相关常用参数如下：  
- `name`: 指定repo的名称，对应于repo文件中的`[name]`  
- `description`: repo的描述信息，对应于repo文件中的`name: xxx`  
- `baseurl`: 指定该repo的路径  
- `file`: 指定repo的文件名，不需要加上`.repo`后缀，会自动加上  
- `reposdir`: repo文件所在的目录，默认为`/etc/yum.repos.d`目录  
- `enabled`: 是否启用该repo，对应于repo文件中的`enabled`  
- `gpgcheck`: 该repo是否启用gpgcheck，对应于repo文件中的`gpgcheck`  
- `state`: present表示保证该repo存在，absent表示移除该repo  

在示例中使用了一个`loop`循环来添加两个repo：os和epel。

第三个任务是使用`yum`模块安装一些rpm包，yum模块可以更新、安装、移除、下载包。常用参数说明：  
- `name`: 指定要操作的包名
   - 可以带版本号  
   - 可以是单个包名，也可以是包名列表，或者逗号分隔的多个包名  
   - 可以是url  
   - 可以是本地rpm包  
- `state`: 
   - present和installed: 保证包已安装，它们是等价的别名  
   - latest: 保证包已安装了最新版本，如果不是则更新  
   - absent和removed: 移除包，它们是等价的别名  
- `download_only`: 仅下载不安装包(Ansible 2.7才支持)  
- `download_dir`: 下载包存放在哪个目录下(Ansible 2.8才支持)  

![](/img/ansible/1625969333721.png)

## 5.6 时间同步

保证时间同步可以避免很多玄学性问题，特别是对集群中的节点。

通常会使用ntpd时间服务器来保证时间的同步，但在这里简单点，直接使用`ntpdate`命令初步保证时间同步，并将同步后的时间同步到硬件。

所以，整个playbook非常的简单：
```yaml
---
- name: sync time
  hosts: new
  gather_facts: false
  tasks: 
    - name: install and sync time
      block: 
        - name: install ntpdate
          yum: 
            name: ntpdate
            state: present

        - name: ntpdate to sync time
          shell: |
            ntpdate ntp1.aliyun.com
            hwclock -w
```

上面使用了一个`block`指令来组织了两个有关联性的任务，将它们作为了一个整体。由此也可以看到，block的用法非常简单。其实block更多的用于多个关联性任务之间的异常处理，下面的案例中我将介绍一个比较常见也很经典的错误处理方式。

## 5.7 关闭Selinux

本案例用于关闭selinux。playbook如下：

```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks: 
    - name: disable on the fly
      shell: setenforce 0
    
    - name: disable forever in config
      lineinfile: 
        path: /etc/selinux/config
        line: "SELINUX=disabled"
        regexp: '^SELINUX='
```

咋一看这个playbook没有任何问题，先是使用命令`setenforce 0`来临时禁用selinux，然后修改/etc/selinux/config文件来永久禁用。但是一运行，发现失败，错误信息如下(内容太长，我调整了下)：
```
TASK [disable on the fly] ***************************
fatal: [192.168.200.34]: FAILED! => {
  ..., 
  "changed": true, 
  "cmd": "setenforce 0", 
  ..., 
  "msg": "non-zero return code", 
  "rc": 1, 
  ..., 
  "stderr": "setenforce: SELinux is disabled", 
  "stderr_lines": ["setenforce: SELinux is disabled"], 
  "stdout": "", 
  "stdout_lines": []
}
```

从错误信息中不难看出，在使用shell模块执行`setenforce 0`命令的时候，发现该命令的退出状态码不是0，所以Ansible认为这是个失败的命令，于是终止了play，后面的任务也不再执行。但其实我们自己明确地知道，这个命令是没错的，而且从上面的`stderr`的内容中也可以看出，setenforce命令给了我们正确的反馈。

使用Ansible去执行命令，可能经常会遇到类似问题，Ansible并没有那么聪明，它默认只认退出状态码0，其它退出状态码全认为是失败的。

所以，需要让Ansible去处理这种异常。处理异常的方式有很多种，这里只介绍最简单的一种：ignore_errors，后面的文章中再统一介绍。

`ignore_errors`指令正如其名，表示忽略失败的任务，直接将值设置为true即可。

```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks: 
    - name: disable on the fly
      shell: setenforce 0
      ignore_errors: true
    
    - name: disable forever in config
      lineinfile: 
        path: /etc/selinux/config
        line: "SELINUX=disabled"
        regexp: '^SELINUX='
```

使用`ignore_errors`虽然简单，但不是很友好。各位去执行一下上面的playbook，会发现它仍然会将报错信息输出在终端或指定的日志文件中，只不过它不会因为任务失败而导致整个play的中止。但无论如何，它能达到目标：在遇到错误的时候继续执行下去。

异常处理(比如此处的`ignore_errors`)也经常会跟block结合使用，因为在block级别上设置异常处理，可以处理block内部的所有错误。例如：
```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks: 
    - block: 
        - name: disable on the fly
          shell: setenforce 0
        
        - name: disable forever in config
          lineinfile: 
            path: /etc/selinux/config
            line: "SELINUX=disabled"
            regexp: '^SELINUX='
      ignore_errors: true
```

实际上，几乎所有任务级别的指令(除循环类指令外，比如loop)都可以写在block级别上，它们会拷贝到Block内部的所有任务上(也可也看作是block内部的任务继承block级别上的指令)。例如：
```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks: 
    - block: 
        - shell: ls /tmp/a.log
        - shell: ls /tmp/b.log
      ignore_errors: true
```

上面block层次的`ignore_errors`指令，其实拷贝到了两个shell任务上，等价于：
```yaml
---
- name: disable selinux
  hosts: new
  gather_facts: false
  tasks: 
    - block: 
        - shell: ls /tmp/a.log
          ignore_errors: true
        - shell: ls /tmp/b.log
          ignore_errors: true
```

## 5.8 配置防火墙

对于一个新的主机，有可能需要配置防火墙规则。

如果使用Ansible去远程配置防火墙规则，方式有好几种，比如：  
- 可以通过shell模块远程执行iptables命令；  
- 可以在Ansible本地端写好iptables规则，然后传送到目标节点上通过`iptables-restore`来恢复规则；  
- 可以使用Ansible提供的iptables模块来远程设置防火墙规则。

下面是使用shell模块远程执行iptables命令来定义基本的防火墙规则。

```yaml
---
- name: set firewall
  hosts: new
  gather_facts: false
  tasks: 
    - name: set iptables rule
      shell: |
        # 备份已有规则
        iptables-save > /tmp/iptables.bak$(date +"%F-%T")
        # 给它三板斧
        iptables -X
        iptables -F
        iptables -Z

        # 放行lo网卡和允许ping
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A INPUT -p icmp -j ACCEPT

        # 放行关联和已建立连接的包，放行22、443、80端口
        iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT

        # 配置filter表的三链默认规则，INPUT链丢弃所有包
        iptables -P INPUT DROP
        iptables -P FORWARD DROP
        iptables -P OUTPUT ACCEPT
```

也可以使用Ansible提供的iptables模块来配置，这里不多介绍，如果会配置iptables，那么这个模块没有任何难度。

上面的示例非常简单，但是可以将其改造得更为完美一点。比如，可以指定要禁用或放行的端口列表，然后通过循环的方式去配置相关规则，还可以指定协议，指定链的默认规则等等。

此处，我做个简单演示，只支持三种操作：  
1. 允许用户指定filter表中三链的默认规则  
2. 允许用户指定INPUT链放行的tcp端口号列表  
3. 允许执行用户指定的iptables  

修改后的playbook内容如下：

```yaml
---
- name: set firewall
  hosts: new
  gather_facts: false
  vars: 
    allowed_tcp_ports: [22,80,443]
    default_policies:
      INPUT: DROP
      FORWARD: DROP
      OUTPUT: ACCEPT
    user_iptables_rule: 
      - iptables -A INPUT -p tcp -m tcp --dport 8000 -j ACCEPT
      - iptables -A INPUT -p tcp -m tcp --dport 8080 -j ACCEPT

  tasks: 
    - block: 
      - name: backup and empty rules
        shell: |
          # 备份已有规则，并清空规则等
          iptables-save > /tmp/iptables.bak$(date +"%F-%T")
          iptables -X
          iptables -F
          iptables -Z

      - name: green light for lo interface and icmp protocol
        shell: |
          # 放行lo接口、ping和已建立连接的包
          iptables -A INPUT -i lo -j ACCEPT
          iptables -A INPUT -p icmp -j ACCEPT
          iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

      # 放行用户指定的tcp端口列表
      - name: allow for given tcp port
        shell: iptables -A INPUT -p tcp -m tcp --dport {{item}} -j ACCEPT
        loop: "{{ allowed_tcp_ports | default([]) }}"

      # 执行用户自定义的iptables命令
      - name: execute user iptables command
        shell: "{{item}}"
        loop: "{{user_iptables_rule | default([]) }}"

      # 设置filter表三链的默认规则
      - name: default policies for filter table
        shell: iptables -P {{item.key}} {{item.value}}
        loop: "{{ query('dict', default_policies | default({})) }}"
```

上面示例中，定义了三个变量，分别用于存放用户指定要放行的tcp端口、filter表中三链的默认规则以及用户自定义的iptables命令。这没什么可解释的，但需要注意的是有些task中使用了类似{% raw %} {{ allowed_tcp_ports | default([]) }} {% endraw %}的代码，这表示当变量`allowed_tcp_ports`未定义时，则将该变量设置为空列表作为其默认值，以免因变量未定义而loop循环出错。对于本示例来说，直接使用{% raw %} {{ allowed_tcp_ports }} {% endraw %}也是健壮的代码。


## 5.9 远程修改sshd配置文件并重启

有时候为了服务器的安全，可能会去修改目标节点上sshd服务的默认配置，比如禁止Root用户登录、禁止密码认证登录而只允许使用ssh密码认证等。

在修改服务的配置文件时，一般有几种方法：  
1. 通过远程执行sed等命令进行修改配置文件  
2. 通过lineinfile模块去修改配置文件  
3. 在Ansible本地端写好配置文件，然后使用copy模块或template模块传输到目标节点上  

相对来说，第三种方案是最统一、最易维护的方案。

此外，对于服务进程来说，修改了配置文件往往意味着要重启服务，使其加载新的配置文件。对于sshd也一样如此，但是sshd要比其它服务特殊一点，因为Ansible默认基于ssh连接，重启sshd服务会使得Ansible连接断开，好在Ansible默认会重试建立连接，无非是多等待几秒钟。但重建连接有可能会失败，比如修改了配置文件不允许重试、修改了sshd的监听端口等，这可能会使得Ansible因连接失败而无法再继续执行后续任务。

所以，在修改sshd配置文件时，个人建议：  
1. 将此任务作为初始化服务器的最后一个任务，即使连接失败也无所谓。这也是我将这个配置任务放在本文最后介绍的原因所在  
2. 在playbook中加入连接失败的异常处理  
3. 如果目标节点修改了sshd端口号，建议通过Ansible自动或我们自己手动去修改inventory文件中的SSH连接端口号  

这里为了简单，我准备采用lineinfile模块去修改配置文件，要修改的内容只有两项：  
1. 将PermitRootLogin指令设置为no，禁止root用户直接登录  
2. 将PasswordAuthentication指令设置为no，不允许使用密码认证的方式登录  

playbook内容如下：

```yaml
---
- name: modify sshd_config
  hosts: new
  gather_facts: false
  tasks:
    # 1. 备份/etc/ssh/sshd_config文件
    - name: backup sshd config
      shell: 
        /usr/bin/cp -f {{path}} {{path}}.bak
      vars: 
        - path: /etc/ssh/sshd_config

    # 2. 设置PermitRootLogin no
    - name: disable root login
      lineinfile: 
        path: "/etc/ssh/sshd_config"
        line: "PermitRootLogin no"
        insertafter: "^#PermitRootLogin"
        regexp: "^PermitRootLogin"
      notify: "restart sshd"

    # 3. 设置PasswordAuthentication no
    - name: disable password auth
      lineinfile: 
        path: "/etc/ssh/sshd_config"
        line: "PasswordAuthentication no"
        regexp: "^PasswordAuthentication yes"
      notify: "restart sshd"

  handlers: 
    - name: "restart sshd"
      service: 
        name: sshd
        state: restarted
```

执行：
```shell
$ ansible-playbook sshd_config.yaml
```

上面的示例中，唯一需要解释的是两个lineinfile任务中的`notify`指令以及最后面的`handlers`。

### 5.9.1 notify和handlers

此前曾提到过，Ansible绝大多数的模块都具备幂等性，它能保证多次执行不会改变结果，这使得Ansible执行任务时更为安全。

Ansible的模块如何去保证幂等性呢？以shell模块为例：
```yaml
shell: 
  cmd: echo haha >> /tmp/only_once.txt
  creates: /tmp/only_once.txt
```

这里使用了shell模块中的creates参数，它表示如果其指定的文件`/tmp/only_once.txt`存在，则不执行shell命令，只有该文件不存在时才执行。这就是保证幂等性的一种体现：既然不能保证多次执行shell命令的结果不变，那就保证只执行一次。

在此有一个重要的概念，叫做`changed`，其实在Ansible的执行结果中经常会看到这个字眼。从广义的角度上理解，`changed`用于表示目标状态是否发生了改变，通俗而不严谨地讲，就是本次任务是否执行了或执行了是否影响结果。

例如，第一次执行上面的shell任务时，由于`/tmp/only_once.txt`文件不存在，所以会执行shell命令，执行完之后创建了这个文件，于是它关注的目标文件存在性状态发生了改变，这时会设置`changed=1`。如果再次执行该任务，因其关注的文件已存在，所以它不会执行shell命令，其关注的目标存在性状态未改变，这时会设置`changed=0`。

再比如，从Ansible端拷贝nginx的配置文件到目标主机上，如果所拷贝配置文件的内容和原有的配置文件内容不同，则表示其关注的状态(即文件内容，其实是比较md5值)发生了改变，如果内容相同，则表示其关注的状态未发生改变。而配置文件内容发生改变，往往伴随着重启nginx服务的操作。

![](/img/ansible/1625969404485.png)

回头看看修改sshd配置文件的示例，就很容易理解：
```yaml
...
  tasks:
    ...
    - name: disable root login
      lineinfile: ...
      notify: "restart sshd"  ----------┐
                                        |
    - name: disable password auth       |
      lineinfile: ...                   |
      notify: "restart sshd"   ---┐     |
                                  |     |
  handlers:                       |     |
    - name: "restart sshd"  <-----┘-----┘
      service: 
        name: sshd
        state: restarted
```

当`lineinfile`任务确实修改了`/etc/ssh/sshd_config`文件中的内容时，Ansible会监控到它的`changed=1`，于是会根据该任务中`notify`指令指定的名称"restart sshd"去寻找`handlers`中对应该名称"restart sshd"的handler，然后去执行找到的handler。

不难看出，handlers中定义的就是任务，此处handlers中的任务使用的是`service`模块，它用于管理服务(sysV或systemd服务都能管理)，比如设置服务是否开机自启，启动、停止、重启服务等。

唯一需要注意的是，**notify和handlers中任务的名称必须一致**。比如`notify: "restart nginx"`，那么handlers中必须得有一个任务设置了`name: restart nginx`。

此外，在上面的示例中，两个`lineinfile`任务都设置了相同的`notify`，但Ansible不会多次去重启sshd，而是在最后重启一次。实际上，Ansible在执行完某个任务之后并不会立即去执行对应的handler，而是**在当前play中所有普通任务都执行完后再去执行handler**，这样的好处是可以多次触发notify，但最后只执行一次对应的handler，从而避免多次重启。

关于handler，暂时只介绍这么多，在后面的文章中再对其进行扩充。

## 5.10 整合所有任务到单个playbook中

这里将前面所有案例的playbook集合到单个playbook文件中去，这样就可以一次性执行所有任务。

```yaml
---
- name: Configure ssh Connection
  hosts: new
  gather_facts: false
  connection: local
  tasks:
    - name: configure ssh connection
      shell: |
        ssh-keyscan {{inventory_hostname}} >>~/.ssh/known_hosts
        sshpass -p'123456' ssh-copy-id root@{{inventory_hostname}}

- name: Set Hostname
  hosts: new
  gather_facts: false
  vars:
    hostnames:
      - host: 192.168.200.34
        name: new1
      - host: 192.168.200.35
        name: new2
  tasks: 
    - name: set hostname
      hostname: 
        name: "{{item.name}}"
      when: item.host == inventory_hostname
      loop: "{{hostnames}}"

- name: Add DNS For Each
  hosts: new
  gather_facts: true
  tasks: 
    - name: add DNS
      lineinfile: 
        path: "/etc/hosts"
        line: "{{item}} {{hostvars[item].ansible_hostname}}"
      when: item != inventory_hostname
      loop: "{{ play_hosts }}"

- name: Config Yum Repo And Install Software
  hosts: new
  gather_facts: false
  tasks: 
    - name: backup origin yum repos
      shell: 
        cmd: "mkdir bak; mv *.repo bak"
        chdir: /etc/yum.repos.d
        creates: /etc/yum.repos.d/bak

    - name: add os repo and epel repo
      yum_repository: 
        name: "{{item.name}}"
        description: "{{item.name}} repo"
        baseurl: "{{item.baseurl}}"
        file: "{{item.name}}"
        enabled: 1
        gpgcheck: 0
        reposdir: /etc/yum.repos.d
      loop:
        - name: os
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch"
        - name: epel
          baseurl: "https://mirrors.tuna.tsinghua.edu.cn/epel/$releasever/$basearch"

    - name: install pkgs
      yum: 
        name: lrzsz,vim,dos2unix,wget,curl
        state: present

- name: Sync Time
  hosts: new
  gather_facts: false
  tasks: 
    - name: install and sync time
      block: 
        - name: install ntpdate
          yum: 
            name: ntpdate
            state: present

        - name: ntpdate to sync time
          shell: |
            ntpdate ntp1.aliyun.com
            hwclock -w

- name: Disable Selinux
  hosts: new
  gather_facts: false
  tasks: 
    - block: 
        - name: disable on the fly
          shell: setenforce 0
        
        - name: disable forever in config
          lineinfile: 
            path: /etc/selinux/config
            line: "SELINUX=disabled"
            regexp: '^SELINUX='
      ignore_errors: true

- name: Set Firewall
  hosts: new
  gather_facts: false
  tasks: 
    - name: set iptables rule
      shell: |
        # 备份已有规则
        iptables-save > /tmp/iptables.bak$(date +"%F-%T")
        # 给它三板斧
        iptables -X
        iptables -F
        iptables -Z

        # 放行lo网卡和允许ping
        iptables -A INPUT -i lo -j ACCEPT
        iptables -A INPUT -p icmp -j ACCEPT

        # 放行关联和已建立连接的包，放行22、443、80端口
        iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
        iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT

        # 配置filter表的三链默认规则，INPUT链丢弃所有包
        iptables -P INPUT DROP
        iptables -P FORWARD DROP
        iptables -P OUTPUT ACCEPT

- name: Modify sshd_config
  hosts: new
  gather_facts: false
  tasks:
    - name: backup sshd config
      shell: 
        /usr/bin/cp -f {{path}} {{path}}.bak
      vars: 
        - path: /etc/ssh/sshd_config

    - name: disable root login
      lineinfile: 
        path: "/etc/ssh/sshd_config"
        line: "PermitRootLogin no"
        insertafter: "^#PermitRootLogin"
        regexp: "^PermitRootLogin"
      notify: "restart sshd"

    - name: disable password auth
      lineinfile: 
        path: "/etc/ssh/sshd_config"
        line: "PasswordAuthentication no"
        regexp: "^PasswordAuthentication yes"
      notify: "restart sshd"

  handlers: 
    - name: "restart sshd"
      service: 
        name: sshd
        state: restarted
```

内容很长，可能你也感受到了，可读性很差，维护也很不方便。

更友好的一种组织方式是将各个任务分类，各自存放在不同的playbook文件中(就像未整合那样)，然后使用一个入口playbook文件引入所有任务文件。

例如：  
- 配置主机SSH连接互信的任务放在sshkey.yaml  
- 设置主机名的任务放在hostname.yaml  
- 互相添加DNS解析记录的任务放在add_dns.yaml  
- 配置yum源并安装常用软件包的任务放在add_repos.yaml  
- 时间同步的任务放在synctime.yaml  
- 禁用selinux的任务放在disable_selinux.yaml  
- 配置防火墙的任务放在iptables.yaml  
- 修改sshd配置文件的任务放在sshd_config.yaml  

因为这些任务都是初始化服务器的任务，所以将这些任务文件共同存在一个单独的目录中(比如init_server目录)，则效果更佳。

最后，通过一个入口文件引入所有这些任务文件将它们组织起来。假设入口文件名为main.yaml，其内容为：

```yaml
---
- import_playbook: "init_server/sshkey.yaml"
- import_playbook: "init_server/hostname.yaml"
- import_playbook: "init_server/add_dns.yaml"
- import_playbook: "init_server/add_repos.yaml"
- import_playbook: "init_server/synctime.yaml"
- import_playbook: "init_server/disable_selinux.yaml"
- import_playbook: "init_server/iptables.yaml"
- import_playbook: "init_server/sshd_config.yaml"
```

然后使用ansible-playbook去执行这个入口playbook文件即可：

```shell
$ ansible-playbook main.yaml
```

如此一来，各个任务自治，维护起来更为容易。当然，这只是小菜，敬请期待后面的文章，届时我将专门介绍组织文件的方式以及Ansible Role。