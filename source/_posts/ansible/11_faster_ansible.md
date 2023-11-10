---
title: 11.Ansible你快点：Ansible执行过程分析、异步、效率优化
p: ansible/11_faster_ansible.md
date: 2021-07-11 10:37:40
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。

# 11.Ansible你快点：Ansible执行过程分析、异步、效率优化

Ansible虽然方便，但有个"为人诟病"的问题：任务执行速度太慢了，在有大量任务、大量循环任务时，其速度之慢真的是会让人等到崩溃的。

Ansible官方给了一些优化选项供用户选择，还可以去网上寻找优化Ansible相关的插件。但在调优Ansible之前，应当先去**理解Ansible的执行流程**，如此才能知道为什么速度慢、要如何调优以及调优后会造成什么后果。此外，还应学会**测量任务的执行速度**。

此外，本文还会回顾部分Ansible执行策略，更详细的执行策略说明，可复习第十章的"理解Ansible执行策略"部分。

## 11.1 测量任务执行速度：profile_tasks插件

Ansible官方提供了几个可用于计时的回调插件：  

- (1).`profile_tasks`：该回调插件用于计时每个任务的执行时长  
- (2).`profile_roles`插件用于计时每个Role的执行时长  
- (3).`timer`插件用于计时每个play执行时长  

要使用这些插件，需要在ansible.cfg配置文件中的`callback_whitelist`中加入各插件。如下：

```ini
[defaults]
callback_whitelist = profile_tasks
# callback_whitelist = profile_tasks, profile_roles, timer
```

上面我只开启了`profile_tasks`插件。

这些回调插件会将对应的计时信息输出，通过观察这些计时信息，便可以知道任务执行消耗了多长时间，并多次比对计时信息，从而可确定哪种方式更高效。

然后执行几个任务看看输出结果，如下playbook文件内容：

```yaml
---
- name: test for timer
  hosts: timer
  gather_facts: no
  tasks:
    - name: only one debug
      debug: 
        var: inventory_hostname
      
    - name: shell
      shell:
        cp /etc/fstab /tmp/
      loop: "{{ range(0, 100)|list }}"

    - name: scp
      copy:
        src: /etc/hosts
        dest: /tmp/
      loop: "{{ range(0, 100)|list }}"
```

其中timer主机组有三个节点，所以整个playbook中，每个节点执行201次任务，总共执行603次任务。以下是开启`profile_tasks`后在屏幕中输出的计时信息：

```shell
$ ansible-playbook -i timer.host timer.yml
...................
......省略输出......
...................
=========================================
scp ------------------------------------ 57.96s
shell ---------------------------------- 42.78s
only one debug ------------------------- 0.07s
```

从结果中可看到，3个节点的debug任务总共花费0.07秒，3个节点的shell任务总共300次任务花费42.78秒，3个节点的scp任务总共300次任务花费57.96秒。

## 11.2 Ansible执行流程分析

ansible命令或ansible-playbook命令加上`-vvv`选项，会输出很多调试信息，包括建立的连接、发送的文件等等。

例如，下面是Ansible 2.9默认配置中执行**每单个任务**都涉及到的步骤，其中我省略了大量信息以便各位能够看懂关键步骤。各位可自行加上`-vvv`去执行一个任务并观察输出信息，同时可与我所做的注释做比较。

> 需注意：不同版本的Ansible为每个任务建立的连接数量不同，Ansible 2.9为每个任务建立7次ssh连接。有的资料或书籍中介绍时说只建立二次、三次、四次ssh连接都是有可能的，版本不同确实是有区别的。

```
# 1.第一个连接：获取用户家目录，此处为/root
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ....... '/bin/sh -c '"'"'echo ~ && sleep 0'"'"''
<node1> (0, '/root\n', ......)

# 2.第二个连接：在家目录下创建临时目录，临时目录由配置文件中remote_tmp指令控制
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ...... '/bin/sh -c '"'"'( umask 77 && mkdir -p "` echo /root/.ansible/tmp/ansible-tmp-1575542743.85-116022411851390 `" && ...... `" ) && sleep 0'"'"''

