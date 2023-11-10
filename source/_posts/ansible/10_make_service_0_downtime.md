---
title: 10.服务0 downtime的追求：Haproxy+Nginx集群服务的滚动发布和节点伸缩
p: ansible/10_make_service_0_downtime.md
date: 2021-07-11 10:37:39
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。

# 10.服务0 downtime的追求：Haproxy+Nginx集群服务的滚动发布和节点伸缩

本文介绍如何通过Ansible配置部署HAProxy反代Nginx，本文不考虑这个集群是否合理，只考虑其中涉及的集群维护的经典逻辑：如何维护集群中后端节点的升降级、如何维护后端的节点伸缩。

说明：本文使用<https://github.com/malongshuai/ansible-column.git>中的"10th/lbcluster"目录。
```shell
$ git clone https://github.com/malongshuai/ansible-column.git
```

## 10.1 环境说明

如下：
```yaml
HAProxy节点:
  IP地址: 192.168.200.48
  操作系统: CentOS 7.2
Nginx节点: 
  - IP地址: 192.168.200.49
    操作系统: CentOS 7.2
  - IP地址: 192.168.200.50
    操作系统: CentOS 7.2
  - IP地址: 192.168.200.51
    操作系统: CentOS 7.2
```

在后文还会介绍后端Nginx节点扩充问题，相关环境到时候再另行说明。

Ansible Inventory文件hosts内容如下：
```ini
[lb]
192.168.200.48

[nginxs]
192.168.200.49
192.168.200.50
192.168.200.51
```

接着，为安装配置nginx和安装配置haproxy分别创建Role，它们都在`lbcluster/roles`目录下：
```shell
$ mkdir -p lbcluster/roles 
$ cd lbcluster/roles
$ ansible-galaxy init nginx
$ ansible-galaxy init haproxy
```

为这两个Role提供入口playbook文件以及全局变量`group_vars/all.yml`文件：
```shell
$ cd lbcluster
$ mkdir group_vars
$ touch group_vars/all.yml
$ touch lb.yml      # 集群的入口playbook
```

在`group_vars/all.yml`中定义全局变量：
```yaml
haproxy_version: 1.8
haproxy_config: /etc/haproxy/haproxy.cfg
nginx_port: 80
```

入口playbook文件lb.yml的内容：
```yaml
---
- name: intstall and config haproxy
  hosts: lb
  gather_facts: no
  roles:
   - haproxy

- name: install and config nginx
  hosts: nginx
  gather_facts: no
  roles: 
    - nginx
```

然后介绍HAProxy Role和Nginx Role相关文件的内容。

注：本文重点在于集群维护而不是jinja2模板，所以本文的配置文件模板都只使用了几个最基本的Jinja2语法。

## 10.2 安装配置HAProxy Role

HAProxy Role用到的文件包括：
```shell
$ tree roles/haproxy/
roles/haproxy/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── haproxy.cfg.j2
└── vars
    └── main.yml
```

所以删除用不到的文件：
```shell
$ rm -rf roles/haproxy/{defaults,files,meta,README.md,tests}
```

vars/main.yml文件内容如下：
```yaml
pidfile: /var/run/haproxy.pid
socket: /var/lib/haproxy/stats
haproxy_port: 80
```

tasks/main.yml文件内容如下：
```yaml
---
# tasks file for haproxy
- name: config ius repo
  yum_repository:
    name: iusrepo
    description: ius repo
    baseurl: https://mirrors.tuna.tsinghua.edu.cn/ius/$releasever/$basearch/
    gpgcheck: no

# 本文使用haproxy 1.8，后面会使用到1.8版本才提供的完美reload功能
- name: install haproxy
  shell: yum install -y haproxy18u

- name: template haproxy config
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
    validate: haproxy -f %s -c
  notify: "reload haproxy"
```

handlers/main.yml内容如下：
```yaml
---
# handlers file for haproxy
- name: reload haproxy
  shell: |
    haproxy -f {{haproxy_config}} -c -q && bash -c '
    if killall -0 haproxy &>/dev/null;then
      # 进程已存在，则reload(haproxy 1.8则socket reload，需指定-x SOCKET选项)
      haproxy -f {{haproxy_config}} -q -p {{pidfile}} -sf $(cat {{pidfile}}) {{ "-x "~socket if haproxy_version is defined and haproxy_version is version("1.8",">=") else "" }}
    else
      # 进程不存在，则启动
      haproxy -f {{haproxy_config}} -q -p {{pidfile}}
    fi'
```

