---
title: Shell脚本深入教程：升级bash 5.0
p: shell/script_course/bash_update.md
date: 2020-05-19 14:17:00
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

--------

# 升级Bash

下载bash 5.0，并解压、编译：

```shell
wget http://ftp.gnu.org/gnu/bash/bash-5.0.tar.gz
tar xf bash-5.0.tar.gz
cd bash-5.0
./configure && make && make install
```

添加软链接：

```shell
mv /bin/bash /bin/bash.old.bak
ln -s /usr/local/bin/bash /bin/bash
```

重载Bash：

```shell
exec $SHELL
```