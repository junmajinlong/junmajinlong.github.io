---
title: 7.更大的舞台(2)：利用Role部署LNMP案例
p: ansible/7_role_examples.md
date: 2021-07-11 10:37:36
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------


> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。


# 7.更大的舞台(2)：利用Role部署LNMP案例

本文将介绍通过Role来搭建LNMP架构的案例，以便熟悉编写Role的逻辑和过程，文章最后还提供了一种优化执行效率的方案。

为了达到循序渐进的效果，最开始我会将本案例所有涉及到的功能写在单个Role当中，后面会将大功能拆分并分离到多个小Role中，这也是一种比较规范的编写Role的方式。但萝卜青菜各有所爱，有些人觉得是最佳实践，有些人则持反对意见，无所谓，只要有自己的想法并认为自己的好就行了。

文章内容较多，并不是想象中那样简简单单安装LNMP就完事，为了编写完美的Role，其中涉及很多重要的逻辑和知识点，甚至是一些理念，所以再次提醒，一定要做好笔记。

## 7.1 实验环境说明

本文的测试环境如下：

| 节点说明  | IP          | 系统版本   | 软件版本                 |
| :--------- | :-------- | :------- | :---------------------- |
| nginx节点 | 192.168.200.42 | CentOS 7.2 | nginx(rpm 1.16版)  |
| PHP节点   | 192.168.200.43 | CentOS 7.2 | PHP 7.4            |
| MySQL节点 | 192.168.200.44 | CentOS 7.2 | MySQL 5.7.22       |

建议在开始实验之前先准备干净的虚拟机(已经配好ssh主机互信)并做好快照，在后面测试的时候，很可能需要多次恢复原始环境。

## 7.2 创建Role并提供inventory

> 从此处开始，使用<https://github.com/malongshuai/ansible-column.git>中的"7th/myroles"目录。

我先将所有功能集中在单个Role中，后面会抽离、重整各类功能到不同的Role中。

创建Role的目录结构：

```shell
$ mkdir roles
$ ansible-galaxy init lnmp --force --init-path roles/
$ tree
.
└── roles
    └── lnmp
        ├── defaults
        │   └── main.yml
        ├── files
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── README.md
        ├── tasks
        │   └── main.yml
        ├── templates
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml
```

然后单独为LNMP这个功能配置一个inventory文件*inventory_lnmp*，它和*roles*目录在同一个目录下。初始时，这个inventory文件内容如下：

```ini
[nginx]
192.168.200.42

[php]
192.168.200.43

[mysql]
192.168.200.44

[dev:children]
nginx
php
mysql
```

## 7.3 配置yum镜像源

在开始安装并配置LNMP之前，第一个步骤就是配置各软件包的yum镜像源。在LNMP这个案例中，需要配置的包括：nginx官方源、php的remi和remisafe源、MySQL源。

配置yum源有多种方式，可以在Ansible控制端写好repo文件，然后拷贝到目标节点上，也可以使用Ansible模块`yum_repository`来配置，使用哪种方式可随意，这里我使用拷贝repo文件的方式。

为图简单，我将所有软件相关的yum镜像源全写在一个文件里，并且会分发给所有目标节点，更佳方式是将nginx的yum源分发给nginx节点，mysql镜像源分发给mysql节点。

步骤1：创建`lnmp/files/lnmp.repo`文件，内容如下：

```ini
[nginx]
name=nginx
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1

[remi]
name=remirepo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/$releasever/php74/$basearch/
enable=1
gpgcheck=0

[remisafe]
name=remisaferepo
baseurl=https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/$releasever/safe/$basearch/
enable=1
gpgcheck=0

[mysql57]
name=MySQL
baseurl=http://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el$releasever/
enabled=1
gpgcheck=0
```

上面repo文件中的baseurl指令使用了yum宏`$releasever`和`$basearch`，所以这个repo文件可以在CentOS 6和CentOS 7版本通用。

步骤2：创建`lnmp/tasks/add_repo.yml`文件，写一个任务去拷贝repo文件。其内容如下：

```yaml
---
- name: add yum_repo for nginx, php, mysql
  copy: 
    src: lnmp.repo
    dest: /etc/yum.repos.d/lnmp.repo
```

步骤3：在`lnmp/tasks/main.yml`中引入`add_repo.yml`任务文件，内容如下：
```yaml
---
- name: add_yum repo for nginx, php, mysql
  import_tasks: add_repo.yml
```

在此我使用的是`import_tasks`指令，使用`include_tasks`指令也没有任何问题。但个人建议，在可以使用`import_tasks`的时候均不要使用`include_tasks`，`include_tasks`指令限制比较多。

步骤4：提供一个入口playbook文件lnmp.yml来引入Role，它和roles目录在同一个层次。内容如下：
```yaml
---
- name: lnmp deploy
  hosts: dev
  gather_facts: false
  roles:
    - role: lnmp
```

然后进行测试：
```shell
$ ansible-playbook -i inventory_lnmp lnmp.yml
```

## 7.4 安装并配置Nginx

然后安装并配置nginx，相关的任务分布在各个步骤中，在nginx配置完成后，我会将所有任务进行简单的重构并整合在一起。

步骤1：创建`lnmp/tasks/nginx_install_config.yml`文件，在其中编写安装和配置nginx的任务。安装任务如下：
```yaml
---
- name: install nginx
  yum: 
    name: nginx
    state: present
  when: "'nginx' in group_names"
```

这里加了`when`条件判断，只有nginx组中的节点才执行任务，如果不加判断，由于play的`hosts`指令指定的是dev主机组，这会使得所有节点都安装nginx。

再介绍此处涉及到的变量`group_names`，它是Ansible的一个预定义特殊变量，之前也曾接触过几个预定义的特殊变量：`inventory_hostname, play_hosts, hostvars`。对于`group_names`变量，它保存了当前节点所在的主机组列表。例如，当192.168.200.42节点执行任务时，由于它属于nginx主机组和dev主机组，所以它获取到的`group_names`的值为`['nginx','dev']`，而192.168.200.43执行任务时，获取到的变量值为`['php','dev']`。

再来看`when: "'nginx' in group_names"`指令的条件，我想无需再解释了。

nginx安装完成后还需配置nginx，配置nginx的过程包括：  
- (1).在本地端写好配置文件，包括nginx服务进程的配置和虚拟主机(即站点)的配置  
- (2).将配置文件发送给目标节点  
- (3).如果可以的话，在目标节点上检查配置文件的语法  
- (4).启动nginx服务  
- (5).提供测试页面，并进行测试  

也可以直接修改远程的配置文件，但如果要修改的配置项较多，使用Ansible来做这样的操作就很不方便。所以，更多的是将本地端写好的配置文件拷贝到目标节点上。

步骤2：创建`lnmp/templates/nginx.conf.j2`文件作为稍后要拷贝到目标节点的nginx配置文件，文件内容如下(每个配置项的含义我不做解释，这是nginx的内容，而非Ansible的内容)：

```
user nginx;
worker_processes {{ worker_processes }};
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

worker_rlimit_nofile {{worker_rlimit_nofile}};

events {
  worker_connections {{ worker_connections }};
  multi_accept {{ multi_accept }};
}

http {
  sendfile {{ send_file }};
  tcp_nopush {{ tcp_nopush }};
  tcp_nodelay {{ tcp_nodelay }};

  keepalive_timeout {{ keepalive_timeout }};
  server_tokens {{ server_tokens }};
  include /etc/nginx/mime.types;

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log warn;

  gzip {{ gzip }};
  gzip_types {{ gzip_types | join(' ') if gzip_types is defined and gzip_types and (gzip_types | length > 0) else 'text/plain' }};
  gzip_min_length {{gzip_min_length}};
  gzip_disable "msie6";

  include /etc/nginx/conf.d/*.conf;
}

```

关于这个配置文件，有几点需要说明：  
- (1).配置文件中并没有将所有nginx指令的值直接配置好，而是使用了大量的变量进行占位；  
- (2).因为使用了变量占位，所以需要提供这些变量的值，可将其定义在变量文件`lnmp/vars/main.yml`中，文件具体的内容见下文；  
- (3).因为使用了变量占位，所以这个配置文件并不是直接发送给目标节点的，而是先将占位变量进行变量替换，替换过程称为渲染(render)，然后将渲染完成后的数据发送给目标节点。所以，这不是一个普通的文件，它应当称之为一个模板文件，应该使用template模块去处理这个文件并发送给目标节点，也因此将文件创建在`lnmp/templates/nginx.conf.j2`，以`.j2`后缀结尾表示这是Jinja2模板文件；  
- (4).在这个模板文件中，除了使用了变量占位，还在`gzip_types`指令行中使用了Jinja2表达式，Jinja2在Ansible的应用中扮演了一个分量很重的角色，在后面我会专门使用一篇文章来详细介绍它的功能和用法。稍后我会简单介绍`gzip_types`这一行中表达式的含义，让各位混个眼熟；
- (5).Role中使用变量占位是非常常见的方式，这种方式非常有助于后期的维护，也降低了配置和使用的难度，并且增强了Role的可移植性和通用性。比如，将这个Role发送给别人，别人只需要在变量文件`nginx_config.yml`中更改变量的值即可完成nginx的配置。

