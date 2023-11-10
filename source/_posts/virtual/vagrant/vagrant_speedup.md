---
title: 熟练使用vagrant(17)：解决vagrant下载很慢的问题
p: virtual/vagrant/vagrant_speedup.md
date: 2020-10-01 17:37:46
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------

# 熟练使用vagrant(17)：解决vagrant下载很慢的问题

vagrant有两方面操作涉及到下载：下载vagrant插件、下载box镜像。

## 设置代理解决vagrant下载慢的问题

这两方面的慢速问题都可以通过设置代理来解决。

在Linux环境下：
```bash
export http_proxy="http://yourproxy:8080"
export https_proxy="https://yourproxy:8080"
```

在cmd下：
```cmd
set http_proxy="http://yourproxy:8080"
set https_proxy="https://yourproxy:8080"
```

在powershell下，设置代理的方式比较麻烦，建议直接切换到cmd下去执行涉及下载类的操作。

然后就可以快速下载box镜像或快速安装vagrant插件：
```
# 安装插件
vagrant plugin install vagrant-vbguest

# 下载镜像
vagrant box add generic/centos7 --provider virtualbox
```

## 解决vagrant安装插件慢的问题

vagrant的插件都是ruby gems包，在vagrant嵌入的ruby环境中安装插件对应的gem就表示安装了插件。

所以，将gem的镜像源改为国内的gem镜像源即可：
```
vagrant plugin install --plugin-clean-sources --plugin-source https://gems.ruby-china.com/ <plugin>
```

例如：
```
vagrant plugin install --plugin-clean-sources --plugin-source https://gems.ruby-china.com/ vagrant-vbguest
```

如果是在bash中，可将上面一大串的命令设置为别名方式
```
$ echo "alias vagrant-install='vagrant plugin install --plugin-clean-sources --plugin-source'" >>~/.bashrc
$ exec bash
$ vagrant-install vagrant-vbguest
```

vagrant plugin install时加上--debug选项，将可以看到所使用的gem sources列表：
```
DEBUG bundler: Enabling user defined remote RubyGems sources
DEBUG bundler: Adding RubyGems source https://gems.ruby-china.com/
DEBUG bundler: Enabling default remote RubyGems sources
DEBUG bundler: Current source list for install: ["https://gems.ruby-china.com/"]
DEBUG bundler: Generating new builtin set instance.
DEBUG bundler: Generating new plugin set instance. Skip gems - []
DEBUG bundler: Generating solution set for installation
```

## vagrant使用国内box镜像源

vagrant下载国外的box镜像会很慢，但下载国内的box镜像肯定没问题。因此，只需要在使用box name的命令参数处使用对应的URL地址即可。

例如，想要下载CentOS 7的vagrant box镜像，可从中科大的镜像站点(或其他镜像站)找到对应的Box镜像。

对于vagrant box add命令：
```
vagrant box add my_centos7 http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2001_01.VirtualBox.box
```

对于vagrant init命令：
```
vagrant init my_centos7 http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2001_01.VirtualBox.box
```

对于Vagrantfile文件中指定的box_url链接：
```
config.vm.box = "my_centos7"
config.vm.box_url = "http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-2001_01.VirtualBox.box"
```

需注意，vagrant box镜像并没有一个固定的存储站点，除了目前官方使用的<https://app.vagrantup.com>汇集了几乎绝大多数的box镜像外，其他的box镜像都需要用户自己去搜索。

比较常见的centos box镜像和ubuntu box镜像，可以在它们的cloud站点找到，例如它们自己的官方站点、中科大镜像源站点：

CentOS7:
```
# CentOS cloud官方镜像站点
https://cloud.centos.org/centos/7/vagrant/x86_64/images/

# CentOS Cloud中科大镜像站点
http://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/
```

Ubuntu(以focal即20.04为例)，找到其中的xxx-yyy-vagrant.box文件：
```
# Ubuntu cloud官方镜像站点
https://cloud-images.ubuntu.com/focal/20201012/

# Ubuntu Cloud中科大镜像站点
http://mirrors.ustc.edu.cn/ubuntu-cloud-images/focal/20201007/
```

<a name="local_box"></a>

## 使用本地box镜像文件

如果手动将box镜像下载下来了，vagrant也可以指定从本地添加box镜像。

例如，本地有一个ubuntu-20.04的vagrant box镜像`V:\vagrant_imgs\focal-server-vagrant.box`。

对于vagrant box add命令：
```
vagrant box add ubunut-2004 V:\vagrant_imgs\focal-server-vagrant.box
```

对于vagrant init命令：
```
# 注意是file://前缀，且路径分隔符为/，
# 即使在Windows下使用\作为路径分隔符也将报错
vagrant init ubunut-2004 "file://V:/vagrant_imgs/focal-server-cloudimg-amd64-vagrant.box"
```

对于Vagrantfile文件中指定的box_url链接：
```
# 注意是file://前缀，且路径分隔符为/，
# 即使在Windows下使用\作为路径分隔符也将报错
config.vm.box = "ubunut-2004"
config.vm.box_url = "file://V:/vagrant_imgs/focal-server-vagrant.box"
```

## Vagrantfile中设置虚拟机的代理

使用插件vagrant-proxyconf可以配置虚拟机的一些代理，包括http_proxy/git/yum/apt/npm/docker等代理。

```
vagrant plugin install vagrant-proxyconf
```

然后在Vagrantfile中设置：
```ruby
Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
    # 设置为true表示插件支持的所有程序都设置代理，设置为false表示不设置任何代理
    # config.proxy.enabled = false
    # 表示除了npm外的其它程序都设置代理
    # config.proxy.enabled = {:npm: false}
    config.proxy.http     = "http://yourproxy:8080"
    config.proxy.https    = "https://yourproxy:8080"
    config.proxy.no_proxy = "localhost,127.0.0.1"
  end
end
```