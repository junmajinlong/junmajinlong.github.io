---
title: 精通awk系列(1)：安装新版本的gawk
p: shell/awk/gawk_install.md
date: 2019-11-23 10:37:30
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# 安装新版本gawk

awk有很多种版本，例如nawk、gawk。gawk是GNU awk，它的功能很丰富。

本教程采用的是gawk 4.2.0版本，4.2.0版本的gawk是一个比较大的改版，新支持的一些特性非常好用，而在低于4.2.0版本时这些语法可能会报错。所以，请先安装4.2.0版本或更高版本的gawk。

查看awk版本

```
awk --version
```

这里以安装gawk 4.2.0为例。

```
# 1.下载
wget --no-check-certificate https://mirrors.tuna.tsinghua.edu.cn/gnu/gawk/gawk-4.2.0.tar.gz

# 2.解压、进入解压后目录
tar xf gawk-4.2.0.tar.gz
cd gawk-4.2.0/

# 3.编译，并执行安装目录为/usr/local/gawk4.2
./configure --prefix=/usr/local/gawk4.2 && make && make install

# 4.创建一个软链接：让awk指向刚新装的gawk版本
ln -fs /usr/local/gawk4.2/bin/gawk /usr/bin/awk

# 此时，调用awk将调用新版本的gawk，调用gawk将调用旧版本的gawk
awk --version
gawk --version
```