再来解释下模板文件中`gzip_types`指令中使用的Jinja2表达式是什么意思，目前各位只需看懂我的解释即可，不用追求会写，后面我会详细介绍。

```
gzip_types {{gzip_types|join(' ') if gzip_types is defined and gzip_types and (gzip_types|length > 0) else 'text/plain'}};
```

它的语法模式为：
```
<expr1> if <condition> else <expr2>
```

如果condition条件返回真，则执行expr1部分的代码并返回，如果condition条件返回假，则执行expr2部分的代码并返回。

对于此处的示例，`gzip_types is defined and gzip_types and (gzip_types | length > 0)`包含了三个条件判断： 
- (1).`gzip_types is defined`用于判断`gzip_types`变量是否已经定义；  
- (2).`and`后面的`gzip_types`等价于`gzip_types is not none`，用于判断变量`gzip_types`定义后是否有赋值或是否被赋值为null；
    - 在变量文件中将变量编写为`var1: `或`var1: null`，表示未赋值或赋值为null，它们等价；  
- (3).`(gzip_types|length > 0)`判断`gzip_types`的元素个数是否大于0，也即表示`gzip_types`为空列表时，将返回假。  
    - 在变量文件中`var1: []`表示赋值为空列表。  

所以，如果`gzip_types`未定义，或定义了但未赋值、赋值为null，或赋值为空列表时，则if条件判断均为假，于是执行else分支，即返回`text/plain`，template模块渲染时会将最终返回值替换到Jinja2代码位置处。所以，该行最终渲染后得到的是`gzip_types text/plain;`。

如果`gzip_types`已定义且赋值为非null、非空列表时，则if条件判断为真，于是执行`gzip_types|join(' ')`。在这里做了一个假设：`gzip_types`变量的值是一个列表，因为这个变量用于指定nginx压缩的文件类型，可能要指定多个要压缩的类型，所以从逻辑上它应当是一个列表。

此外，这里还使用了筛选器(filter)，即那根竖线`|`，它表示将竖线左边的返回值作为参数传递给竖线右边的筛选器函数进行处理，并最终返回。比如此处，将`gzip_types`变量的返回值也即变量自身的值当作参数传递给筛选器join()，join()筛选器的作用是将列表中的各个元素按照所指定的空格连接符连接起来，比如`[1,2,3,4]|join('_')`得到的是`1_2_3_4`。

各位可能会觉得有些晦涩难懂，毕竟是第一次接触到Jinja2里的筛选器，没关系，混个眼熟即可，后面还会详细介绍，现在将其当作管道来理解即可。

步骤3：然后提供模板中涉及到的变量，将它们定义在变量文件`lnmp/vars/main.yml`中，内容如下：  
```yaml
worker_processes: 1
worker_rlimit_nofile: 65535
worker_connections: 10240
multi_accept: "on"
send_file: "on"
tcp_nopush: "on"
tcp_nodelay: "on"
keepalive_timeout: 65
server_tokens: "off"
gzip: "on"
gzip_min_length: 1024
# 1. You should assign a list to gzip_types，if you don't
#    want to use this variable, set it to an empty list,
#    such as "gzip_types: []".
# 2. There is no need to add "text/html" type for gzip_types,
#    gzip will always include "text/html".
gzip_types: 
  - text/plain
  - text/css
  - text/javascript
  - application/x-javascript
  - application/xml
  - image/jpeg
  - image/jpg
  - image/gif
  - image/png
```

注意其中的`gzip_types`是一个列表。

步骤4：提供默认变量，将它们保存在`lnmp/defaults/main.yml`中，内容如下：  
```yaml
worker_processes: 1
worker_rlimit_nofile: 1024
worker_connections: 1024
multi_accept: "on"
send_file: "on"
tcp_nopush: "off"
tcp_nodelay: "on"
keepalive_timeout: 65
server_tokens: "on"
gzip: "off"
gzip_min_length: 1024
gzip_types: ['text/plain']
```

通常来说，对于配置文件里的变量都应当提供默认值，以保证即使在主变量文件中没有配置对应变量时，模板文件也能正确渲染。

比如上面变量文件中的`gzip_types`本应是一个列表，但因为用户不想要压缩功能或想要使用Nginx的默认值，于是将`gzip_types`变量设置为空甚至直接删除了这个变量，但因为在模板配置文件中已经明确使用了nginx的`gzip_types`指令，模板引擎必须得为其渲染出一个值出来，否则这个nginx配置文件就是语法错误。

所以，我在上面示例模板文件中加入了处理这类异常情况的if判断。更简单、更方面、更具可读性的方式，是直接为这些变量提供默认值，并在主变量文件中为需要注意的变量加上注释(正如我上面提供的`lnmp/vars/main.yml`文件加的注释一样)。

> 无论在哪门语言中，注释都显得无比重要，没想到Ansible中也是如此吧？所以，也尽量为你自己的playbook加上注释。另外，要为play和task设置好名称。

有了这些默认变量，且有了注释的保证，前面模板文件中的if判断语句就可以简化为：
```
{{ gzip_types | join(' ') if gzip_types|length > 0 else 'text/plain'}}
```

为了节省篇幅，我就不去再做修改，各位理解便可。

步骤5：既然nginx主配置文件相关内容都处理好了，接下来要编写一个使用`template`模块的任务，由该模板负责模板文件的渲染并将渲染好的内容拷贝到目标节点上。任务仍然写在`lnmp/tasks/nginx_install_config.yml`文件中，追加的内容如下：

```yaml
- name: render and copy nginx config
  template: 
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    backup: true
    validate: "/usr/sbin/nginx -t -c %s"
  when: "'nginx' in group_names"
  notify: "reload nginx"
```

template模块的`backup`参数，表示在拷贝文件之前先备份目标节点上已经存在的文件。

`validate`参数在不少模块中都存在，可用于数据校验，这是非常实用的功能。比如此例中的template模块，它表示文件拷贝到目标节点后但放置在目标位置之前(也就是说，拷贝的文件是先以临时文件方式存在的，其实Ansible大多涉及到目标文件的操作都是先保存在目标节点的临时文件中，然后再执行相关操作)，先执行`validate`参数指定的命令`nginx -t -c %s`，该命令的作用是检查拷贝过去的配置文件语法，`%s`表示拷贝后、替换前的目标文件名。如果validate校验成功，则将临时文件重命名为最终目标/etc/nginx/nginx.conf，如果校验失败，则Ansible端报错。过程如下图：

![](/img/ansible/1578111328578.png)

另外，修改了nginx配置文件后需要重启nginx(这里是第一次配置nginx，只需启动操作即可)，所以这里使用notify来触发一个重启nginx的handler，因此还需要编写这个handler。

步骤6：在`lnmp/handlers/main.yml`中编写`reload nginx`的handler，内容如下：

```yaml
- name: reload nginx
  service: 
    name: nginx
    state: reloaded
    enabled: true
```

service模块可以管理sysV和systemd的服务，换句话说，在CentOS 6和CentOS 7上都有效。

这里各位可能会想，第一次将nginx配置文件拷贝过去时，应当是启动或重启操作，而不是reload，以后再拷贝配置文件过去，才应该是reload。

逻辑确实应该如此，但各位可以去`ansible-doc -s service`中看看手册，`state: reloaded`状态的功能有二：  
- (1)如果服务已启动，则reload；  
- (2)如果服务未启动，则start。  

这完善的功能非常人性化。想象一下，如果service模块没有提供这个功能，就需要我们自己去判断Nginx节点上的nginx服务是否已启动。但要知道，如果想要远程判断的某个状态，在Ansible中没有原生提供相关功能，我们就只能自己使用命令的逻辑去判断(比如远程执行ps命令判断进程状态)，但这种判断方式比较麻烦。换句话说，Ansible在这一点上并不友好。

因为远程状态判断是一个非常常见的需求，所以这里临时转移一下话题，我将判断的方式提供给各位，以后肯定会有用武之地。如果暂时看不懂也没关系，因为经过下一篇文章之后肯定能看懂。

以获取进程状态为例：
```yaml
- name: get pids for sshd
  shell: |
    ps -ef | awk '/ssh[d]/{print $2}'
  register: pids
  failed_when: false
  changed_when: false

- name: print pids
  debug: 
    msg: "{{pids.stdout_lines | join(' ')}}"
  when: (pids.stdout_lines | length) > 0
```

另外，在Ansible 2.8中提供了`pids`模块，可以远程获取给定进程名的pid列表，但和这里的shell命令并没有什么区别，如有兴趣各位可`ansible-doc pids`。

回到正题，现在nginx主配置文件相关的内容都处理好了，接下来再提供nginx的站点配置文件，即使用Ansible配置nginx虚拟主机。

步骤7：移除nginx提供的默认虚拟主机文件`/etc/nginx/conf.d/default.conf`并提供我们自己编写的虚拟主机配置。相关任务稍后编写，现在先编写虚拟主机的模板配置文件`lnmp/templates/vhost.conf.j2`。

如果只需要提供一个虚拟主机，那么直接写虚拟主机文件即可，比如下面的：
```
server {
  listen       {{port}};
  server_name  {{server_name}};
  index index.html index.htm;
  location / {
    root {{document_root}};
  }
}
```

但我想更多场景下应该不止一个虚拟主机。所以，使用循环来生成多个虚拟主机的方式或许更佳。

