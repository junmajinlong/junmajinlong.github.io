---
title: 熟练使用vagrant(10)：vagrant批量创建虚拟机
p: virtual/vagrant/vagrant_multi_vms.md
date: 2020-10-01 17:37:39
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(10)：vagrant批量创建虚拟机

官方手册已经给出了批量创建虚拟机的用法：[Multi-Machine](https://www.vagrantup.com/docs/multi-machine)。

`config.vm.define`可以定义虚拟机在vagrant中的name标识符。例如：
```
Vagrant.configure('2') do |config|
  config.vm.box = "generic/ubuntu2004"
  config.vm.define "ubuntu2004_junmajinlong"
end
```

如果为`config.vm.define`指定了语句块，则这个语句块相当于在config内部创建了一个独立的作用域更小的config环境。通过这种方式，可以在一个Vagrantfile中同时定义多个虚拟机。

例如，官方手册给的例子：

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision "shell", inline: "echo Hello"

  config.vm.define "WEB" do |web|
    web.vm.box = "apache"
  end

  config.vm.define "DB" do |db|
    db.vm.box = "mysql"
  end
end
```

在上面示例中，web和db两个语句块参数和config语句块参数具有相同的功能，不同的仅仅是它们的作用域不同，config在整个`Vagrant.configure do...end`语句块中生效，而web或db只在`config.vm.define do...end`语句块中生效。

因此，能定义在config下的属性，也都能定义在config.vm.define的语句块内。例如：
```ruby
Vagrant.configure("2") do |config|
  config.vm.provision :shell, inline: "echo A"

  config.vm.define :test1 do |test|
    test.vm.provision :shell, inline: "echo B"
  end
  config.vm.define :test2 do |test|
    test.vm.provision :shell, inline: "echo D"
  end

  config.vm.provision :shell, inline: "echo C"
end
```

需注意，不同作用域内定义重复的属性(例如上面的provision)，外层优先级高于内层，同层则按顺序。因此，对于上面的示例中，将先输出A，再输出C，最后输出B或D。

当在Vagrantfile中通过config.vm.define定义了多个虚拟机时，某些控制虚拟机的vagrant子命令的语义将发生变化，例如：
- `vagrant ssh`需要指定名称或ID表明要连接到哪个虚拟机  
- `vagrant up`默认将启动Vagrantfile中定义的所有虚拟机，如果想要只启动某个或某些虚拟机，需指定名称或ID或匹配名称的正则表达式  
- vagrant halt、suspend、reload等也都类似于up子命令  

例如，某Vagrantfile中定义了leader、follower0、follower1、follower2共四台虚拟机：
```powershell
# 连接到leader这台虚拟机
vagrant ssh leader

# 启动这四台虚拟机
vagrant up

# 关闭follower1这台虚拟机
vagrant halt follower1

# 重启所有follower虚拟机
#   当vagrant发现目标参数包含在双斜线//内，
#   将使用正则去匹配目标
vagrant reload /follower\d/
```

对于像vagrant ssh这样需要指定单个目标虚拟机的子命令，如果确实想要省略单个目标，可将某个虚拟机定义为默认虚拟机。最多只能定义一个默认虚拟机。

```ruby
config.vm.define "web", primary: true do |web|
  # ...
end
```

默认情况下，`vagrant up`会启动Vagrantfile中定义的所有虚拟机，但可以通过参数`autostart`来控制某虚拟机是否自动在`vagrant up`时自动启动，该参数默认值为true。

例如：
```
config.vm.define "web"
config.vm.define "db"
config.vm.define "db_follower", autostart: false
```
这表示执行`vagrant up`时，将启动web和db，不会启动db_follower，只有`vagrant up db_follower`时才会启动db_follower。

借助ruby的迭代功能，可以方便地定义多个虚拟机。例如使用each(注意不要使用for循环，因为for每次迭代时都引用同一个迭代变量，而each等方法式迭代语句块的迭代变量在每次迭代过程中都会重新创建)。
```ruby
(1..3).each do |i|
  config.vm.define "node-#{i}" do |node|
    node.vm.provision "shell",
      inline: "echo hello node #{i}"
  end
end
```

## 使用vagrant批量创建LNMP所需虚拟机

下面是我使用vagrant自动安装LNMP所需虚拟机的Vagrantfile文件。

该示例总共安装5个虚拟机，它们都是centos7系统：  
- 一个nginx节点  
- 一个php节点  
- 3个mysql节点，一个准备做主从同步的主，另外两个准备做从节点  

同时还做了一点点基本的配置：  
- 所有节点都配置了国内的yum源和epel源  
- nginx节点安装nginx  
- php节点配置remi和remisafe源，并安装php和php-fpm  
- 三个mysql节点都配置了mysql源，并安装了mysql和mysql-server  

```ruby
Vagrant.configure("2") do |config|
  ############## yum repos variable ##############
  base_repo = <<~BASE
    [base]
    name=CentOS-$releasever - Base
    baseurl=https://mirrors.ustc.edu.cn/centos/$releasever/os/$basearch/
    gpgcheck=0

    [updates]
    name=CentOS-$releasever - Updates
    baseurl=https://mirrors.ustc.edu.cn/centos/$releasever/updates/$basearch/
    gpgcheck=0

    [extras]
    name=CentOS-$releasever - Extras
    baseurl=https://mirrors.ustc.edu.cn/centos/$releasever/extras/$basearch/
    gpgcheck=0
  BASE
  
  epel_repo =<<~EPEL
    [epel]
    name=Extra Packages for Enterprise Linux 7 - $basearch
    baseurl=https://mirrors.ustc.edu.cn/epel/7/$basearch
    enabled=1
    gpgcheck=0
  EPEL
  
  php_repo = <<~PHP
    [remi]
    name=remirepo
    baseurl=http://mirrors.ustc.edu.cn/remi/enterprise/7/php74/x86_64/
    enable=1
    gpgcheck=0

    [remisafe]
    name=remisaferepo
    baseurl=http://mirrors.ustc.edu.cn/remi/enterprise/7/safe/x86_64/
    enable=1
    gpgcheck=0
  PHP
  
  mysql_repo = <<~MYSQL
    [mysql57]
    name=MySQL
    baseurl=https://mirrors.cloud.tencent.com/mysql/yum/mysql57-community-el7/
    enabled=1
    gpgcheck=0
  MYSQL
  
  ########## base repo and epel repo for all vm ###########
  config.vm.provision "shell", inline: <<-SHELL
    sudo bash
    mkdir /etc/yum.repos.d/bak
    mv /etc/yum.repos.d/CentOS-*.repo /etc/yum.repos.d/bak
    # yum base repo
    cat <<'EOF' >/etc/yum.repos.d/base.repo
#{base_repo}
EOF
    # yum epel repo
    cat <<'EOF' >/etc/yum.repos.d/epel.repo
#{epel_repo} 
EOF
  SHELL
  
  # all vm: centos7
  config.vm.box = "generic/centos7"
  
  ######### nginx node ##########
  config.vm.define "nginx" do |ngx|
    ngx.vm.hostname = "nginx-vm"
    ngx.vm.provider :virtualbox do |vb|
      vb.name = "nginx_centos7"
    end
    
    ngx.vm.provider :hyperv do |v|
      v.vmname = "nginx_centos7"
    end
    
    ngx.vm.provision "shell", inline: "sudo yum install -y nginx"
  end

  ######### php node ##########
  config.vm.define "php" do |php|
    php.vm.hostname = "php-vm"
    php.vm.provider :virtualbox do |vb|
      vb.name = "php_centos7"
    end
    
    php.vm.provider :hyperv do |v|
      v.vmname = "php_centos7"
    end
    
    php.vm.provision "shell", inline: <<-SHELL
      sudo bash
      cat <<'END' >/etc/yum.repos.d/remi.repo
#{php_repo}
END
      yum install -y php php-fpm
    SHELL
  end
  
  ########### mysql master ###########
  config.vm.define "mysql-master" do |mysql|
    mysql.vm.hostname = "mysql-master-vm"
    mysql.vm.provider "virtualbox" do |vb|
      vb.name = "mysql-master_centos7"
    end

    mysql.vm.provider :hyperv do |v|
      v.vmname = "mysql-master_centos7"
    end
      
    mysql.vm.provision "shell", inline: <<-SHELL
      sudo bash
      cat <<'END' >/etc/yum.repos.d/mysql.repo
#{mysql_repo}
END
      yum install -y mysql-community-{server,client}
    SHELL
  end

  ########## mysql slaves ############
  2.times do |i|
    config.vm.define "mysql-slave-#{i}" do |mysql|
      mysql.vm.hostname = "mysql-slave-#{i}-vm"

      mysql.vm.provider :virtualbox do |vb|
        vb.name = "mysql-slave-#{i}_centos7"
      end

      mysql.vm.provider :hyperv do |v|
        v.vmname = "mysql-slave-#{i}_centos7"
      end
      
      mysql.vm.provision "shell", inline: <<-SHELL
        sudo bash
        cat <<'END' >/etc/yum.repos.d/mysql.repo
#{mysql_repo}
END
        yum install -y mysql-community-{server,client}
      SHELL
    end
  end
end
```

然后启动即可：
```powershell
# virtualbox
$ vagrant up

# hyper-v
$ vagrant up --provider hyperv
```

如果想要并行启动(virtualbox不支持并行启动)：
```powershell
$ vagrant up --parallel
```

如果发现mysql节点错误想要重新启动mysql并让它们执行provision：
```powershell
$ vagrant up /mysql.*/ --provision
```

删除所有虚拟机：
```powershell
$ vagrant destroy -f
```
