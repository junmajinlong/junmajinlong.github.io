---
title: 安装Ruby和安装Rails
p: ruby/ruby_rails_install.md
date: 2020-05-15 17:46:16
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# rbenv安装Ruby

rbenv可以管理多个版本的ruby。可以分为3种范围(或者说不同生效作用域)的版本：  

- local版：本地，针对各项目范围(只在某个目录下有效)  

- global版：全局，没有shell和local版时使用global版  

- shell版：当前终端，只针对当前所在终端  

查找优先级为`shell>local>global`。

## 安装rbenv和Ruby

1.安装rbenv

```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL
```

2.安装ruby-build工作，可自动编译安装ruby。它可以作为rbenv的插件，也可以作为独立程序，建议采用插件的方式。(如果已经有了，就跳过这一步，只要确保有rbenv命令就可以)


```bash
# 作为rbenv插件
mkdir -p "$(rbenv root)"/plugins
git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build

# 作为独立程序
git clone https://github.com/rbenv/ruby-build.git ~/ruby-build
PREFIX=/usr/local ./ruby-build/install.sh
```

3.选择ruby版本，安装ruby。安装Ruby之前，可能需要先安装一些依赖包，所需依赖包以及常见安装方式，以及如何开启YJIT功能，见[安装依赖包以及开启YJIT](#dep)。

```bash
rbenv install --list
rbenv install 3.2.2
```

默认情况下，安装是很慢的，因为要从官方下载源码包进行编译，下载的过程非常慢。所以，参见后面[解决rbenv安装慢问题](#installslow)。

如果编译失败，可能是少了一些依赖包，在编译失败的时候会提示你执行什么命令来安装这些包(非常人性)。比如需要readline-devel包。

```bash
yum -y install readline-devel
```

4.安装完ruby或切换了ruby之后，都需要执行rehash操作，让rbenv知道刚才新装了一个ruby。

```bash
rbenv rehash
```

5.进入到项目目录/ror/ror1，设置local ruby版本

```bash
cd /ror/ror1
rbenv local 3.2.2
```

6.设置gem源

```bash
# 注意是ruby-china.com/，ruby-china.org的域名已经改成了.com
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
gem sources -l
```

<a name="dep"></a>

## 安装Ruby所需的依赖包以及开启YJIT

下面是安装Ruby所需的包：

```bash
# apt系的包管理系统(如果libgdbm6不存在，尝试libgdbm5)
apt-get install autoconf patch build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libgmp-dev libncurses5-dev libffi-dev libgdbm6 libgdbm-dev libdb-dev uuid-dev

# yum系的包管理系统
yum install -y gcc-6 patch bzip2 openssl-devel libyaml-devel libffi-devel readline-devel zlib-devel gdbm-devel ncurses-devel
```

如果需要开启YJIT的即时编译功能，还需要安装Rust。

```bash
curl -sSl https://sh.rustup.rs | sh -s -- -y
exec $SHELL
```

然后再安装ruby：

```bash
rbenv install 3.2.2
```

<a name="installslow"></a>

## 解决rbenv安装慢问题

**方案1**：

从 https://cache.ruby-china.com/pub/ruby/ 将ruby对应版本文件下载下来，将文件丢到~/.rbenv/cache目录下。

注意点：  
1. ~/.rbenv/cache目录可能不存在，需要先创建  
2. 下载保存下来的版本可能不是rbenv install时所需的版本，因为同一个版本的文件有.tar.bz2的，有.tar.xz的等等，rbenv对安装不同的ruby版本使用的文件后缀可能不一样，可以先执行下`rbenv install 3.2.2`后立马ctrl+c，再去下载显示出来对应后缀的包  

以下是一个示例：
```bash
#  先rbenv install看看使用什么后缀的版本文件
# 这里显示的是使用.tar.bz2后缀的文件
$ rbenv install 3.2.2
Downloading ruby-3.2.2.tar.bz2...
^C

# 所以下载.tar.bz2的文件
$ wget 'https://cache.ruby-china.com/pub/ruby/3.2/ruby-3.2.2.tar.bz2' -P ~/.rbenv/cache

# 安装即可
$ rbenv install 3.2.2
```

**方案2**： 

可以从 https://cache.ruby-china.com/pub/ruby/ 将ruby对应版本文件下载下来，然后安装。但注意先设置环境变量，并且在此环境变量url之后加上特殊符号`#`或`?`：

```bash
# 以ruby-3.2.2为例
wget https://cache.ruby-china.com/pub/ruby/3.2/ruby-3.2.2.tar.bz2 -P ~
RUBY_BUILD_MIRROR_URL='file:///~/ruby-3.2.2.tar.bz2#' rbenv install 3.2.2 --verbose

# 另：也可以设置代理https_proxy=IP:PORT加速下载
```

**方案3**：

有时候上面的方案2会失效，不同版本可能不一样。但是，这里可以使用一个rbenv插件，让rbenv直接使用中国的镜像站点下载。直接执行下面的命令即可。

```bash
git clone https://github.com/andorchen/rbenv-china-mirror.git "$(rbenv root)"/plugins/rbenv-china-mirror
```

## 更新rbenv的ruby版本列表

安装rbenv一段时间之后，ruby可能发布了新的版本，这时rbenv无法获取到这个新版本的信息。因此需要更新rbenv的可安装列表。

实际上，更新ruby-build插件即可：

```bash
# ruby-build作为rbenv插件时
git -C "$(rbenv root)"/plugins/ruby-build pull

# ruby-build作为独立程序时
cd
git clone https://github.com/rbenv/ruby-build.git
PREFIX=/usr/local ./ruby-build/install.sh
```

然后就可以查看新的ruby版本并安装。

## 多版本ruby

上面已经装了一个ruby了，现在再装一个ruby 3.2.1：

```bash
# 以ruby-3.2.1为例
$ wget https://cache.ruby-china.com/pub/ruby/3.2/ruby-3.2.1.tar.bz2 -P /root

$ RUBY_BUILD_MIRROR_URL='file:///~/ruby-3.2.1.tar.bz2#' rbenv install 3.2.1 --verbose

$ rbenv rehash
```

现在，就有了两个版本，可以使用`rbenv versions`命令查看（复数versions表示列出已装所有版本，单数version表示列出当前所使用的ruby版本）。

```bash
$ rbenv versions
```

现在，就可以通过`rbenv [local | shell | global]  VERSION`来设置多版本共存的ruby了。

比如：

```bash
$ rbenv local 3.2.1
$ rbenv version
```

## rbenv命令行

```bash
$ rbenv --help
Usage: rbenv <command> [<args>]

Some useful rbenv commands are:
   commands    列出rbenv的所有命令列表
   local       设置或显示local application-specific Ruby version
   global      设置或显示global Ruby version
   shell       设置或显示shell-specific Ruby version
   install     使用ruby-build安装指定的ruby版本
   uninstall   卸载指定版本
   rehash      rehash，每次安装完ruby后都要执行，否则rbenv不知道刚才新装ruby的信息
               (rbenv通过检查~/.rbenv/shims来获取ruby信息)
   version     显示当前ruby版本
   versions    显示所有已装ruby版本
   which       显示ruby命令的全路径
   whence      列出包含该可执行命令的所有ruby版本

See `rbenv help <command>' for information on a specific command.
For full documentation, see: https://github.com/rbenv/rbenv#readme
```

完整的命令列表可查看`rbenv commands`，各命令使用方法，可查看`rbenv help COMMAND`。


# 安装rails

```bash
cd /ror/ror1

# 查看已有的rails版本号
gem list --remote | grep '^rails' | head

# 安装最新版的rails
gem install rails

# 安装指定版本的rails
# gem install rails -v VERSION
gem install rails -v 5.1.3
```

安装了指定版本的rails后，rails创建的项目不一定就是指定版本的。比如上面安装的是5.1.3版本的rails，`rails new blog`可能会创建rails 6.0.3.2版本的项目blog。如果想要让创建的项目也是指定版本的，可：

```
rails _5.1.3_ new blog
```

# Windows安装Ruby和Rails

下载Windows下的Ruby安装包：<https://rubyinstaller.org/downloads/>。

要下载with-devkit的。例如：

```
https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-2.6.6-1/rubyinstaller-devkit-2.6.6-1-x64.exe
```

如果下载慢，上这下载<https://download.csdn.net/download/a905815661/12450196>。

下载OK后，双击安装，一路点下一步：

![](/img/ruby/1590202994214.png)

最后安装ruby所需的包：

![](/img/ruby/1590203276305.png)

安装完成后，打开cmd或powershell：更改中国gem镜像仓库。

```
gem sources --remove https://rubygems.org/ --add https://gems.ruby-china.com/
```

安装rails或其它gem：

```
gem install rails
gem install mysql2
```