所以，在`lnmp/vars/main.yml`变量文件中追加如下变量内容，表示要配置两个nginx的虚拟主机：server1和server2。
```yaml
vhosts:
  server1:
    server_name: www.abc.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:9000"
  server2: 
    server_name: www.def.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:9000"
```

这里使用了另一个Ansible预定义特殊变量`groups`，它保存了inventory中所有主机组以及每个组中的所有节点。它是一个字典结构，key为主机组名称，value为主机组中包含的节点列表。所以`group.php`表示找出php主机组中的节点列表，`group.php[0]`表示php组中第一个节点(示例中的php组只有一个节点)。

> 题外话：
> 发现不美好的地方了吗？按照Role规范，所有的变量只能写在固定的vars/main.yml里，但可能我们想要将各类变量分开存放，比如为Nginx主配置文件模板提供一个变量文件，为虚拟主机配置文件模板提供一个变量文件。
> 后文我重整这个案例的时候，会介绍如何去调整变量的位置。

然后再提供虚拟主机模板配置文件`lnmp/templates/vhost.conf.j2`，其完整的内容如下：
```
server {
    listen       {{item.value.listen}};
    server_name  {{item.value.server_name}};

    location / {
        root   /usr/share/nginx/html/{{item.key}};
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location ~ \.php$ {
        fastcgi_pass   {{item.value.fastcgi_pass}};
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME /usr/share/www/{{item.key}}/php$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

关于其中的`{{item.key}}`和`{{item.value}}`是什么意思，稍后写循环任务时会解释。

另外注意其中的几个root指令指定的目录，稍后需要写任务去创建它们。

其实，这个模板文件很不完善，大多数配置项都写死了，这非常不利于维护。但因为还没有深入介绍Jinja2，所以目前只能达到这种程度，后面介绍Jinja2语法的时候，我将会为各位完善这个模板，实现可以随意添加虚拟主机、随意添加配置项、随意添加一项或多项location的功能，敬请各位的期待。

步骤8：编写和虚拟主机相关的任务，这些任务仍然编写在`lnmp/tasks/nginx_install_config.yml`文件中。包括的任务有：  
- (1).渲染虚拟主机模板文件并拷贝到目标节点
    - server1的目标文件名为：/etc/nginx/conf.d/server1.conf  
    - server2的目标文件名为：/etc/nginx/conf.d/server2.conf  
- (2).移除nginx默认虚拟主机的文件/etc/nginx/conf.d/default.conf；  
- (3).创建每个虚拟主机涉及到的目录  
    - `location /`段涉及的目录为：/usr/share/nginx/html/server{1,2}  
    - `location ~ \*.php$`段涉及的目录为/usr/share/www/server{1,2}/php/，且在php节点上创建  

首先写移除Nginx默认虚拟主机文件的任务：
```yaml
- name: remove nginx default vhost file
  file: 
    name: /etc/nginx/conf.d/default.conf
    state: absent
  when: "'nginx' in group_names"
```

然后写渲染虚拟主机模板文件的任务：
```yaml
- name: render and copy vhosts config
  template: 
    src: vhost.conf.j2
    dest: "/etc/nginx/conf.d/{{item.key}}.conf"
    # validate: "/usr/sbin/nginx -t -c %s"
  loop: "{{vhosts|dict2items}}"
  when: "'nginx' in group_names"
  notify: "reload nginx"
```

这里有几项需要说明：  
1. template模块不要加上`validate`参数去检查语法，因为虚拟主机的配置文件是nginx配置文件的一部分，而且`%s`表示的是临时文件名，不算是完整的Nginx配置文件，只检查这部分文件的语法会报错。  
2. notify会触发`reload nginx`这个handler，它和之前拷贝nginx主配置文件时触发的是同一个handler，但并没有影响，因为handler是在当前阶段的任务执行完成后统一、且不重复执行的。  
3. 这里loop指令迭代的目标是{% raw %} vhosts|dict2items {% endraw %}，这里又用到了筛选器filter，这里简单解释下`dict2items`的作用。

在前面定义的vhosts变量是一个dict结构，在以前版本的Ansible中迭代字典的方式是使用`with_dict`指令，但现在都建议将`with_xxx`指令转换成`loop`指令迭代，而loop指令只能迭代列表，所以这里使用了`dict2items`筛选器将dict转换成列表形式。

例如这里执行{% raw %} vhosts|dict2items {% endraw %}之后，与之等价的结果是：
```yaml
- key: server1
  value: 
    server_name: www.abc.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:9000"
- key: server2
  value: 
    server_name: www.def.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:9000"
```

于是，`with_dict: "{{vhosts}}"`和{% raw %} loop: "{{vhosts|dict2items}}" {% endraw %}是等价的，迭代之后，可以通过`{{item.key}}`分别获取server1和server2，通过`{{item.value}}`分别获取对应虚拟主机的配置项。

然后再编写虚拟主机涉及到的目录的任务。这里有几个要求：  
- (1)./usr/share/nginx/html/server{1,2}目录在nginx节点上创建  
- (2)./usr/share/www/server{1,2}/php/目录在php节点上创建  

该任务可以写成如下不太友好的方式：
```yaml
- name: create dir on nginx
  file: 
    name: "/usr/share/nginx/html/{{item.key}}"
    state: directory
  loop: "{{vhosts|dict2items}}"
  when: "'nginx' in group_names"

- name: create dir on php
  file: 
    name: "/usr/share/www/{{item.key}}/php"
    state: directory
  loop: "{{vhosts|dict2items}}"
  when: "'php' in group_names"
```

之所以不友好是因为创建php相关目录本该是nginx节点上触发的任务，但此处却是在php节点执行任务时创建的，假如play的`hosts`指令中没有选中php主机组呢？换句话说，nginx节点触发的任务却脱离了nginx。

更好的方式是使用之前曾提到过的`delegate_to`指令进行任务委托，在nginx节点执行任务到此任务时，临时委托给php节点去创建php目录。

所以，改写成如下方式：
```yaml
- name: create dir on nginx
  file: 
    name: "/usr/share/nginx/html/{{item.key}}"
    state: directory
  loop: "{{vhosts|dict2items}}"
  when: "'nginx' in group_names"

- name: create dir on php
  file: 
    name: "/usr/share/www/{{item.key}}/php"
    state: directory
  loop: "{{vhosts|dict2items}}"
  when: "'nginx' in group_names"
  delegate_to: "{{groups.php[0]}}"
```

这里还能继续进行优化，我也想继续给各位完善下去，以便让各位能写出尽善尽美的Role，但实在篇幅受限，所以还是继续下面的步骤。

步骤9：提供虚拟主机的测试页面，这里分别为两个虚拟主机提供两个页面：index.html和index.php，它们放在`lnmp/files`目录下。

文件`lnmp/files/index.html`的内容如下：
```html
<h1>hello world from index.html</h1>
```

文件`lnmp/files/index.php`的内容如下：
```php
<?php echo "hello world from index.php"; ?>
```

步骤10：将测试页面拷贝到nginx节点的/usr/share/nginx/html/server{1,2}和php节点的/usr/share/www/server{1,2}/php/。在`lnmp/tasks/nginx_install_config.yml`文件中继续追加如下内容：

```yaml
- name: copy index.html
  copy: 
    src: "index.html"
    dest: "/usr/share/nginx/html/{{item.key}}"
  loop: "{{vhosts|dict2items}}"
  when: "'nginx' in group_names"

- name: copy index.php
  copy:
    src: "index.php"
    dest: "/usr/share/www/{{item.key}}/php"
  loop: "{{vhosts|dict2items}}"
  when: "'php' in group_names"
```

为什么这里又不使用委托的方式呢？使用任务委托也没问题，只不过这是测试用的任务，没必要太过讲究，而且从更佳实践的角度上考察，测试任务应当单独编写而不是和主任务放在一起，只不过在这个示例中我把所有任务都集中在单个Role中。

步骤11：最后做页面测试。因为php节点尚未部署，所以暂时只测试index.html。

```yaml
- name: flush handler
  meta: flush_handlers

- name: test web page
  uri:
    url: "http://{{item.value.server_name}}:{{item.value.listen}}"
  loop: "{{vhosts|dict2items}}"
  run_once: true
  delegate_to: localhost
