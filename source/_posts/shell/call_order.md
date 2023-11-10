---
title: Shell内置命令、外部命令、别名、函数、保留关键字优先级
p: shell/call_order.md
date: 2019-11-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# Shell内置命令、外部命令、别名、函数、保留关键字的优先级

在Shell中，有5种可调用的东西：别名(alias)、函数(function)、shell保留关键字、shell内置命令、外部命令。

如果它们同名了，那么优先调用谁呢？可使用`type -a cmd`查看。

```
# 内置命令、别名、函数、外部命令
$ alias kill="echo haha"
$ function kill()(echo hehe)

$ type -a kill
kill is aliased to `echo haha'   # 1.别名kill
kill is a function               # 2.函数kill
kill () 
{ 
    ( echo hehe )
}
kill is a shell builtin      # 3.内置kill
kill is /usr/bin/kill        # 4.外部kill

# 别名、函数、保留关键字、外部命令
$ alias time="echo haha"
$ function time()(echo hehe)  

$ type -a time
time is aliased to `echo haha'  # 1.别名
time is a shell keyword         # 2.保留关键字
time is a function              # 3.函数
time () 
{ 
    ( echo hehe )
}
time is /usr/bin/time            # 4.外部命令
```

![](/img/shell/733013-20191106210813754-634588161.jpg)


例如：

```
# 调用外部命令time
[root@me ~]$ command time echo haha
[root@me ~]$ /usr/bin/time echo haha

# 调用内置命令printf
[root@me ~]$ alias printf="echo hehe"
[root@me ~]$ printf
hehe
[root@me ~]$ builtin printf 'hello'
hello
```

此外，可以使用反斜线对命令转义，使其跳过别名和保留关键字的阶段。除了最常见的在命令前面使用反斜线外，其他应用方式都可以禁止别名扩展和解析成保留关键字。下面几种方式等价：

```
$ \time echo haha
$ t\i\m\e echo haha
$ "time" echo haha
$ "ti"'me' echo haha
```