# 3.第三个连接：探测目标节点的平台和python解释器的版本信息
<node1> Attempting python interpreter discovery
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ......

# 4.第四个连接：将要执行的模块相关的代码和参数放到本地临时文件中，并使用sftp将任务文件传输到被控节点的临时文件中
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ......
Using module file /usr/lib/python2.7/site-packages/ansible/modules/system/ping.py
......
<node1> SSH: EXEC sftp ......
<node1> (0, 'sftp> put /root/.ansible/tmp/ansible-local-78628na2FKL/tmpaE1RbJ /root/.ansible/tmp/ansible-tmp-1575542743.85-116022411851390/AnsiballZ_ping.py\n', ......

# 5.第五个连接：对目标节点上的任务文件授以执行权限
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ...... '/bin/sh -c '"'"'chmod u+x /root/.ansible/tmp/ansible-tmp-1575542743.85-116022411851390/ /root/.ansible/tmp/ansible-tmp-1575542743.85-116022411851390/AnsiballZ_ping.py && sleep 0'"'"''
......

# 6.第六个连接：执行目标节点上的任务
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ...... '/bin/sh -c '"'"'/usr/bin/python /root/.ansible/tmp/ansible-tmp-1575542743.85-116022411851390/AnsiballZ_ping.py && sleep 0'"'"''
<node1> (0, '\r\n{"invocation": {"module_args": {"data": "pong"}}, "ping": "pong"}\r\n', 
......

# 7.第七个连接：删除目标节点上的临时目录
<node1> ESTABLISH SSH CONNECTION FOR USER: None
<node1> SSH: EXEC ssh -vvv ...... '/bin/sh -c '"'"'rm -f -r /root/.ansible/tmp/ansible-tmp-1575542743.85-116022411851390/ > /dev/null 2>&1 && sleep 0'"'"''
......
```

总结一下Ansible为**每单个任务**建立7次ssh连接所作的事情：  

- (1).第一个连接：获取远程主机时行目标用户的家目录，此处为/root  
- (2).第二个连接：在远程家目录下创建临时目录，临时目录可由ansible.cfg中`remote_tmp`指令控制  
- (3).第三个连接：探测目标节点的平台和python解释器的版本信息  
- (4).第四个连接：将待执行模块的相关代码和参数放到本地临时文件中，并使用sftp将任务文件传输到被控节点的临时文件中  
- (5).第五个连接：对目标节点上的任务文件授以执行权限  
- (6).第六个连接：执行目标节点上的任务  
- (7).第七个连接：删除目标节点上的临时目录，并将执行结果返回给Ansible端  

从单个任务的执行流程跳出来，更全局一点，那么整个执行流程(默认配置下)大致如下(不考虑inventory阶段或执行完任务后的回调阶段，只考虑执行的任务流程)：  

- (1).进入第一个play，挑选forks=5设置的5个节点  
- (2).每个节点执行第一个任务，每个节点都会建立7次ssh连接  
- (3).每个节点执行第二个任务，每个节点都再次建立7次ssh连接  
- (4).按照相同逻辑执行该play中其它任务...
- (5).所有节点执行完该play中的所有任务后，进入下一个play  
- (6).按照上面的流程执行完所有play中的所有任务  

以上便是整个执行流程，各位大概也看出来了，Ansible在建立ssh连接方面上实在是"不遗余力"，可能是因为Ansible官方团队太爱ssh了……开玩笑的啦……。

## 11.3 回顾Ansible的执行策略

使用forks、serial、strategy等指令可以改变Ansible的执行策略。

默认情况下forks=5，这表明在某一时刻最多只有5个执行任务的工作进程(还有一个主进程)，也即最多只能挑选5个节点同时执行任务。

`serail`是play级别的指令，用于指定几个节点作为一批去执行该play，该play执行完后才让下一批节点执行该play中的任务。如果不指定serial，则默认的行为等价于将所有节点当作一批。

`strategy`指令用于指定节点执行任务时的策略，其侧重点在于节点而在于任务，默认情况下其策略为`linear`，表示某个节点先执行完一个任务后等待其余所有节点都执行完该任务，才统一进入下一个任务。另一种策略是`free`策略，表示某节点执行完一个任务后不等待其它节点，而是毫不停留的继续执行该play中的剩余任务，直到该play执行完成，才释放节点槽位让其它未执行任务的节点开始执行任务。

前面的文章已经详细介绍过Ansible的执行策略，所以此处仅作简单回顾，如有所遗忘，请复习前面的文章。

## 11.4 加大forks的值

![](/img/ansible/1625971566415.png)

## 11.5 修改执行策略

默认情况下Ansible会让所有节点(或者serial指定的数量)执行完同一个任务后才让它们进入下一个任务，这体现了各节点的公平性和实时性：每个节点都能尽早执行到任务。这其实和操作系统的进程调度是类似的概念，只不过相对于操作系统的调度系统来说，Ansible的调度策略实在是太简陋了。

假设forks设置的比较大，可以一次性让足够多的节点并发执行任务，那么同时设置任务的执行策略为`strategy=free`便能让这些执行任务的节点彻底放飞自我。只是剩余的一部分节点可能会比较悲剧，它们处于调度不公平的一方。但是从整体来说，先让大部分节点快速完成任务是值得的。

但是要注意，有些场景下要小心使用free策略，特别是节点依赖时。比如，某些节点运行服务A，另一些节点运行服务B，而服务B是依赖于服务A的，那么必须不能让运行B服务的节点先执行，对于有节点依赖关系的任务，为了健壮性，一般会定义好等待条件，但是出现等待有可能就意味着浪费。

## 11.6 使Ansible异步执行任务

默认情况下，Ansible按照同步执行的方式执行每个任务。即对每个任务来说，都需要等待目标节点执行完该任务后回馈给Ansible端的报告，然后Ansible才认为该节点上的该任务已经执行完成，才会考虑下一步骤，比如free策略下该节点继续执行下一个任务，或者等待其它节点完成该任务，等等。

### 11.6.1 async和poll指令

Ansible允许在**task级别**(且只支持task级别)指定该task是否以异步模式(即放入后台)执行，即将该异步任务放入后台。例如：

```yaml
- name: it is an async task
  copy:
    src:
    dest:
  async: 200
  poll: 2
- name: a sync task
  copy:
    src: 
    dest: 
```

其中`async`指令表示该任务将以异步的模式执行。async指令的值200表示，如果该后台任务200秒还未完成，则认为该任务失败。`poll`指令表示该任务丢入后台后，Ansible每隔多久去检查一次异步任务是否已成功、是否报错等，只有检查到已完成后才认为该异步任务执行完成，才会进入下一个任务。

如此看来，似乎这个异步执行模式并非想象中那样真正的异步：将一个任务放入后台执行，立即进入下一个任务。而且这里的异步似乎会减慢任务的执行流程。比如后台任务在第3秒完成，也必须等到第4秒检查的时候才认为执行完成。

如果**poll指令的值大于0，这确实不是真正的异步**，每个工作进程必须等待放入后台的任务执行完成才会进入下一个任务，换句话说，尽管使用了async异步指令，也仍然会阻塞在该异步任务上。这会减慢任务的执行速度，但此时执行该异步任务的Ansible工作进程会放弃CPU，使得CPU可以执行其它进程(对于Ansible控制节点来说，这算哪门子优点？)。

但如果**poll指令的值为0，将会以真正的异步模式执行任务**，表示Ansible工作进程不检查后台任务的执行状况，而是直接执行下一个任务。

不管poll指令的值是否大于0，只要使用了异步，那么强烈建议将forks指令的值设置的足够大。比如能够一次性让所有节点都开始异步执行某任务，这样的话，无论poll的值是否大于0，都能提升效率。

此外，也可以在ansible命令中使用`-B N`选项指定async功能，N为超时时长，`-P N`选项指定poll功能，N为检查后台任务状况的时间间隔。

例如：

```shell
$ ansible inventory_file -B200 -P 0 -m yum -a 'name=dos2unix' -o -f 20
```

### 11.6.2 等待异步任务

无论是编程语言还是如Ansible一般的工具，只要提供异步执行模式，都必不可少的需要提供一个等待异步任务执行完成的功能(注：同步执行模式不需要等待，因为同步执行模式本就是从上到下依次执行的)。

例如下面的任务执行流程：

```
T1(async) --> T2(sync) --> T3(sync) --> T4(wait T1) --> T5
```

T1是一个异步任务，放入后台执行时立即去执行T2和T3，但是T5这个任务比较特殊，它依赖于T1任务的执行成功，于是在T5任务之前插入一个等待T1异步任务执行完成的等待任务，只要T1没有完成，就会一直阻塞在T4上，那么自然不会执行到T5，如果T1完成了，T4便等待完成，于是可以执行T5。

Ansible中想要等待异步任务需要借助于`async_status`模块，该模块接受一个后台任务的job id作为参数，然后获取该后台任务的状态并返回。

该模块返回的状态信息包含以下几项属性：  

- (1).ansible_job_id：异步任务的job id  
- (2).finished：表示所等待的异步任务是否已执行完成，值为1表示完成，0表示未完成  
- (3).started：表示所等待的异步任务是否已开始执行，值为1表示已开始，0表示未开始  


例如，下面是官方提供的一个典型的异步等待示例：

```yml
- name: Asynchronous yum task
  yum:
    name: nginx
    state: present
  async: 1000
  poll: 0
  register: yum_sleeper

- name: Wait for asynchronous job to end
  async_status:
    jid: '{{ yum_sleeper.ansible_job_id }}'
  register: job_result
  until: job_result.finished
  retries: 30
```

此示例中，异步任务注册了一个变量`yum_sleeper`，该变量中包含一个`ansible_job_id`的属性。将该属性交给`async_status`模块的jid选项，该模块便可以获取该异步任务的状态，并将状态注册到变量`job_result`中，结合`until`指令不断等待`job_result.finished`事件发生，即表示异步任务执行完成。

同时等待多个异步任务也是常见的需求：只有所有想要等待的任务全都完成了才继续向下执行。Ansible中可以对`async_status`模块使用loop循环来完成该功能。

例如：

```yaml
---
- name: test for timer
  hosts: timer
  gather_facts: no
  tasks:
    - name: async task 1
      shell: sleep 5
      async: 200
      poll: 0
      register: async_task1

    - name: async task 2
      shell: sleep 10
      async: 200
      poll: 0
      register: async_task2

    - name: waiting for all async task
      async_status: 
        jid: "{{item}}"
      register: job_result
      until: job_result.finished
      loop: 
        - "{{ async_task1.ansible_job_id }}"
        - "{{ async_task2.ansible_job_id }}"

    - name: after waiting
      debug: 
        msg: "after waiting"
```

### 11.6.3 何时使用异步任务

有时候合理应用异步任务能大幅提升Ansible的执行效率，但也并非所有场景都能够使用异步任务。

总结来说，以下一些场景可能使用到Ansible的异步特性：  

![](/img/ansible/1625971628662.png)  

## 11.7 开启ssh长连接

Ansible对ssh的依赖性非常强，优化ssh连接在一定程度上也是在优化Ansible。

其中一项优化是开启ssh的长连接，即长时间保持连接状态。开启长连接后，在ssh连接过期前会一直保持ssh连接已建立的状态，使得下次和目标节点建立ssh连接时将直接使用该连接。相当于对ssh连接进行了缓存。

要开启ssh长连接，要求Ansible端的openssh版本高于或等于5.6。使用`ssh -V`可以查看版本号。然后设置ansible使用ssh连接被控端的连接参数，此处修改/etc/ansible/ansible.cfg，在此文件中启动下面的连接选项，其中`ControlPersist=5d`是控制ssh连接会话保持时长为5天。

```shell
ssh_args = -C -o ControlMaster=auto -o ControlPersist=5d
```

除此之外直接设置`/etc/ssh/ssh_config`(不是sshd_config，因为ssh命令是客户端命令)中对应的长连接选项也是可以的。

以后只要有了一次ssh连接，就会将连接保留下来，例如：执行一次Ansible的ad-hoc操作，会建立ssh连接。

```shell
ansible centos -m ping
```

查看netstat，发现ssh进程的会话一直是established状态(为了排版，我略了前面3个字段)。

```shell
$ netstat -tnalp

Local Address         Foreign Address    State       PID/Program name    
0.0.0.0:22            0.0.0.0:*          LISTEN      1143/sshd           
127.0.0.1:25          0.0.0.0:*          LISTEN      2265/master         
192.168.200.26:58474  192.168.200.59:22  ESTABLISHED 31947/ssh: /root/.a 
192.168.200.26:22     192.168.200.1:8189 ESTABLISHED 29869/sshd: root@pt 
192.168.200.26:37718  192.168.200.64:22  ESTABLISHED 31961/ssh: /root/.a 
192.168.200.26:38894  192.168.200.60:22  ESTABLISHED 31952/ssh: /root/.a 
192.168.200.26:48659  192.168.200.61:22  ESTABLISHED 31949/ssh: /root/.a 
192.168.200.26:33546  192.168.200.65:22  ESTABLISHED 31992/ssh: /root/.a 
192.168.200.26:54824  192.168.200.63:22  ESTABLISHED 31958/ssh: /root/.a 
:::22                 :::*               LISTEN      1143/sshd           
::1:25                :::*               LISTEN      2265/master
```

同时，会在当前用户家目录的`.ansible/cp`目录下生成一些socket文件，每个ssh连接会话一个文件。

```shell
$ ls -l ~/.ansible/cp/
total 0
srw------- 1 root root 0 Jun  3 18:26 5c4a6dce87
srw------- 1 root root 0 Jun  3 18:26 bca3850113
srw------- 1 root root 0 Jun  3 18:26 c89359d711
srw------- 1 root root 0 Jun  3 18:26 cd829456ec
srw------- 1 root root 0 Jun  3 18:26 edb7051c84
srw------- 1 root root 0 Jun  3 18:26 fe17ac7eed
```

这些socket文件的存放路径由ansible.cfg文件中的`control_path_dir`指令决定。

### 11.7.1 开启ssh长连接后的注意事项

开启ssh长连接固然会将连接缓存下来以避免频繁建立ssh连接，但也因此带来了一个问题：只要目标节点sshd监听地址和端口未变，那么只要ssh长连接未过期，客户端(比如Ansible)总能使用已缓存的连接和目标节点通信。

这是什么意思呢？比如A节点上的ssh开启了长连接(注：长连接是ssh的特性，不是Ansible的特性，所以ssh自身也可以设置)，当A通过ssh第一次连接到`root@B`节点后，A会缓存到B节点的ssh连接，如果此时B节点目标用户root修改了密码，A节点借助缓存下来的ssh长连接仍然能够连接到`root@B`节点。

对于Ansible来说，开启长连接后可能会带来一些问题，比如缓存了某节点的ssh连接后，又修改了Inventory中该节点的ssh连接变量，但这些连接变量在ssh长连接过期之前将不会生效。对于没有注意到这一现象的人来说，这样的问题是非常难以排查的，因为很难想到ssh长连接这方面。

## 11.8 开启Pipelining

从前面对Ansible执行任务的流程中可以发现，Ansible执行每个任务时都会在本地将模块(通常是Python脚本程序)和相关参数打包后通过sftp发送到目标节点上，然后执行目标节点上的临时脚本文件。这些行为还带来了副作用，比如多建立了几个ssh连接来创建临时目录、删除目录等。

Ansible现在也支持使用ssh的pipelining特性(注意，仍然是ssh的特性)，当Ansible中开启了Pipelining后，一个任务的所有动作都在一个ssh会话中完成，也会省去sftp到远端的过程，它会直接将要执行任务涉及到的指令(比如python语句)通过远程shell的方式发送到目标节点的标准输入(stdin)中，然后在目标节点执行这些代码。

如果不理解这个过程，可以理解下面这个更直观的ssh命令：

```shell
$ echo 'hostname -I' | ssh root@192.168.200.48 'bash'
192.168.200.48 
```

上面的命令中，ssh连接到192.168.200.48，同时ssh命令会读取标准输入中的`hostname -I`并将其写入到远程主机上的标准输入供bash命令读取，于是bash命令执行读取到的数据。所以，相当于是在远程主机上执行了`echo "hostname -I" | bash`。

类似的，当Ansible开启Pipelining特性后，会将任务相关的指令(通常是Python语句)通过ssh发送到目标节点的标准输入中，然后python解释器程序读取指令并执行。相当于：

```shell
$ echo 'print("hello world")' | ssh root@192.168.200.48 'python'
```

也相当于在远程主机上执行了：

```shell
$ echo 'print("hello world")' | python
```

既然任务相关的指令已经发送到目标的标准输入，那自然就不需要再通过传输文件的方式将任务传输到目标节点再执行，这显然也减少了一大堆副作用而建立的ssh连接。事实上，当Ansible开启了Pipelining特性后，提升的效率是巨大的。

Ansible开启Pipelining的方式是在配置文件(如ansible.cfg)中设置`pipelining=true`，默认是false，即默认Pipelining是禁用状态。

```shell
$ grep '^pipelining' /etc/ansible/ansible.cfg
pipelining = True
```

### 11.8.1 开启Pipelining后的注意事项

但是要注意，如果在Ansible中使用sudo相关行为时，需要在被控节点的/etc/sudoers中禁用"requiretty"。

例如，对于下面的play：

```yaml
---
- name: test for timer
  hosts: timer
  gather_facts: no
  become: yes
  become_user: root
  become_method: sudo
  tasks:
    - shell: sleep 1
```

不禁用`requiretty`将报错：

```
Pseudo-terminal will not be allocated because stdin is not
a terminal.\r\nsudo: sorry, you must have a tty to run sudo
```

可以通过visudo编辑配置文件，注释该选项来禁用requiretty。

```shell
$ grep requiretty /etc/sudoers
Defaults    requiretty  # 注释此行表示禁用
```

之所以要设置/etc/sudoers中的requiretty，是因为ssh远程执行命令时，它的环境是非登录式非交互式shell，默认不会分配tty，没有tty，ssh的sudo就无法关闭密码回显(使用"-tt"选项强制SSH分配tty)。所以出于安全考虑，/etc/sudoers中默认是开启requiretty的，它要求只有拥有tty的用户才能使用sudo，也就是说ssh连接过去不允许执行sudo。

但是，修改设置/etc/sudoers的操作是在被控节点上进行的(或者ansible连接过去修改)，其实在ansible端也可以解决sudo的问题，只需在ansible的ssh参数上加上"-tt"选项即可(注：经测试，Ansible2.9中使用-tt会阻塞，-t或—ttt不阻塞但仍然失败，但我肯定，以前的版本是可以这么做的，所以相关方案仍然留在此处)。

```shell
$ grep 'ssh_args' /etc/ansible/ansible.cfg
ssh_args = -C -o ControlMaster=auto -o ControlPersist=1d -tt

# 或者在命令行中指定
$ ansible-playbook --ssh-extra-args="-tt"" xxxxxx
```

### 11.8.2 开启Pipelining后的执行流程

开启Pipelining后，再来看下执行单个任务时的执行流程(为排版也为让各位能一眼看懂，我省略了一些信息)：

```
TASK [test task] *******************************************
task path: /root/ansible/timer.yml:6

# 第一个SSH连接，用于探测目标节点上支持的Python版本
<192.168.200.48> Attempting python interpreter discovery
<192.168.200.48> ESTABLISH SSH CONNECTION FOR USER: None
<192.168.200.48> SSH: EXEC ssh -C -o ......
<192.168.200.48> (0, b'
                      PLATFORM\nLinux\nFOUND\n
                      /usr/bin/python\n
                      /usr/bin/python3.6\n
                      /usr/bin/python2.7\n
                      /usr/bin/python3\n
                      /usr/bin/python\n
                      ENDFOUND\n', b'')

# 第二个SSH连接用于探测目标节点操作系统的信息
<192.168.200.48> ESTABLISH SSH CONNECTION FOR USER: None
<192.168.200.48> SSH: EXEC ssh -C -o ......
<192.168.200.48> (0, b'{"osrelease_content": 
                        "NAME=\\"CentOS Linux\\"\\n
                         VERSION=\\"7 (Core)\\"\\n
                         ID=\\"centos\\"\\n
                         ID_LIKE=\\"rhel fedora\\"\\n
                         VERSION_ID=\\"7\\"\\n
                         ......b'')

# 准备执行任务，加载任务使用的模板文件，且发现开启了Pipelining
Using module file ......ansible/modules/commands/command.py
Pipelining is enabled.

# 第三个SSH连接用于执行任务
<192.168.200.48> ESTABLISH SSH CONNECTION FOR USER: None
<192.168.200.48> SSH: EXEC ssh -C ......
 '/bin/sh -c '"'"'/usr/bin/python && sleep 0'"'"''
<192.168.200.48> (目标节点执行任务返回的结果)
```

从执行流程上已经看到，开启Pipelining后，除了建立两个必要的SSH连接探测Python版本(老版本的Ansible不支持多Python版本自动探测功能)和操作系统信息外，执行任务相关的SSH连接只有一个。

### 11.8.3 开启和不开启Pipelining的效率比较

下面是开启和不开启Pipelining时执行本文开头的计时任务，3个节点总共603个任务。

```shell
# 开启Pipelining之前
$ ansible-playbook -i timer.host timer.yml
...................
......省略输出......
...................
=========================================
scp ------------------------------------ 57.96s
shell ---------------------------------- 42.78s
only one debug ------------------------- 0.07s

# 开启Pipelining后
$ ansible-playbook -i timer.host timer.yml
...................
......省略输出......
...................
=========================================
scp ------------------------------------ 39.99s
shell ---------------------------------- 20.29s
only one debug ------------------------- 0.07s
```

可见，使用了Pipelining后，总时间几乎是对半减，效率提升不可谓不高。

## 11.9 修改facts收集行为

Ansible默认会收集所有节点的所有facts信息，而收集facts信息是非常慢的。

如果能够确保play中使用不到facts中的信息，则可以`gather_facts: no`关闭收集功能。

如果只想要facts中的一部分信息，那么在收集时可以指定只收集这一部分信息，其它的不要收集。例如只想收集网卡相关信息可以设置`gather_subset=!all,!any,network`，这样可以减少收集的数据量，从而提升效率。

最后，还可以将facts缓存下来。关于facts缓存，在前面"回归Ansible并进阶"的章节中已经详细介绍过。所以此处略过。

## 11.10 Shell层次上的优化：将任务分开执行

在前面"利用Role部署LNMP"的章节最后，我提到过这种优化手段。这里再简单回顾一下。

在LNMP的示例中，分别为nginx和php和MySQL都单独定义了自己的Role，它们分别在三批节点上执行。为了统筹这些Role，一般会定义一个汇聚了所有的Role的playbook文件，称为入口playbook，比如称为main.yml或site.yml。

但是，把这些Role聚集到单个playbook文件中后就必然会产生前后顺序关系。比如执行nginx Role的时候，PHP Role和MySQL Role对应的节点都在空闲。这是一种很低效的执行方式。

![](/img/ansible/1625971698492.png)

这样一来，分别执行这三个Role的三批节点就可以同时开始执行任务了。

而且，如果某个Role依赖于另一个Role，可以协调它们的顺序并取消被依赖Role的后台执行方式。比如php Role依赖于mysql Role时(只是假设)，可以将mysql.yml以非后台的方式放在php Role的前面执行。

```shell
$ ansible-playbook nginx.yml >/tmp/nginx.log &
$ ansible-playbook mysql.yml >/tmp/mysql.log # 被依赖，所以不后台
$ ansible-playbook php.yml   >/tmp/php.log
```

当然，更推荐也更健壮的方式是在php Role中定义等待MySQL Role的任务。

再者，还可以写一个简单的Shell脚本，在Shell脚本中加入一些判断逻辑来决定如何执行这些Role，这样实现的逻辑可能比Ansible本身还要更丰富。

## 11.11 第三方策略插件：Mitogen for Ansible

Ansible的执行策略(即由strategy指令指定的值)是以插件方式提供的。Ansible官方目前提供了四种策略插件：  

- (1).linear  
- (2).free   
- (3).host-pinned 
- (4).debug  

作为使用Ansible的用户来说，可以去编写自己的策略插件或网上寻找相关的策略插件。有一款备受青睐的策略插件名为Mitogen for Ansible。

官方介绍和文档：Mitogen for Ansible--<https://mitogen.networkgenomics.com/ansible_detailed.html>。

使用该插件后，将尽可能地尽早执行任务，注意是尽早而不是尽快，它所作的工作不可能会影响一个已编码完成的Ansible模块的执行速度，换句话说，Mitogen提升的是从某个任务开始到该任务对应模块开始执行之间的速度。按照Mitogen官方描述，该插件可以将Ansible的执行速度提升到1.25倍-7倍。

不管怎么官方怎么说，自己测试一下是最好的。

首先下载并解压：

```shell
$ wget 'https://networkgenomics.com/try/mitogen-0.2.9.tar.gz'
$ mkdir -p ~/.ansible/plugins
$ tar xf mitogen-0.2.9.tar.gz -C ~/.ansible/plugins/
```

然后在ansible.cfg中设置使用该策略插件，并指定该策略插件提供的策略。

```ini
[defaults]
strategy_plugins = ~/.ansible/plugins/mitogen-0.2.9/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

mitogen插件提供了三种策略：  

```shell
$ ls -1 ~/.ansible/plugins/mitogen-0.2.9/ansible_mitogen/plugins/strategy/
__init__.py
mitogen_free.py
mitogen_host_pinned.py
mitogen_linear.py
mitogen.py
```

其中：  

- (1).`mitogen_linear`对应于Ansible自身的linear策略  
- (2).`mitogen_free`对应于Ansible自身的free策略  
- (3).`mitogen_host_pinned`对应于Ansible自身的host_pinned策略  

然后就可以开始计时测试了，仍然是本文开头的计时任务。目前Ansible已开启Pipelining特性，观察使用mitogen_linear的效率和原生linear的效率相比如何：

```shell
# 开启Pipelining但未使用mitogen插件
$ ansible-playbook -i timer.host timer.yml
...................
......省略输出......
...................
=========================================
scp ------------------------------------ 39.99s
shell ---------------------------------- 20.29s
only one debug ------------------------- 0.07s

# 开启Pipelining且使用mitogen插件
$ ansible-playbook -i timer.host timer.yml
...................
......省略输出......
...................
=========================================
shell -------------------------------- 8.02s
scp ---------------------------------- 3.37s
only one debug ----------------------- 0.33s
```

从结果可见，效率的提升非常明显。

使用mitogen时，有些配置可能和Ansible原生冲突，或需要做额外配置。比如：

- (1).原生Ansible允许使用forks设置最大并发节点数量，但mitogen默认固定最多32个连接，需要修改环境变量`MITOGEN_POOL_SIZE`的值来设置最大并发量。  
- (2).mitogen的sudo处理行为和Ansible不一样，所以可能需要单独在目标节点的sudoer配置中加入对应用户的配置。比如`your_ssh_username = (ALL) NOPASSWD:/usr/bin/python -c*`。  

使用mitogen后，不少Ansible的内部操作会发生变化，mitogen自然会以相同的结果不同的效率来完成目标，所以作为使用者，不用关心这些内部的事。

最后，如果要使用mitogen，建议阅读一次官方手册，了解使用mitogen时的一些注意事项。