```

在这里使用了`uri`模块，它可以用来测试http/https页面。如果只给定url参数，则默认GET该页面，并在响应状态码为200时表示测试成功。它还有更多、更灵活的用法，比如判断页面中是否包含了某个关键字，可在需要页面测试时去参考官方手册。

另外，测试任务是在Ansible端进行的，所以将这个任务委托给localhost执行。也因此注意，在测试时需要先在Ansible端/etc/hosts中添加DNS相关记录：
```
192.168.200.42 www.abc.com www.def.com
```

另一方面，测试只需进行一次就可以了，没必要重复测试，所以上面使用了`run_once`指令。这个指令的作用是只在第一个选中的目标节点上执行一次(严格地说是在第一批的第一个节点)，后续节点不再执行这个任务，但后续节点却能获取到第一个节点执行完成后的成果，比如在该任务中声明了一个变量，虽然后续节点未执行该任务，但后续节点也能获取到这个变量的值。

正如示例中，理论上它第一个选中的应该是nginx节点，在nginx节点执行到该任务时会委托给localhost执行一次，之后php节点和mysql节点虽然也选中了，但不会再执行到该任务，也就不会再委托。

`run_once`某些時候也有其他实现方式，比如：
```yaml
when: inventory_hostname == play_hosts[0]
```

但并非总是能等价，`run_once`自有其优点，比如这个示例中已经有了`run_once`，那么就可以继续使用when做其它判断。

现在测试页面已经写好了，似乎已经完事了，不过再想一想，在执行测试任务之前，nginx服务启动了吗？没有，因为这是第一次配置nginx，启动nginx的操作留给了handler来完成，但是handler是在当前阶段的所有任务执行完后才开始执行的。

所以，在测试之前应当先启动nginx。这里我使用了`meta: flush_handlers`任务，当执行到该任务时，它会立即去执行当前阶段已经触发的所有handler任务。注意，这是改变Ansible执行流程的一种方式，必须掌握。

关于`meta`，它是Ansible中一个非常神奇的模块，它的功能都直接作用在Ansible本身，虽然提供的功能不算多，但却能解决一些比较棘手的需求，比如示例中所用的`flush_handlers`可以立即去执行已经触发的handler，再比如`meta: end_play`可以立即终止当前play并进入下一个play，也就是说，当前play中剩下的任务不执行了，这也改变了任务的执行流程。此外还有几个功能，各位可参考官方手册：<https://docs.ansible.com/ansible/latest/modules/meta_module.html>去了解了解，以备不时之需。

最后，还有一个注意事项需要提醒一下，上面的测试任务是在nginx启动后立即执行的，对于这里的nginx reload来说这没什么问题，但对于某些启动较慢的服务，在启动任务后加上一个延迟行为，这样的效果是否会更佳？Ansible提供了`wait_for`模块和`pause`模块，前者用于等待指定的条件发生后才继续执行后续任务，后者用于等待指定的超时时间，它们用法都非常简单，各位可去看看官方手册，后续的文章中可能也有用上它们的机会。

比如，上面的测试任务可以如下方式修改：

```yaml
- name: flush handler
  meta: flush_handlers

# 等待80端口开启才继续执行后面的任务
# - wait_for:
#     port: 80
#     state: started

# 睡眠1秒后才继续执行后面的任务
# - pause: 
#     seconds: 1

- name: test web page
  uri:
    ...
```

步骤12：将`nginx_install_config.yml`引入到`lnmp/tasks/main.yml`文件中。

`lnmp/tasks/main.yml`文件中的内容为：
```yaml
---
# 添加yum源
- name: add_yum repo for nginx, php, mysql
  import_tasks: add_repo.yml

# 安装nginx、配置nginx、配置nginx虚拟主机、测试虚拟主机
- name: nginx config
  import_tasks: nginx_install_config.yml
```

步骤13：执行Role进行测试。

因为当前已经编写的所有任务都具有幂等性，所以现在执行测试且多次执行也没有任何影响。

```shell
$ ansible-playbook -i inventory_lnmp lnmp.yml
```

如果测试一切OK，那么进行最后一步，整理众多的task。

## 7.5 整理nginx的众多任务

目前为止，绝大多数任务都写在`lnmp/tasks/nginx_install_config.yml`中，但这里面的任务有不少指令重复了，比如几乎所有任务中都带上了when判断只在nginx节点上执行，比如和虚拟主机配置相关的任务中都有{% raw %} loop: "{{vhosts|dict2items}}" {% endraw %}迭代指令。对于重复的指令，如果条件允许，都建议将其抽取出来放在更高层次以避免重复书写，比如写在引入任务文件的指令层次上。再者，前面曾提到过测试任务和主任务写在一起并不合理，所以应当将它们分开。

在这个步骤中我会在nginx Role层次对这三方面问题进行整理。

第一步，将`nginx_install_config.yml`中重复的when指令移除，并将when添加在`lnmp/tasks/main.yml`引入该任务文件的指令处。

修改后的`lnmp/tasks/main.yml`文件内容为：
```yaml
---
# 添加yum源
- name: add_yum repo for nginx, php, mysql
  import_tasks: add_repo.yml

# 安装nginx、配置nginx、配置nginx虚拟主机、测试虚拟主机
- name: nginx config
  import_tasks: nginx_install_config.yml
  when: "'nginx' in group_names"
```

因为使用的是`import_tasks`引入方式，所以when指令会拷贝到所有子任务上。

第二步，将`nginx_install_config.yml`中重复的{% raw %} loop: "{{vhosts|dict2items}}" {% endraw %}任务抽取出来，写在另一个任务文件`nginx_vhost_config.yml`中，然后`include_tasks`引入该文件，并加上loop指令。

所以，整理后的`nginx_install_config.yml`文件内容如下：(因为测试任务要单独编写，所以这里也将测试相关任务移除了)
```yaml
---
- name: install nginx
  yum: 
    name: nginx
    state: present

- name: render and copy nginx config
  template: 
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    backup: true
    validate: "/usr/sbin/nginx -t -c %s"
  notify: "reload nginx"

- name: nginx vhost config
  include_tasks: nginx_vhost_config.yml
  loop: "{{vhosts|dict2items}}"
  loop_control: 
    extended: yes
```

注意其中的`loop_control`，稍后我会解释为什么要加上它。

其中`include_tasks`所引入的`nginx_vhost_config.yml`文件的内容如下：
```yaml
# 注意这个任务的改变
- name: remove nginx default vhost file
  file: 
    name: /etc/nginx/conf.d/default.conf
    state: absent
  when: ansible_loop.first

# 下面这些任务除了移除了when指令和loop指令，没有任何变化
- name: render and copy vhosts config
  template: 
    src: vhost.conf.j2
    dest: "/etc/nginx/conf.d/{{item.key}}.conf"
  notify: "reload nginx"

- name: create dir on nginx
  file: 
    name: "/usr/share/nginx/html/{{item.key}}"
    state: directory

- name: copy index.html
  copy: 
    src: "index.html"
    dest: "/usr/share/nginx/html/{{item.key}}"

- name: create dir on php
  file: 
    name: "/usr/share/www/{{item.key}}/php"
    state: directory
  delegate_to: "{{groups.php[0]}}"

- name: copy index.php
  copy:
    src: "index.php"
    dest: "/usr/share/www/{{item.key}}/php"
  delegate_to: "{{groups.php[0]}}"
```

这个文件里唯一需要注意的是第一个任务：移除nginx默认虚拟主机文件。这个任务原本是没有带{% raw %} loop: "{{vhosts|dict2items}}" {% endraw %}指令的，但它也属于nginx虚拟主机配置相关的任务，所以也抽取到这个文件中。为了让这个原本不在循环内的任务只执行一次，所以加了循环判断`when: ansible_loop.first`，这表示只在第一轮loop循环时执行，之后每轮loop循环都不再执行。由于使用了`ansible_loop.first`这个循环过程中的信息，所以在前面loop指令中还需加上`loop_control`的extended参数。仍然是那句话，如果这里暂时看不懂也没关系，后面在进阶深入Ansible的章节中会系统性介绍。

第三步，单独编写测试任务，将它与主任务分离。

编写测试任务的逻辑有很多种，比如在入口playbook文件中编写，或者单独写一个测试任务文件test.yml，或者将测试任务写在Role任务执行流程的尾部，等等。但我想，将测试Role的任务规范化应该更佳，比如将所有的测试任务放在Role的test目录中。

不知各位是否曾注意到，在`ansible-galaxy init`创建Role结构的时候，同时也会创建tests目录，并且在此目录下有inventory文件和test.yml文件：
```
$ tree roles/lnmp/tests/
roles/lnmp/tests/
├── inventory
└── test.yml
```

在inventory文件中可以提供测试所用的inventory，在test.yml中可以编写测试所用的playbook。

其实这两个文件中初始时已经有一些内容，比如对于lnmp这个Role，它的test.yml初始内容如下：
```
$ cat roles/lnmp/tests/test.yml 
---
- hosts: localhost
  remote_user: root
  roles:
    - lnmp
```

对于手动的Ansible测试，上面的文件测试时会报错，因为在tests目录下找不到`roles/lnmp`这个Role。如果借助测试工具或者自己写测试脚本则可以改变测试逻辑。

不过对于这里手动测试，对这个测试文件稍作修改即可。下面是我修改后的test.yml文件内容：

```yaml
---
- hosts: dev
  gather_facts: false
  roles:
    - ../../lnmp

- name: test nginx
  hosts: dev
  gather_facts: false
  vars_files:
    - ../../lnmp/vars/main.yml
  tasks:
    - name: test web pages
      uri:
        url: "http://{{item.value.server_name}}:{{item.value.listen}}"
      loop: "{{vhosts|dict2items}}"
      run_once: true
      delegate_to: localhost
      tags: test
```

然后执行test.yml来测试：
```shell
$ ansible-playbook roles/lnmp/tests/test.yml -i inventory_lnmp --tags test
```

至此，nginx配置部署算是完成了，真正需要编写的内容并不多，但这其中花了很大的篇幅来描述这个部署过程，希望各位能从繁琐的知识点中整理好自己的笔记并将整个过程在自己的脑海里化繁为简。

在经历了部署nginx这个复杂的块头之后，接下来再部署的PHP和MySQL节点，相对来说要简单许多。

## 7.6 安装并配置PHP

安装、配置PHP的过程出奇的简单，因为做实验只需修改下php-fpm的监听地址和端口即可，其它没什么可配置的^_^。如果各位确实想要修改配置，使用sed命令或lineinfile模块去修改，或者按照配置nginx时的逻辑，将配置项定义成变量占位方式并提供变量文件即可。

因为任务全写在单个Role中，所以创建`lnmp/tasks/php_install_config.yml`，内容如下：
```yaml
---
- name: install php and php-fpm
  yum: 
    name: "{{item}}"
    state: installed
  loop:
    - php
    - php-fpm