上面haproxy reload的handler我没有使用service模块，因为service模块的reload是普通的reload，它可能会丢失极少量的会话，而这里我要通过haproxy 1.8版的`-x SOCKET`选项来执行socket reload，从而保证haproxy的reload操作连接无损。

templates/haproxy.cfg.j2内容如下：
```jinja2
global
  log         127.0.0.1 local2
  chroot      /var/lib/haproxy
  pidfile     {{pidfile|default("/var/run/haproxy.pid")}}
  maxconn     20000
  user        haproxy
  group       haproxy
  daemon
  {# haproxy 1.8完美reload需expose-fd listeners #}
  stats socket {{socket|default("/var/lib/haproxy/stats")}} level admin {{ "expose-fd listeners" if haproxy_version is defined and haproxy_version is version("1.8",">=") else "" }}
  spread-checks 2
defaults
  mode                    http
  log                     global
  option                  httplog
  option                  dontlognull
  option http-server-close
  option forwardfor       except 127.0.0.0/8
  option                  redispatch
  timeout http-request    2s
  timeout queue           3s
  timeout connect         1s
  timeout client          10s
  timeout server          2s
  timeout http-keep-alive 10s
  timeout check           2s
  maxconn                 18000

frontend main
  bind             *:{{haproxy_port|default(80)}}
  mode             http
  log              global
  capture request  header Host len 20
  capture request  header Referer len 60
  acl url_static   path_beg -i /static /images /stylesheets
  acl url_static   path_end -i .jpg .jpeg .gif .png .ico .bmp .css .js
  acl url_static   path_end -i .html .htm .shtml .pdf .mp3 .mp4 .rm .rmvb .txt
  acl url_static   path_end -i .zip .rar .gz .tgz .bz2 .tgz

  use_backend      static_group   if url_static

backend static_group
  balance            roundrobin
  option             http-keep-alive
  http-reuse         safe
  option httpchk     GET /index.html
  http-check expect  status 200
{% for ngx in groups['nginx'] %}
  server nginx{{loop.index}} {{ngx}}:{{nginx_port|default(80)}} check rise 1 maxconn 5000
{% endfor %}
```

## 10.3 安装配置Nginx Role

Nginx Role用到的文件：
```shell
$ tree roles/nginx/
roles/nginx/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   ├── index.html.j2
│   ├── nginx.conf.j2
│   └── nginx.vhost.conf.j2
└── vars
    └── main.yml
```

删除不需要的文件：
```
$ rm -rf roles/nginx/{defaults,files,meta,README.md,tests}
```

vars/main.yml内容如下：
```yaml
---
# vars file for nginx
worker_processes: 1
worker_rlimit_nofile: 65535
worker_connections: 10240
multi_accept: "on"
send_file: "on"
tcp_nopush: "on"
tcp_nodelay: "on"
keepalive_timeout: 65
server_tokens: "off"
ngx_port: "{{nginx_port}}"
```

tasks/main.yml内容如下：
```yaml
---
# tasks file for nginx

- name: config ius repo
  yum_repository:
    name: nginxrepo
    description: nginx repo
    baseurl: http://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck: no

- name: install nginx
  shell: yum install -y nginx

- name: template nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: "reload nginx"

- name: template nginx vhost config
  template: 
    src: nginx.vhost.conf.j2
    dest: /etc/nginx/conf.d/default.conf
  notify: "reload nginx"

- name: template index.html for test
  template: 
    src: index.html.j2
    dest: /usr/share/nginx/html/index.html
```

handlers/main.yml内容如下：
```yaml
---
# handlers file for nginx

- name: check nginx syntax
  shell: |
    nginx -t -c /etc/nginx/nginx.conf
  listen: reload nginx
  register: syntax

- name: reload nginx
  service:
    name: nginx
    state: reloaded
  listen: reload nginx
  when: syntax.rc == 0
```

templates/nginx.conf.j2内容如下：
```jinja2
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

  include /etc/nginx/conf.d/*.conf;
}
```

