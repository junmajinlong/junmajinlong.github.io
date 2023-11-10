---
title: Shell脚本深入教程：Bash路径通配规则
p: shell/script_course/shell_glob.md
date: 2020-05-19 14:17:28
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash路径通配规则

```
*：匹配任意个任意字符
?：匹配任意单个字符
[]：中括号通配。包括下面几种模式：
  [abc]：匹配中括号中任意单个字符，即匹配a或b或c均可
  [^abc]和`[!abc]`：匹配非中括号中的任意单个字符
  [a-z]：匹配abc...z，但和Locale环境的排序规则有关。
         Locale C环境下，[a-d]表示abcd，字典排序规则的[a-d]表示aBbCcDd
  字符类：
    [:alpha:]、[:alnum:]、[:ascii:]、[:blank:]、[:cntrl:]、
    [:digit:]、[:graph:]、[:lower:]、[:print:]、[:punct:]、
    [:space:]、[:upper:]、[:word:]、[:xdigit:]
```

此外:

(1).如果开启了globstar，则双星号`**`可以递归匹配目录。例如：

```shell
shopt -s globstar
grep 'PATH' /etc/**/*.sh
```

(2).如果开启了dotglob，则`*`可以匹配以点开头的隐藏文件。例如：

```shell
shopt -s dotglob
ls ~/*bashrc
```

(3).如果开启了extglob，则可以使用下面列出的复杂匹配模式。下面的pattern-list是一个或多个简单pattern，多个pattern之间使用竖线`|`隔开，且下面的模式可以组合。

```
?(pattern-list)
匹配0到1次

*(pattern-list)
匹配0或多次

+(pattern-list)
匹配1或多次

@(pattern-list)
多选一

!(pattern-list)
匹配不能被pattern匹配的部分
```

例如：

```shell
$ touch abcdef33.sh ABCdef4f.sh
$ ls *(abc*|ABC*)?([:digit:]*)
abcdef33.sh ABCdef4f.sh
```

更全的通配介绍：

```shell
# 1.通配符(包括*)会扩展成所有能匹配的文件名
$ echo ls /*tc    # ls /etc

# 2.开启globstar(默认没开启)，可以让连续的两个星号**递归匹配
$ shopt -s globstar
$ echo ls /etc/**/*.conf

# 3.开启dotglob(默认没开启)，可以让星号*匹配以点开头的隐藏文件
$ shopt -s dotglob
$ echo ls ~/*bashrc  # ls $HOME/.bashrc

# 4.开启nocaseglob(默认没开启)，可以在匹配文件名的时候忽略大小写
$ shopt -s nocaseglob
$ echo ls *TC     # ls etc

# 5.默认情况下，如果没有文件名匹配成功，则保留通配符号
$ echo ls /*tcx   # ls /*tcx

# 6.如果开启failglob(默认没开启)，如果没有文件名可以匹配成功，将报错，而不是默认情况下的保留通配符
$ shopt -s failglob
$ echo ls *tcx    #  -bash: no match: *tcx

# 7.如果开启nullglob(默认没开启)，如果没有文件名可以匹配成功，则带有通配符部分的单词全都替换成空字符串
$ shopt -s nullglob
$ echo ls *tcx    # ls
```