- name: change php-fpm listen address and port
  shell: |
    sed -ri 's/^(listen *= *)127.0.0.1.*$/\1{{phpfpm_addr}}:{{phpfpm_port}}/' /etc/php-fpm.d/www.conf
    sed -ri 's/^(listen.allowed_clients.*)$/;\1/' /etc/php-fpm.d/www.conf

- name: restart php-fpm
  service:
    name: php-fpm
    state: reloaded
```

因为修改了php-fpm的监听地址，并且使用的是变量占位方式，所以在`lnmp/vars/main.yml`中提供相关变量，追加内容如下：
```yaml
phpfpm_addr: 0.0.0.0
phpfpm_port: 9000
```

然后在`lnmp/tasks/main.yml`中引入该任务文件，追加的内容如下：
```yaml
- name: php config
  import_tasks: php_install_config.yml
  when: "'php' in group_names"
```

注意php配置任务只在php节点执行，所以引入时加上when条件判断。

然后执行测试：
```yaml
$ ansible-playbook -i inventory_lnmp lnmp.yml
```

顺带着，可以测试一下index.php页面。在`lnmp/tests/main.yml`中添加：
```yaml
- name: test web pages
  uri:
    url: "http://{{item.value.server_name}}:{{item.value.listen}}/index.php"
  loop: "{{vhosts|dict2items}}"
  run_once: true
  delegate_to: localhost
  tags: test
```

然后执行：
```shell
$ ansible-playbook roles/lnmp/tests/test.yml -i inventory_lnmp --tags test
```

上面测试index.php页面的任务和之前测试index.html页面的任务几乎是重复定义的，唯一不同的是在URL尾部加上了"/index.php"后缀，其实应该用循环的方式来迭代测试index.html和index.php，这样可以将两个任务合并。

只是前面已经存在了{% raw %} loop: "{{vhosts|dict2items}}" {% endraw %}循环，再加入一个循环迭代，就得将嵌套循环派上用场。

在以前版本中，嵌套循环使用`with_nested`指令来实现，现在可以使用`loop`指令加`product`筛选器实现。下面两种方式是等价的：(混个眼熟吧)

```yaml
- name: test web pages
  uri:
    url: "http://{{item[0].value.server_name}}:{{item[0].value.listen}}/{{item[1]}}"
  with_nested:
    - "{{vhosts|dict2items}}"
    - ['index.html', 'index.php']
  run_once: true
  delegate_to: localhost
  tags: test

- name: test web pages
  uri:
    url: "http://{{item[0].value.server_name}}:{{item[0].value.listen}}/{{item[1]}}"
  loop: "{{vhosts|dict2items|product(['index.html','index.php'])|list }}"
  run_once: true
  delegate_to: localhost
  tags: test
```

## 7.7 安装并配置MySQL

最后部署MySQL。部署MySQL其实......算了，不多说废话了，直接开始吧。

步骤1：创建一个任务文件`lnmp/tasks/mysql_install_config.yml`，其中编写安装mysql server、mysql client、python2-PyMySQL、MySQL-python相关的任务。

其中python2-PyMySQL包和MySQL-python包是使用Ansible MySQL相关模块时需要的，前者安装时指定为`python-PyMySQL`即可，它在epel镜像源中，所以需要先设置好epel镜像源。

任务文件`lnmp/tasks/mysql_install_config.yml`的内容如下：
```yaml
---
- name: install mysql
  yum: 
    name: "{{item}}"
    state: installed
  loop: 
    - mysql-community-server
    - mysql-community-client
    - python-PyMySQL
    - MySQL-python
```

然后在`lnmp/tasks/main.yml`中引入该任务文件，所以在此文件中追加如下内容：
```yaml
- name: mysql config
  import_tasks: mysql_install_config.yml
  when: "'mysql' in group_names"
```

步骤2：提供MySQL的模板配置文件`lnmp/templates/mysql.cnf.j2`。

模板文件mysql.cnf.j2内容如下：
```yaml
[client]
socket = {{mysql.client.socket}}

[mysqldump]
max_allowed_packet = {{mysql.mysqldump.max_allowed_packet}}

[mysqld]
port = {{mysql.mysqld.port}}
datadir = {{mysql.mysqld.datadir}}
socket = {{mysql.mysqld.socket}}
server_id = {{mysql.mysqld.server_id}}
log-bin = {{mysql.mysqld.log_bin}}
sync_binlog = {{mysql.mysqld.sync_binlog}}
binlog_format = {{mysql.mysqld.binlog_format}}
character-set-server = {{mysql.mysqld.character_set_server}}
skip_name_resolve = {{mysql.mysqld.skip_name_resolve}}
pid-file = {{mysql.mysqld.pid_file}}
log-error = {{mysql.mysqld.log_error}}

```

注意，变量名中一定不能出现`-`，因为YAML中的`-`是保留符号，它表示定义一个列表，因此在key中不能出现短横线。

步骤3：在`lnmp/vars/main.yml`中提供MySQL配置文件中涉及的变量。

在变量文件`lnmp/vars/main.yml`中追加如下内容：
```yaml
mysql:
  client: 
    socket: "/data/mysql.sock"
  mysqldump: 
    max_allowed_packet: "32M"
  mysqld: 
    port: 3306
    datadir: "/data"
    socket: "/data/mysql.sock"
    server_id: 100
    log_bin: "mysql-bin"
    sync_binlog: 1
    binlog_format: "row"
    character_set_server: "utf8mb4"
    skip_name_resolve: 1
    pid_file: "/data/mysql.pid"
    log_error: "/data/error.log"
```

在这个变量中`/data`重复出现了多次，或许有人会想，将重复部分以变量的方式抽取出来多好，但很遗憾，YAML自身并不支持在同一个结构当中引用兄弟元素的值。

换句话说，下面这个结构中，没有任何办法将/home/junmajinlong这个重复的部分抽取出来并复用到rubyhome上(我翻遍网上资料也没找到方案，如果你知道请一定告诉我)。
```yaml
- path:
    home: /home/junmajinlong
    rubyhome: /home/junmajinlong/ruby
```

但不同结构中是可以引用的，比如下面的示例：
```yaml
- home: /home/junmajinlong
- rubyhome: "{{home}}/ruby"
```

此外，如果只是简简单单的引用目标元素，而不连接其它字符串时，还可以使用YAML的锚定和引用(也称别名alias)功能，即使是引用兄弟元素也能实现。关于YAML的锚定和别名用法，在编程时用的比较多，在Ansible中用的很少很少，所以这里我只给几个示例稍作介绍，各位如有兴趣可自行搜索相关资料。

例如，下面的YAML语法是正确的：
```yaml
- path: 
    p1: 
      home: &refme /home/junmajinlong
      rubyhome: *refme
    p2: 
      home: *refme
      pythonhome: *refme
```

这个结构等价于：
```yaml
- path: 
    p1: 
      home: /home/junmajinlong
      rubyhome: /home/junmajinlong
    p2: 
      home: /home/junmajinlong
      pythonhome: /home/junmajinlong
```

其中`&`符号表示创建一个锚定，`*`表示引用指定的锚定。

注意，引用锚时不能额外连接字符串，只能单独引用。例如`*refme/ruby`或`*refme"/ruby"`都是错误的语法，YAML自身也没有任何方法实现这样的功能。

步骤4：在`mysql_install_config.yml`中编写任务，渲染配置文件，并在MySQL节点创建配置文件中涉及到的datadir目录，并设置该目录的所有者和所属组为mysql。

追加的内容如下：
```yaml
- name: render mysql config
  template:
    src: mysql.cnf.j2
    dest: /etc/my.cnf
  notify: restart mysql

- name: create mysql datadir
  file: 
    name: "{{mysql.mysqld.datadir}}"
    state: directory
    owner: mysql
    group: mysql
```

步骤5：在`lnmp/handlers/main.yml`中编写restart mysql这个handler任务。

追加的内容如下：
```yaml
- name: restart mysql
  service:
    name: mysqld
    state: restarted
```

注意，mysqld服务不支持reload操作。

步骤6：从MySQL 5.7开始，初始化mysql(或第一次启动)时会创建临时密码并保存在MySQL的error log中，之后要连接到MySQL进行操作，必须先使用`ALTER USER`语句修改`root@localhost`的密码。既然要使用Ansible来完成MySQL的初始化配置，它应当要去完成这个任务。

所以，先从远程MySQL节点获取到这个密码，然后使用mysql的`ALTER USER`语句更新密码。于是，在`mysql_install_config.yml`中继续编写任务：

```yaml
- block:
    - name: get initialize temp password
      shell: |
        sed -rn '/temp.*pass/s/^.*root@localhost: (.*)$/\1/p' {{mysql.mysqld.log_error}}
      register: tmp_passwd

    - name: modify root@localhost password before any op
      shell: |
        mysql -uroot -p'{{tmp_passwd.stdout}}' \
        --connect-expired-password \
        -NBe \
        'ALTER USER "root"@"localhost" identified by "{{mysql.mysql_passwd}}";'
  tags: 
    - never