templates/nginx.vhost.conf.j2内容如下：
```jinja2
server {
    listen       {{ngx_port}};
    server_name  {{inventory_hostname}};

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

templates/index.html.j2内容如下：
```jinja2
<h1>page from: {{inventory_hostname}}</h1>
```

写完haproxy和nginx的Role后，启动haproxy和nginx：
```shell
$ cd lbcluster
$ ansible-playbook -i hosts lb.yml
```

## 10.4 滚动发布：Load Balance后端节点升降级

假设需要维护haproxy后端的服务节点(本文即nginx)，比如服务版本升降级，修改服务配置等，该如何操作才能完美维护整个集群？

维护的大概步骤如下：  
1. 在haproxy中禁用一个或多个后端待维护节点，但不能全部禁用，否则整个服务会直接对外下线  
2. 维护后端节点  
3. 重启后端节点服务  
4. 在haproxy中检测重启后的后端节点服务，服务可用后重新启用这些维护完成的后端节点  
5. 按照上面同样的逻辑，维护剩下的后端节点，直到所有节点维护完成  

上面的逻辑就是"滚动发布"的逻辑，所谓滚动发布，一般是取一个或者多个节点停止服务，执行更新，并重新将其投入使用，第一批维护成功后，逐渐扩散维护剩下的节点，直到所有节点都更新成新版本。

滚动发布要求集群的前端有一个无连接损失的负载均衡或反向代理服务，本文的haproxy便是一个相当完美的反代软件，它可以在线禁用某个后端待维护节点以及在线启用维护完成的节点，并且haproxy自身的重启也可以做到完全不丢失任何连接(要求HAProxy 1.8)。

> 注：HAProxy重启如何不丢失连接？
> 在大并发流量下，HAProxy直接reload操作会导致极少量连接丢失，大多数场景下这是可以接受的，但是在极为严格的场景下，这是不允许的，极严格的场景下要求不因重启而丢失任何连接或导致任何错误，HAProxy 1.8版本提供了连接转移到socket的reload方式，这可以保证不丢失任何连接。在HAProxy 1.8之前，也能做到不丢失任何连接的reload，但操作比较复杂，有兴趣的可Google。

这里不多展开滚动发布相关的内容，本文的重点是如何用Ansible来自动化上面滚动发布的维护逻辑？这里需要用到Ansible提供的`serial`指令，它会改变Ansible的执行策略，此外还需要介绍一种free策略，这种策略在批量服务重启时也会用到。既然如此，那就完整一点，将Ansible的执行策略系统性地介绍介绍。

## 10.5 理解Ansible执行策略

这里将涉及几个关键字：forks、serial、strategy，这些关键字指定的策略，其效果可能会相互交叉，所以还请各位仔细体会下面的解释。

此外，任务的异步执行也会改变执行策略，相关内容在后面关于Ansible效率优化的章节再介绍。

### 10.5.1 Ansible的默认执行策略

之前章节中曾介绍过Ansible的默认执行策略。Ansible默认的执行策略如下流程，其中T1、T2和T3是当前Play中的三个任务，n1、n2、n3、n4、n5、n6、n7是该play选中的待执行任务的节点。

```
T1(n1,n2,n3,n4,n5) --> T1(n6,n7) --> 
T2(n1,n2,n3,n4,n5) --> T2(n6,n7) --> 
T3(n1,n2,n3,n4,n5) --> T3(n6,n7)
```

如图：

![](/img/ansible/1576213846.gif)

默认执行策略总结起来，即：  
1. 根据`forks`指定的值N，最多有N个节点同时执行任务  
2. 所有节点执行完第一个任务，才进入第二个任务，所以先执行完任务的节点会空闲着等待剩余的节点  

这里的`forks`控制了最多有多少个Ansible进程(实际上是python进程)，例如默认forks=5，即表示最多有5个Ansible进程同时执行任务，更形象一点，即最多有5个节点同时执行任务。

需要说明，forks=5的情况下还要再加上一个Ansible主控制进程，所以总共最多有6个Ansible进程。Ansible主控进程会监控节点执行任务的状态，并决定是否要创建子Ansible进程来执行任务。

例如下面是默认forks=5时的进程显示：
```shell
$ ps j | grep pytho[n]
PPID   PID    PGID   SID   TTY    TPGID  STAT UID  TIME COMMAND
16067  16540  16540  16067 pts/0  16540  Rl+  0    0:01 /usr/bin/python3
16540  16546  16540  16067 pts/0  16540  R+   0    0:00 /usr/bin/python3
16540  16547  16540  16067 pts/0  16540  R+   0    0:00 /usr/bin/python3
16540  16549  16540  16067 pts/0  16540  S+   0    0:00 /usr/bin/python3
16540  16552  16540  16067 pts/0  16540  S+   0    0:00 /usr/bin/python3
16540  16555  16540  16067 pts/0  16540  S+   0    0:00 /usr/bin/python3
```

所以上面的流程图并不严谨，因为有的节点执行任务可能较慢。比如forks=5时，第一批5个节点中第2个节点先执行完该任务，Ansible主控进程会立即创建一个新Ansible进程让第6个节点执行任务。

可以在配置文件中或命令行的-f选项指定forks的值，它们都会对所有play生效。

```shell
$grep 'forks' /etc/ansible/ansible.cfg 
#forks          = 5

