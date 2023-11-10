---
title: Perl文件名通配和文件查找
p: perl/perl_glob_find.md
date: 2019-07-06 17:37:57
tags: Perl
categories: Perl
---

# Perl文件名通配和文件查找

在shell中使用`*`来对文件名进行通配扩展，在Perl中也同样支持文件名通配。而且perl中的glob通配方式和shell的通配方式完全一致，实际上perl的glob函数就是直接调用csh来通配的(如果不存在csh，则使用其它shell)，也因此通配是一个效率较低的操作。

## glob通配函数

```
元字符    意义
--------------------------------
[]       字符类，匹配中括号中的任一字符。
*        匹配任意个字符
?        匹配任意单个字符，注意，不匹配0个
~        匹配家目录
```

注意：
- `[]`支持`[0-9] [a-z] [A-Z] [A-z]`类似的范围通配，其中`[A-z]`等价于`[A-Za-z]`
- 不支持`[^]`取反
- 特别需要注意的是`[1-34]`通配的是`[1234]`，而不是1到34，因为中括号只能匹配单个字符


例如：
```
use 5.010;

say glob "*";      # 匹配当前目录下所有非"."开头的隐藏文件
say glob ".* *";   # 匹配当前目录下所有文件，包括"."开头的隐藏文件
say glob "23.p[ly]";  # 匹配23.pl或23.py文件
say glob "[0-9][0-9].pl"   # 匹配两个数值开头的pl文件
say glob "/root/23.p?";   # 匹配家目录下后缀以p开头，后面还有一个字符的文件
say glob "~/*.sh";  # 匹配家目录下的所有sh文件
```

在glob函数中，空格具有特殊含义，如果想要匹配包含空格的文件名，必须将其使用引号(单/双引号皆可)包围。


例如，要匹配`hello world.log`文件：
```
glob "'hello w*.log'";
glob '"hello w*.log"';
glob qq('hello w*.log');
glob qq("hello w*.log");
glob q('hello w*.log');
glob q("hello w*.log");
```

File\::Glob模块提供更丰富的通配规则，可以去查看下手册。不过说实话，用到的几乎应该不多。


## 尖括号<>通配写法

在glob出现之前，人们都使用尖括号表达式来通配。它和glob的实现是完全一致的，仅仅只是从尖括号改为了glob函数。

例如，匹配/root下的sh文件，下面两种写法完全等价：
```
say </root/*.sh>;
say glob "/root/*.sh";
```

还可以使用变量替换：
```
$dir="/etc";
my @file_list = glob "$dir/*.sh $dir/*.py";
my @file_list = <$dir/*.sh $dir/*.py>;
```

同样，匹配包含空格的文件，可能需要使用引号包围：
```
say <"hello w*.log">;
```

在这里需要搞清楚尖括号内的到底会被解析成文件句柄还是解析成通配符。perl的解析规则是：假如尖括号内的内容满足标识符规则(文件句柄的名称要满足此规则)，则会解析为文件句柄，否则解析成通配符。

以下是几种情况的区分示例：
```
my @files = <FOO/*>;     # 文件名通配
my @lines = <FOO>;       # 读取文件句柄
my @lines = <$fred>;     # 读取文件句柄
my $name = 'FOO';
my @files = <$name>;     # 读取文件句柄
my @files = <$name/*>;   # 文件名通配
```

从第三行和第5行可以看出，当使用变量且只有变量名替换的时候，会优先解析为文件句柄。


## 文件查找：关于find2perl脚本

在unix下有一个find工具，用来查找文件非常方便。perl提供了一个find2perl的工具(该工具是在安装perl时自带的)，它可以将find查找文件时的表达式转换成perl对应的查找语句。

find2perl的选项和用法和find的用法99%都一样，只有几项额外的是find2perl自身提供的，但这样的选项非常少。

注意，find2perl不是文件查找工具，而是将我们写的find命令表达式转换为等价的perl文件查找语句。

例如，搜索/etc目录下所有`.cnf`结尾的文件，find命令的表达式如下：
```
find /etc -type f -name "*.cnf"
```

执行find2perl：
```
[root@xuexi perlapp]# find2perl /etc/ -type f -name "*.cnf"
#! /usr/bin/perl -w
    eval 'exec /usr/bin/perl -S $0 ${1+"$@"}'
        if 0; #$running_under_some_shell

use strict;
use File::Find ();

# Set the variable $File::Find::dont_use_nlink if you're using AFS,
# since AFS cheats.

# for the convenience of &wanted calls, including -eval statements:
use vars qw/*name *dir *prune/;
*name   = *File::Find::name;
*dir    = *File::Find::dir;
*prune  = *File::Find::prune;

sub wanted;



# Traverse desired filesystems
File::Find::find({wanted => \&wanted}, '/etc/');
exit;


sub wanted {
    my ($dev,$ino,$mode,$nlink,$uid,$gid);

    (($dev,$ino,$mode,$nlink,$uid,$gid) = lstat($_)) &&
    -f _ &&
    /^.*\.cnf\z/s
    && print("$name\n");
}
```

可以看出，上面生成了一个wanted子程序，以后要查找`/etc/*.cnf`文件时，只需调用wanted子程序即可。例如：
```
[root@xuexi perlapp]# find2perl /etc/ -type f -name "*.cnf" >24.plx
[root@xuexi perlapp]# chmod +x 24.plx 
[root@xuexi perlapp]# perl 24.plx 
/etc/my.cnf
/etc/proxysql.cnf
/etc/pki/tls/openssl.cnf
```