```

上面第二个任务中使用了变量`mysql.mysql_passwd`来指定修改后的密码，所以在`lnmp/vars/main.yml`中加入这个变量：
```yaml
mysql:
  mysql_passwd: "P@ssword1!"
  client: 
    socket: "/data/mysql.sock"
  mysqldump: 
    max_allowed_packet: "32M"
  mysqld: 
    port: 3306
    ......
```

注意，此处为了简单，MySQL密码直接以明文方式保存在变量文件中，后面会有一篇文章介绍如何使用Ansible Vault来加密敏感数据，以保证安全性。

然后解释下block中的这两个任务。

第一个任务使用了shell模块执行sed命令来筛选远程mysql节点上创建的临时密码，这个命令的执行结果(即密码)是稍后要在Ansible中继续使用的，所以应当将其保存下来。

使用`register`指令可以将模块执行后的返回值注册为一个变量，比如上面示例中是将shell模块的返回值保存在`tmp_passwd`变量中，之后的任务比如修改密码的操作就可以使用`tmp_passwd`这个变量。

每个模块的返回值类型都不相同，具体可参考对应模块的手册。对于shell模块执行的远程命令来说，经常会将其结果使用register注册成变量，所以有必要了解它的返回结果中包含了哪些内容。对于此，测试便知。

```yaml
---
- hosts: localhost
  gather_facts: false
  tasks: 
    - shell: echo hahaha
      register: result
    - debug: 
        var: result
```

执行结果：
```shell
$ ansible-playbook a.yml
......
TASK [debug] **************************
ok: [localhost] => {
    "result": {
        "changed": true,
        "cmd": "echo hahaha",
        "delta": "0:00:00.001855",
        "end": "2019-12-29 02:02:40.411330",
        "failed": false,
        "rc": 0,
        "start": "2019-12-29 02:02:40.409475",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "hahaha",
        "stdout_lines": [
            "hahaha"
        ]
    }
}
......
```

从结果中不难发现，shell模块的返回值中包含了多项，它们都保存在一个字典结构中，包括：  
- (1).所执行的命令cmd  
- (2).开始执行的时间start、执行结束的时间end以及执行命令花掉的时间delta  
- (3).是否失败failed  
- (4).命令的退出状态码rc，Ansible默认只认0退出状态码为正确，其它状态码均报错处理  
- (5).标准错误stderr以及列表方式保存的标准错误行stderr_lines  
- (6).标准输出stdout以及列表方式保存的标准输出行stdout_lines  
- (7).shell模块的changed状态(对于shell模块，只要命令执行了，changed状态一定为true)  

所以，要获取命令的输出结果，只需使用`result.stdout`即可：

```yaml
- debug:
    var: result.stdout
```

执行结果：
```shell
$ ansible-playbook a.yml
......
TASK [debug] *********************
ok: [localhost] => {
    "result.stdout": "hahaha"
}
......
```

再回到block这个指令。它组织了两个比较特殊的任务，特殊之处在于修改临时密码是只在第一次连接MySQL时执行的，其它时候均不会执行，所以在block上加了一个名为`never`的特殊标签，这个特殊标签的作用是：只有在ansible-playbook命令行中显式指定了`--tags never`时才会执行带有这个标签的任务，其它时候均不执行这些任务。

对于本例来说，修改MySQL临时密码的操作只应在第一次执行，所以第一次执行playbook时需以如下方式执行：
```shell
$ ansible-playbook -i inventory_lnmp --tags "all,never" lnmp.yml
```

以后再执行则不用加never标签，直接执行即可，它会自动跳过所有带never标签的任务，：
```shell
$ ansible-playbook -i inventory_lnmp lnmp.yml
```

还有几个其它的特殊标签：all、never、tagged、untagged、always，它们的含义参考官方手册：[特殊tag](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#special-tags) 。

另一种实现任务只执行一次的方式是在变量文件中提供一个状态变量，比如`temp_passwd_updated`，初始时该变量值为0，当修改密码后通过lineinfile模块来修改这个变量的值为1(该任务委托给Ansible端即可)，然后在修改临时密码的block层次上加一个when指令，判断该变量的值。在后文重整Role的时候将采用这种方式。

步骤7：对于刚安装的MySQL，除了修改密码外，还应移除不安全用户，创建某些用户，创建、删除某些数据库，等等。(注：MySQL 5.7初始化后就已经移除了test数据库和不安全的用户)

这些操作可以使用shell模块执行mysql命令或mysqladmin命令来完成。这里需要提醒各位，在Ansible的shell模块中使用mysql命令远程操作时，可以考虑加上`-N -B`这两个选项：-N选项表示不输出查询的字段名，-B表示不输出边框。

对比下就知道这两个选项的作用：

```shell
$ mysql -uroot -pP@ssword1! -e'select user,host from mysql.user;'
+---------------+-----------+
| user          | host      |
+---------------+-----------+
| mysql.session | localhost |
| mysql.sys     | localhost |
| root          | localhost |
+---------------+-----------+

$ mysql -uroot -pP@ssword1! -Ne'select user,host from mysql.user;'
+---------------+-----------+
| mysql.session | localhost |
|     mysql.sys | localhost |
|          root | localhost |
+---------------+-----------+

$ mysql -uroot -pP@ssword1! -Be'select user,host from mysql.user;'
user    host
mysql.session   localhost
mysql.sys       localhost
root    localhost

$ mysql -uroot -pP@ssword1! -NBe'select user,host from mysql.user;'
mysql.session   localhost
mysql.sys       localhost
root    localhost
```

除了使用shell模块远程执行mysql命令，也可以使用Ansible为MySQL提供的几个模块：  
- `mysql_db`：用于创建、删除MySQL数据库  
- `mysql_info`：用于收集MySQL服务的信息  
- `mysql_user`：用于创建、删除MySQL上的用户以及权限管理  
- `mysql_variables`：用于管理MySQL的全局变量  
- `mysql_replication`：用于管理MySQL replication相关功能，比如获取slave、更改master、启停slave等等  

通过这些模块，可以在远程通过python mysql相关的库连接到MySQL上并执行操作。注意，不是Ansible端直接连接远端MySQL，而是将连接的任务分发到MySQL节点上，并在MySQL节点上进行本地连接。

在使用这些模块时，有两个注意事项：  
- (1).要求目标节点上已经安装了python-PyMySQL、MySQL-python  
- (2).既然要连接MySQL，就需要提供用户名和密码，可以使用这些模块的`login_user`和`login_password`参数来指定，如果不指定，则首先从MySQL本地的`~/.my.cnf`中读取，如果MySQL节点没有该文件或没有想读取的内容，则默认以root用户、空密码进行连接  

如果使用`~/.my.cnf`文件提供连接时的用户名和密码，则至少需要提供如下内容：
```ini
[client]
user = ...
password = ...
```

所以，为了以后能使用Ansible操作MySQL，写一个任务来创建这个远程文件，于是在`lnmp/tasks/mysql_install_config.yml`中追加如下任务：
```yaml
- name: set mysql connection info for ansible
  template:
    src: .my.cnf.j2
    dest: /root/.my.cnf
```

并创建`lnmp/templates/.my.cnf.j2`文件，内容如下：
```
[client]
user = root
password = {{mysql.mysql_passwd}}
socket = {{mysql.mysqld.socket}}
```

然后就可以使用这些MySQL模块来执行操作。比如，管理MySQL用户和权限、创建和删除数据库、备份和恢复数据库，等等。

下面是一些示例，更多用法请参见这些模块的官方手册。

```yaml
# 移除localhost的匿名用户
- name: remove anonymous user for localhost
  mysql_user:
    name: ""
    host: localhost
    state: absent

# 移除所有匿名用户
- name: remove all anonymous user
  mysql_user:
    name: ""
    host_all: true
    state: absent

# 手动指定连接MySQL的用户名和密码，然后创建新用户并指定权限
- name: create user and grant privileges
  mysql_user: 
    name: "junmajinlong"
    host: "192.168.200.%"
    password: 'P@ssword2!'
    priv: '*.*:ALL'
    state: present
    login_user: root
    login_password: 'P@ssword1!'

# 创建数据库
- name: Create new databases with names 'foo' and 'bar'
  mysql_db:
    name: test
    state: present

# 创建多个数据库
- name: Create new databases with names 'foo' and 'bar'
  mysql_db:
    name:
      - foo
      - bar
    state: present

# 删除test数据库
- name: drop test database
  mysql_db: 
    name: test
    state: absent

# 删除多个数据库
- name: drop databases with names 'foo' and 'bar'
  mysql_db:
    name:
      - foo
      - bar
    state: absent

# 备份多个数据库
- name: Dump multiple databases
  mysql_db:
    state: dump
    name: foo,bar
    target: /tmp/dump.sql

# 备份数据库并压缩为.bz2、.xz或.gz格式
- name: Dump multiple databases
  mysql_db:
    state: dump
    name: foo,bar
    target: /tmp/dump.sql.bz2

# 恢复数据库，需要先将文件拷贝到MySQL节点
- name: Copy database dump file
  copy:
    src: /tmp/dump.sql.bz2
    dest: /tmp