$ ansible-playbook -f 5 ...
```

### 10.5.2 serial将节点分批执行play

![](/img/ansible/1625971469088.png)

另一方面，指定`serial`并不会改变每个节点执行任务的方式：同一serial批中的某节点先执行完某任务时，必须等待本批中其它节点执行完该任务才切换到下一个任务。要改变每个节点执行任务的方式，需要使用`stragegy`指令，稍后会介绍。

所以，serial指令的效果是改变执行play的策略：不再让所有节点执行完某任务才进入下一个任务(默认)，而是让一批节点执行完play中所有任务才切换到下一批节点。

serial指定几个节点作为一个批次的方式有多种：  
1. 指定为单个数值N或百分数N%：每N个节点或该play中N%的节点作为一个批次去执行play中的所有任务，如`serial: 3`或`serial: 50%`  
2. 指定为数值或百分数列表：如`[1,3,5]`，表示第一批只让一个节点执行该play所有任务，然后挑3个节点作为一批，再挑5个节点作为一批，之后每次都挑5个节点作为一批。列表中数值和百分数可共存，例如`[1,3,50%]`  

显然，没有指定serial时，默认行为等价于`serial: 100%`，即将所有节点作为一个批次。

例如：

```yaml
- hosts: all
  gather_facts: no
  serial: 3
  
#
  serial: 30%
  serial: 
    - 1
    - 3
    - 6
  serial: 
    - 20%
    - 50%
  serial: 
    - 1
    - 2
    - 50%
```

将serial设置为列表，且节点数量越来越多是比较符合更新逻辑的。即，先取少量节点做小白鼠，成功之后逐渐扩大更新的节点数量，加快整个集群的更新。

例如两个play，第一个play设置`serial=[1,2,6]`，第二个play默认策略，整个执行过程如下图所示：

![](/img/ansible/ansible_serial.gif)


### 10.5.3 strategy指定执行策略

`strategy`指令可以设置play中每个节点执行任务的方式，即各节点的任务执行策略。

strategy有两种值：  
- linear：默认策略，即每个节点必须等待所有节点(如果执行了serial，则是同批所有节点)执行完某任务才进入下一个任务  
- free：如果某节点先执行完任务，不再等待其它节点全部执行完当前任务，而是直接切换到下一个任务，直到该节点执行完当前play的所有任务  

例如：
```yaml
- hosts: all
  strategy: free
  tasks:
```

也可以在配置文件中设置对所有play生效的策略：
```shell
$ grep 'strategy' /etc/ansible/ansible.cfg
# by default, ansible will use the 'linear' strategy but you may want to try
#strategy = free
```

分析一种场景来理解这些策略，假如13个节点n1到n13，play中2个任务t1和t2，当设置`strategy=free,forks=5,serial=[1,6,6]`时：首先是第一个serial批的节点n1执行完所有任务t1和t2后，切换到下一个serial批的6个节点(n2-n7)，但forks=5，所有最多只有5个节点同时执行任务，因为strategy=free，所以多出的那个节点必须等待某节点完成所有任务t1和t2才开始执行第一个任务t1。直到6个节点都执行完t1和t2，切换到第三批节点，第三批执行逻辑和第二批的逻辑是一样的。

什么时候会用到free策略？一种常见的场景是大批量节点的服务停止、再维护、再启动，如果使用默认的linear策略，则每一批中先执行完停止任务的节点必须等待其它节点停止完成才能开始执行维护任务，而这可能会导致服务长时间不可用，而且每批的节点越多，影响越大。而使用free策略，每个节点执行停止、维护、启动任务时都是马不停蹄的，这可以尽快完成服务的重启使其可用。

最后要说明的是，用户可以编写strategy插件来自定义执行策略。比如定义一种策略，让每个节点执行完3个任务后等待同批其它节点，只有同批其它节点都执行完3个任务后才进入另外3个任务。事实上，Ansible官方目前提供了4种策略插件：free、linear、host_pinned、debug。

### 10.5.4 throttle限制最大并发任务数

forks可限制最多几个节点同时执行任务，但它只能在配置文件中设置，所以forks的设置是对所有play所有task生效的。

serial可将节点分批执行play，所以也能限制最多几个节点同时执行任务，但是它是对整个play的所有任务而言的。

Ansible还提供了在play、role、block、task级别都可设置的`throttle`指令，它也可以限制最多几个节点同时执行任务。throttle的值需小于等于forks或serial的值，如果超出了forks或serial的值，则超出的并发数无效。

例如：
```yaml
tasks:
  - service:
      name: nginx
      state: reloaded
    throttle: 3