- name: Restore database
  mysql_db:
    name: my_db
    state: import
    target: /tmp/dump.sql.bz2
```

至此，整个LNMP结构就部署完成了，这漫长的过程并不像想象中那样简单，其中涉及了很多逻辑和知识点，好在现在终于完成了所有任务。但是真的结束了吗？并没有。在前面我将所有的任务全都集中在单个Role中，这些任务数量多了，虽然已经分别保存在不同的文件当中，但仍显得混乱。另一方面，前面所有的变量都定义在单个变量文件`vars/main.yml`中，这些变量也很混乱。

所以，更佳方式是将各类任务分别定义成Role来自治。

## 7.8 任务分离到多个Role中

> 从此处开始，使用<https://github.com/malongshuai/ansible-column.git>中的"7th/lnmp"目录。

既然要分离成多个Role，这里我准备将安装配置nginx、PHP、MySQL的任务分别分离为三个Role。

另外，有些任务是所有节点上都执行的，比如配置SSH主机互信、配置时间同步、配置防火墙等等，这类通用性任务通常会分离成另一个称为common的Role(当然，名称是随意的，common比较见名知意)。这里我也划分一个common的Role，用来保存配置yum源的任务。

所以，重新组织的目录结构如下：
```shell
$ tree -L 2 -F lnmp/ 
lnmp/
├── inventory_lnmp
├── lnmp.yml
└── roles/
    ├── common/
    ├── mysql/
    ├── nginx/
    └── php/
```

先编写入口playbook文件`lnmp/lnmp.yml`，内容如下：
```yaml
---
- name: common config
  hosts: all
  gather_facts: false
  roles: 
    - common
  tags: 
    - common

- name: install and config nginx
  hosts: nginx
  gather_facts: false
  roles: 
    - nginx
  tags: 
    - nginx

- name: install and config PHP
  hosts: php
  gather_facts: false
  roles: 
    - php
  tags: 
    - php

- name: install and config mysql
  hosts: mysql
  gather_facts: false
  roles: 
    - mysql
  tags: 
    - mysql
```

然后再整理各个Role。各个Role中大多数任务的内容基本没有改变，只是换了存放位置，所以可快速浏览下面的内容。

### 7.8.1 common Role

common Role的内容很简单，仅仅只是添加yum源。

所以，只需一个`common/tasks/main.yml`任务文件即可，其它全删掉。
```shell
$ rm -rf lnmp/roles/common/{defaults,vars,files,templates,handlers,tests,meta,README.md}
```

文件`common/tasks/main.yml`的内容：
```yaml
---
# add yum repos
- name: backup origin yum repos
  shell: 
    cmd: "mkdir bak; mv *.repo bak"
    chdir: /etc/yum.repos.d
    creates: /etc/yum.repos.d/bak

- name: add os,epel,nginx,remi,remi-save,mysql repos
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
    - name: nginx
      baseurl: "http://nginx.org/packages/centos/$releasever/$basearch/"
    - name: remi
      baseurl: "https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/$releasever/php74/$basearch/"
    - name: remisafe
      baseurl: "https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/$releasever/safe/$basearch/"
    - name: mysql57
      baseurl: "http://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el$releasever/"
```

### 7.8.2 Nginx Role

nginx Role只有安装、提供主配置文件、虚拟主机配置文件的任务，提供index.html测试页面并测试的任务并没有包含在此。如果需要，可在测试时提供。

把用不上的目录删掉：
```shell
$ rm -rf lnmp/roles/nginx/{defaults,tests,files,meta,README.md}
```

文件`lnmp/roles/nginx/tasks/main.yml`内容：
```yaml
---
- name: install and config nginx
  import_tasks: nginx_install_config.yml
```

文件`lnmp/roles/nginx/tasks/nginx_install_config.yml`内容：
```yaml
---
- name: install nginx
  yum: 
    name: nginx
    state: present

- name: render and copy nginx config
  template: 
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    backup: true
    validate: "/usr/sbin/nginx -t -c %s"
  notify: "reload nginx"

- name: nginx vhost config
  include_tasks: nginx_vhost_config.yml
  loop: "{{vhosts|dict2items}}"
  loop_control: 
    extended: yes
```

文件`lnmp/roles/nginx/tasks/nginx_vhost_config.yml`内容：
```yaml
---
- name: remove nginx default vhost file
  file: 
    name: /etc/nginx/conf.d/default.conf
    state: absent
  when: ansible_loop.first

- name: render and copy vhosts config
  template: 
    src: vhost.conf.j2
    dest: "/etc/nginx/conf.d/{{item.key}}.conf"
  notify: "reload nginx"

- name: create dir on nginx
  file: 
    name: "/usr/share/nginx/html/{{item.key}}"
    state: directory

- name: create dir on php
  file: 
    name: "/usr/share/www/{{item.key}}/php"
    state: directory
  delegate_to: "{{groups.php[0]}}"
```

`lnmp/roles/nginx/handlers/main.yml`文件内容：

```yaml
---
- name: reload nginx
  service: 
    name: nginx
    state: reloaded
    enabled: true
```

`lnmp/roles/nginx/templates/nginx.conf.j2`文件内容：

```
user nginx;
worker_processes {{ worker_processes }};
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

worker_rlimit_nofile {{worker_rlimit_nofile}};

events {
  worker_connections {{ worker_connections }};
  multi_accept {{ multi_accept }};
}