```

## 10.6 Ansible完成滚动发布

以前文的HAProxy+Nginx示例来演示，Ansible如何完成滚动发布。

假设nginx节点就是集群中需要升级的节点，需要改配置文件(例如加一个location、修改端口等等)，然后重启。在此期间，需要先在haproxy中禁用某节点的流量，nginx节点维护完成后，haproxy再重新启用该节点。

对于haproxy来说，无论是禁用某后端节点，还是再次启用该后端节点，都要保证不丢失连接(会话)。好在，haproxy支持在线禁用、在线启用，甚至haproxy自身的重启也不会丢失连接。

关于haproxy如何在线禁用、启用，可以在stat页面点鼠标设置，也可以通过socat命令或nc命令在命令行中设置，如何设置不是本文重点，有兴趣可以网上搜索相关操作。这里我将使用Ansible提供的haproxy模块，它也可以在线设置haproxy中各节点的状态。

那么，现在来配置这个滚动发布的过程吧。

首先，按照滚动发布的逻辑，所有任务应在nginx节点上执行，如果要操作haproxy节点，可使用委托的方式委托给haproxy。也就是说，要完成滚动发布，只需使用nginx Role来写相关任务，不需要在haproxy Role上写任务。

另一方面，根据滚动发布的策略：先挑一批小白鼠(一个或多个节点)来测试所有操作，只有小白鼠最终存活，才挑其它小白鼠继续操作。显然，Ansible完成滚动发布应使用serial来分批，而serial是play级别的指令，所以滚动发布相关的任务要单独定义成play。

所以可以重新写一个nginx滚动发布的Role，和之前安装配置nginx的Role区分开来，这样的好处是两个nginx Role独立了，出现问题时回滚更方便，但缺点是两个Nginx的Role不共享。

另一种方式是直接在之前安装配置nginx的Role中定义滚动发布的任务，这样的好处是两个nginx Role可以共享相关变量，方便以后的维护，但缺点是不方便出问题时回滚，而且要在同一个Role中区分配置安装的任务和滚动发布的任务，有点繁琐。

所以个人建议，为滚动发布的后端服务单独定义一套Role，对于需要共享的那些变量，抽取到全局变量文件`group_vars/all.yml`中。

例如，创建滚动发布的Role：
```shell
$ cd lbcluster/roles
$ ansible-galaxy init nginx_rolling_update
```

然后重新定义roles的入口playbook文件lb.yml：
```
- hosts: lb
  roles: 
    - haproxy
  tags:
    - haproxy
    - install

- hosts: nginx
  roles: 
    - nginx
  tags: 
    - nginx
    - install

- hosts: nginx
  serial: [1,3,50%]
  roles: 
    - nginx_rolling_update
  tags:
    - nginx
    - nginx_update
```

在之前的文章中曾说过，为每个Role都单独定义一个入口playbook文件，也是很好的方式。这里，我决定不去修改之前的入口playbook文件lb.yml，而是为滚动升级的Role单独定义一个入口playbook文件nginx\_rolling\_update.yml，内容如下：
```yaml
- hosts: nginx
  gather_facts: no
  roles: 
    - nginx_rolling_update
  serial: [1,3,50%]
```

此外，haproxy模块需要用到haproxy socket文件的路径，所以修改一下`group_vars/all.yml`加上变量`haproxy_socket`：
```yaml
# haproxy
haproxy_version: 1.8
haproxy_config: /etc/haproxy/haproxy.cfg
haproxy_socket: /var/lib/haproxy/stats

# nginx
nginx_port: 80
```

修改了all.yml，那么相应的要修改haproxy Role中关于socket路径的变量haproxy/vars/main.yml：
```yaml
---
# vars file for haproxy

pidfile: /var/run/haproxy.pid
socket: "{{haproxy_socket}}"
haproxy_port: 80
```

接下来是nginx\_rolling\_update Role用到的文件：
```shell
$ tree roles/nginx_rolling_update/
roles/nginx_rolling_update/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── nginx.vhost.conf.j2
└── vars
    └── main.yml
```

删除不用的文件：
```shell
$ rm -rf roles/nginx_rolling_update/{defaults,files,meta,README.md,tests}
```

下面是vars/main.yml文件中设置的变量：
```yaml
---
# vars file for roles/nginx_rolling_update
ngx_port: "{{nginx_port}}"
socket: "{{haproxy_socket}}"
```

下面tasks/main.yml文件内容：
```yaml
---
# tasks file for roles/nginx_rolling_update

- name: render nginx config
  template: 
    src: nginx.vhost.conf.j2
    dest: /etc/nginx/conf.d/default.conf
  notify: "reload nginx"
  register: render

# if template changed=0, skip remaining tasks for this node
- meta: end_host
  when: not render.changed

- name: disable current nginx node in haproxy
  haproxy:
    backend: static_group
    host: "{{inventory_hostname}}"
    state: disabled
    socket: "{{socket}}"
  delegate_to: "{{groups['lb'][0]}}"

- meta: flush_handlers

- name: ensure nginx is healthy running
  wait_for: 
    port: "{{ngx_port}}"

- name: enable current nginx node in haproxy
  haproxy: 
    backend: static_group
    host: "{{inventory_hostname}}"
    state: enabled
    socket: "{{socket}}" 
  delegate_to: "{{groups['lb'][0]}}"
```

在这个任务文件中：  
- 第一个任务是先渲染出新的nginx配置文件，在haproxy禁用节点之前生成配置文件，可稍微减少服务的下线时间  
- 如果渲染的配置文件没有发生任何修改，则通过`meta: end_host`让当前节点直接终止该play，而不要继续执行后续任务，一方面可以提升效率，另一方面可避免执行后续任务(即haproxy禁用再启用该节点)导致不必要的服务下线  
- 第三个任务是通过委托的方式在haproxy中禁用当前nginx节点，disable后nginx节点将不再接受新请求，但是会处理之前已建立好的连接，直到所有请求处理完成，该节点的状态被设置为maintenance  
- 第四个任务是手动`flush_handlers`来重启nginx，因为后续haproxy重新启用该节点的任务要求nginx服务已经重启完成  
- 第五个任务是等待nginx的80端口已打开，这表示nginx服务已正常运行。如果不等待端口就直接执行下一个任务，有可能会让haproxy将请求分发给还未正常运行的nginx节点  
- 最后一个任务是通过委托的方式在haproxy中将重启完成的nginx节点重新启用  

下面是handlers/main.yml文件的内容：
```yaml
---
# handlers file for roles/nginx_rolling_update
- name: check nginx syntax
  shell: |
    nginx -t -c /etc/nginx/nginx.conf
  listen: reload nginx
  register: syntax

- name: reload nginx
  service:
    name: nginx
    state: reloaded
  listen: reload nginx
  when: syntax.rc == 0