http {
  sendfile {{ send_file }};
  tcp_nopush {{ tcp_nopush }};
  tcp_nodelay {{ tcp_nodelay }};

  keepalive_timeout {{ keepalive_timeout }};
  server_tokens {{ server_tokens }};
  include /etc/nginx/mime.types;

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log warn;

  gzip {{ gzip }};
  gzip_types {{ gzip_types | join(' ') if gzip_types | length > 0 else 'text/plain'}};
  gzip_min_length {{gzip_min_length}};
  gzip_disable "msie6";

  include /etc/nginx/conf.d/*.conf;
}

```

`lnmp/roles/nginx/templates/vhost.conf.j2`文件内容：
```
server {
    listen       {{item.value.listen}};
    server_name  {{item.value.server_name}};

    location / {
        root   /usr/share/nginx/html/{{item.key}};
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location ~ \.php$ {
        fastcgi_pass   {{item.value.fastcgi_pass}};
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME /usr/share/www/{{item.key}}/php$fastcgi_script_name;
        include        fastcgi_params;
    }
}

```

`lnmp/roles/nginx/vars/main.yml`文件内容：

```yaml
---
worker_processes: 1
worker_rlimit_nofile: 65535
worker_connections: 10240
multi_accept: "on"
send_file: "on"
tcp_nopush: "on"
tcp_nodelay: "on"
keepalive_timeout: 65
server_tokens: "off"
gzip: "on"
gzip_min_length: 1024
# 1. You should assign a list to gzip_types，if you don't
#    want to use this variable, set it to an empty list,
#    such as "gzip_types: []".
# 2. There is no need to add "text/html" type for gzip_types,
#    gzip will alway include "text/html".
gzip_types:
  - text/plain
  - text/css
  - text/javascript
  - application/x-javascript
  - application/xml
  - image/jpeg
  - image/jpg
  - image/gif
  - image/png

vhosts:
  server1:
    server_name: www.abc.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:9000"
  server2: 
    server_name: www.def.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:9000"
```

有一点需要说明，上面变量文件中的9000端口本是PHP Role中定义的变量，这里将端口号写死了并不友好。

像这种跨Role引用的变量，也即多个Role共用的变量，可以将它们定义在inventory文件的all主机组中，也可也定义在playbook同目录层次的`group_vars`目录下的all.yml文件中。此处简单做个演示，在后面介绍变量的文章里会再做解释。

例如：
```shell
$ tree -L 2 -F lnmp
lnmp
├── group_vars/
│   └── all.yml
├── inventory_lnmp
├── lnmp.yml
└── roles/
    ├── common/
    ├── mysql/
    ├── nginx/
    └── php/

$ cat lnmp/group_vars/all.yml
phpfpm_port: 9000
```

于是上面nginx变量文件中就可以引用这个共用变量的值，例如：
```yaml
vhosts:
  server1:
    server_name: www.abc.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:{{phpfpm_port}}"
  server2: 
    server_name: www.def.com
    listen: 80
    fastcgi_pass: "{{groups.php[0]}}:{{phpfpm_port}}"
```


### 7.8.3 PHP Role

PHP Role比较简单，只需一个任务文件加一个变量文件即可。
```shell
$ rm -rf lnmp/roles/php/{defaults,files,templates,meta,handlers,tests,README.md}
```

`lnmp/roles/php/tasks/main.yml`文件内容：
```yaml
---
- name: install php and php-fpm
  yum: 
    name: "{{item}}"
    state: installed
  loop:
    - php
    - php-fpm

- name: change php-fpm listen address and port
  shell: |
    sed -ri 's/^(listen *= *)127.0.0.1.*$/\1{{phpfpm_addr}}:{{phpfpm_port}}/' /etc/php-fpm.d/www.conf
    sed -ri 's/^(listen.allowed_clients.*)$/;\1/' /etc/php-fpm.d/www.conf

- name: restart php-fpm
  service:
    name: php-fpm
    state: reloaded
```

`lnmp/roles/php/vars/main.yml`文件内容：
```yaml
phpfpm_addr: 0.0.0.0
phpfpm_port: 9000
# phpfpm_port: "{{phpfpm_port}}" # 这是错误的
```

注意，这里的`phpfpm_port`变量不可以引用共用变量的值，因为变量重名了。要想引用共用变量，要么这里的变量改名，要么加一个变量层次，要么使用inventory变量访问方式访问`group_vars`中的变量。所以下面三种方式都可以：
```yaml
phpfpm_addr: 0.0.0.0
phpfpm_port1: "{{phpfpm_port}}"

phpfpm_addr: 0.0.0.0
phpfpm_port: "{{hostvars[groups.php[0]].phpfpm_port}}"

phpfpm:
  phpfpm_addr: 0.0.0.0
  phpfpm_port: "{{phpfpm_port}}"
```

### 7.8.4 MySQL Role

MySQL的任务比较多，也比较零散，所以将它们分开单独定义。

把用不上的目录删掉：
```shell
$ rm -rf lnmp/roles/mysql/{defaults,tests,files,meta,README.md}
```

`lnmp/roles/mysql/tasks/main.yml`文件内容如下：
```yaml
---
- import_tasks: install_mysql.yml
- import_tasks: mysql_cnf.yml
- import_tasks: modify_mysql_temp_passwd.yml
  when: not temporary_passwd_updated
- import_tasks: ansible_mysql_connection_info.yml
- import_tasks: do_anything_in_mysql.yml
```

`lnmp/roles/mysql/tasks/install_mysql.yml`文件内容如下：

```yaml
---
- name: install mysql
  yum: 
    name: "{{item}}"
    state: installed
  loop: 
    - mysql-community-server
    - mysql-community-client
    - python2-PyMySQL
    - MySQL-python
```

`lnmp/roles/mysql/tasks/mysql_cnf.yml`文件内容如下：
```yaml
---
- name: render mysql config
  template:
    src: mysql.cnf.j2
    dest: /etc/my.cnf
  notify: restart mysql

- name: create mysql datadir
  file: 
    name: "{{mysqld.datadir}}"
    state: directory
    owner: mysql
    group: mysql

# 提供了MySQL配置文件后就要启动MySQL服务，以便后面的任务连接MySQL
# 可以编写启动服务的任务，也可以直接flush handlers来启动MySQL服务
- meta: flush_handlers
```

`lnmp/roles/mysql/tasks/modify_mysql_temp_passwd.yml`文件内容如下：

```yaml
---
- name: get initialize temp password
  shell: |
    sed -rn '/temp.*pass/s/^.*root@localhost: (.*)$/\1/p' {{mysqld.log_error}}
  register: tmp_passwd

- name: modify root@localhost password before any op
  shell: |
    mysql -uroot -p'{{tmp_passwd.stdout}}' \
    --connect-expired-password \
    -NBe \
    'ALTER USER "root"@"localhost" identified by "{{mysql_passwd}}";'

- name: update variable temporary_passwd_updated to TRUE
  lineinfile:
    path: "{{role_path}}/vars/main.yml"
    line: 'temporary_passwd_updated: true'
    regexp: '^temporary_passwd_updated:.*'
  delegate_to: localhost
```

修改临时密码的任务是只执行一次的任务，上面采用的方法是前文提到过的提供状态变量，然后判断它。

注意上面使用了一个Ansible的预定义特殊变量`role_path`，它表示的是当前Role的路径，这对于修改mysql Role的变量文件来说正合适。

`lnmp/roles/mysql/tasks/ansible_mysql_connection_info.yml`文件内容如下：
```yaml
---
- name: set mysql connection info for ansible
  template:
    src: .my.cnf.j2
    dest: /root/.my.cnf
```

`lnmp/roles/mysql/tasks/do_anything_in_mysql.yml`文件内容如下：

```yaml
---
# 移除localhost的匿名用户
- name: remove anonymous user for localhost
  mysql_user:
    name: ""
    host: localhost
    state: absent

# 移除所有匿名用户
- name: remove all anonymous user
  mysql_user:
    name: ""
    host_all: true
    state: absent

# 创建用户并指定权限
- name: create user and grant privileges
  mysql_user: 
    name: "junmajinlong"
    host: "192.168.200.%"
    password: 'P@ssword2!'
    priv: '*.*:ALL'
    state: present
    login_user: root
    login_password: 'P@ssword1!'

# 创建数据库
- name: Create new databases with names 'foo' and 'bar'
  mysql_db:
    name: test
    state: present

# 创建多个数据库
- name: Create new databases with names 'foo' and 'bar'
  mysql_db:
    name:
      - "foo"
      - "bar"
    state: present

# 删除test数据库
- name: drop test database
  mysql_db: 
    name: test
    state: absent

# 删除多个数据库
- name: drop databases with names 'foo' and 'bar'
  mysql_db:
    name:
      - "foo"
      - "bar"
    state: absent
```

`lnmp/roles/mysql/templates/mysql.cnf.j2`内容如下：
```yaml
[client]
socket = {{client.socket}}

[mysqldump]
max_allowed_packet = {{mysqldump.max_allowed_packet}}

[mysqld]
port = {{mysqld.port}}
datadir = {{mysqld.datadir}}
socket = {{mysqld.socket}}
server_id = {{mysqld.server_id}}
log-bin = {{mysqld.log_bin}}
sync_binlog = {{mysqld.sync_binlog}}
binlog_format = {{mysqld.binlog_format}}
character-set-server = {{mysqld.character_set_server}}
skip_name_resolve = {{mysqld.skip_name_resolve}}
pid-file = {{mysqld.pid_file}}
log-error = {{mysqld.log_error}}
```

`lnmp/roles/mysql/templates/.my.cnf.j2`内容如下：
```yaml
[client]
user = root
password = {{mysql_passwd}}
socket = {{mysqld.socket}}
```

`lnmp/roles/mysql/handlers/main.yml`内容如下：

```yaml
- name: restart mysql
  service:
    name: mysqld
    state: restarted
```

`lnmp/roles/mysql/vars/main.yml`内容如下：
```yaml
temporary_passwd_updated: false
mysql_passwd: "P@ssword1!"
client: 
  socket: "/data/mysql.sock"
mysqldump: 
  max_allowed_packet: "32M"
mysqld: 
  port: 3306
  datadir: "/data"
  socket: "/data/mysql.sock"
  server_id: 100
  log_bin: "mysql-bin"
  sync_binlog: 1
  binlog_format: "row"
  character_set_server: "utf8mb4"
  skip_name_resolve: 1
  pid_file: "/data/mysql.pid"
  log_error: "/data/error.log"
```

## 7.9 为每个Role提供一个playbook

有时候看别人写的Role，可能会发现为每个Role提供了一个playbook。

比如nginx Role有一个nginx.yml playbook，php Role有一个php.yml，最后还有一个操作所有Role的playbook(本文lnmp Role示例便是这种方式)。如下：
```shell
$ tree -L 2 -F lnmp 
lnmp
├── common.yml   # 单独操作common Role的playbook
├── group_vars/
│   └── all.yml
├── inventory_lnmp
├── lnmp.yml     # 汇总了所有Role的playbook
├── mysql.yml    # 单独操作MySQL Role的playbook
├── nginx.yml    # 单独操作nginx Role的playbook
├── php.yml      # 单独操作php   Role的playbook
└── roles/
    ├── common/
    ├── mysql/
    ├── nginx/
    └── php/
```

之所以要为每个Role都单独提供一个playbook，仅仅只是希望在某些情况下能单独操作那个Role，这样就不用每次都执行所有Role。比如写完common Role的时候，为了测试，会执行一下这个Role以保证这个Role中没有错误，但既然它已经执行过了，下次写完nginx Role就没必要再执行common Role。

另一方面，为每个Role提供单独的playbook，还能将这些Role解耦。

比如在这个LNMP的示例中，nginx Role、php Role和MySQL Role分别在不同的节点上执行，而且它们没有任何依赖和关联关系，如果将它们汇集在同一个playbook中执行，效率会非常低。比如Ansible控制nginx节点执行nginx Role任务的时候，php节点和MySQL节点都空闲着。特别是这几个Role中都会yum安装软件包，而刚初始化的系统或更新了repo文件时，安装软件包时都会先更新yum的metadata，这个过程体现在执行yum安装包任务时，速度会非常非常慢。

> 很多人(也包括我)都觉得yum模块比yum命令更慢，而且慢的多。所以如果可以的话，将你的playbook中所有的yum模块替换成shell模块执行yum命令吧。

因此，非常有必要优化这种低效。如果将所有Role的任务集中在同一个playbook中，将没有办法优化，Ansible只提供了任务级别的并行执行能力，没有提供play级别和Role级别的并行。但如果为每个Role提供一个playbook，就可以在Shell层次下实现并行，比如写一个简单的Shell脚本：

```shell
#!/bin/bash

# common不能放在后台，因为其它Role依赖common Role
ansible-playbook -i inventory_lnmp common.yml
ansible-playbook -i inventory_lnmp nginx.yml &
ansible-playbook -i inventory_lnmp php.yml &
ansible-playbook -i inventory_lnmp mysql.yml &
```

关于优化Ansible效率的话题，后面我会专门写一篇文章。但希望大家记住，这里提供的Shell层次的并行也是一种优化方式，而且很香，但很多人都忽略了这种优化方式。