```

下面是templates/nginx.vhost.conf.j2文件的内容(相比之前的nginx虚拟主机配置文件，随便做点修改即可)：
```jinja2
server {
    listen       {{ngx_port}};
    server_name  {{inventory_hostname}};

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location /abc {
        root   /usr/share/nginx/html/abc;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

最后执行滚动发布：
```shell
$ ansible-playbook -i hosts nginx_rolling_update.yml
PLAY [nginx] *****************

TASK [render nginx config] ******************
changed: [192.168.200.49]

TASK [disable current nginx node in haproxy] ***
ok: [192.168.200.49 -> 192.168.200.48]

RUNNING HANDLER [check nginx syntax] ********
changed: [192.168.200.49]

RUNNING HANDLER [reload nginx] **************
changed: [192.168.200.49]

TASK [ensure nginx is healthy running] ********
ok: [192.168.200.49]

TASK [enable current nginx node in haproxy] ***
ok: [192.168.200.49 -> 192.168.200.48]

PLAY [nginx] *****************

TASK [render nginx config] ******
changed: [192.168.200.50]
changed: [192.168.200.51]

TASK [disable current nginx node in haproxy] ****
ok: [192.168.200.50 -> 192.168.200.48]
ok: [192.168.200.51 -> 192.168.200.48]

RUNNING HANDLER [check nginx syntax] ******
changed: [192.168.200.51]
changed: [192.168.200.50]

RUNNING HANDLER [reload nginx] ************
changed: [192.168.200.50]
changed: [192.168.200.51]

TASK [ensure nginx is healthy running] ****
ok: [192.168.200.50]
ok: [192.168.200.51]

TASK [enable current nginx node in haproxy] ******
ok: [192.168.200.50 -> 192.168.200.48]
ok: [192.168.200.51 -> 192.168.200.48]

PLAY RECAP **********************
192.168.200.49  : ok=6  changed=3  ...
192.168.200.50  : ok=6  changed=3  ...
192.168.200.51  : ok=6  changed=3  ...
```

注意观察上面的任务执行顺序，它们确实符合serial指令指定的执行策略。

如果再次执行，因为template渲染的文件没有发生任何改变，所以后续的任务都不再执行(因为`meta: end_host`终止了它们)：
```shell
$ ansible-playbook -i hosts nginx_rolling_update.yml
PLAY [nginx] ************

TASK [render nginx config] ****
ok: [192.168.200.49]

PLAY [nginx] 

TASK [render nginx config] ***
ok: [192.168.200.50]
ok: [192.168.200.51]

PLAY RECAP ********************
192.168.200.49  : ok=1  changed=0......
192.168.200.50  : ok=1  changed=0......
192.168.200.51  : ok=1  changed=0......
```

## 10.7 集群后端节点的伸缩

云计算火热之后，向一个已存在的集群中加入新节点以及从一个集群中踢掉某些节点，都是非常常见的操作。如何通过Ansible来完成后端节点的伸缩？这对Ansible来说，实在太easy了。

假设要向haproxy反代的nginx集群中加入三个新节点，扩充完成后再踢掉这三个节点。

对于云主机或Openstack等生成的新主机节点，可以使用动态inventory来设置这些新节点，甚至可以直接在Ansible中通过模块设置它们的IP地址，并将它们加入主机组。但这里假设已经获取到了这三个节点的IP地址，并且已经在Ansible控制端配置好了和这三个节点的ssh互信，所以在静态inventory文件中加入它们：
```ini
[lb]
192.168.200.48

[nginx]
192.168.200.49
192.168.200.50
192.168.200.51

# 加入下面两段
[nginx20200206]
192.168.200.31
192.168.200.32
192.168.200.33

[nginx:children]
nginx20200206
```

现在只需重新执行一下haproxy Role和nginx Role任务即可。
```shell
$ ansible-playbook -i hosts lb.yml
```

这便是扩充后端节点需要做的所有事情。

执行上述操作后，三个新节点会自动安装配置好nginx并启动，对于haproxy节点，因为haproxy.cfg.j2模板文件中遍历了nginx主机组来创建后端节点，所以直接reload haproxy即可，这会自动将3个新节点加入haproxy后端。

那么想要踢掉后端节点呢？一样很简单，直接在静态inventory文件中删掉对应的信息即可：
```ini
[lb]
192.168.200.48

[nginx]
192.168.200.49
192.168.200.50
192.168.200.51

[nginx20200206]
192.168.200.31
192.168.200.32
192.168.200.33

# 删除子组即可
[nginx:children]
#nginx20200206
```

然后重新执行haproxy Role和nginx Role即可：
```shell
$ ansible-playbook -i hosts lb.yml
```

为什么后端节点的伸缩操作这么简便？得益于两点：  
1. haproxy配置文件中通过遍历inventory中所有的后端节点的方式生成配置文件  
2. haproxy的reload操作可以"平滑"重启，不会丢失连接  

也就是说，因为有了haproxy这个负载均衡，才让Ansible管理整个集群的伸缩变得异常简单。

## 10.8 处理Ansible部署、维护过程中的异常

前文已经介绍完了Ansible安装、配置集群服务、滚动升级集群服务、集群后端节点的伸缩相关内容，但还没有考虑过部署过程中出现异常该如何处理。但维护集群过程中出现异常是非常常见的，这些异常也必须得做好处理，否则有可能会将错误的节点上线到集群中，从而影响整个集群。

在之前的章节中(第八章)曾系统性地介绍过如何处理异常问题，所以这里不再赘述，建议回头复习一遍。