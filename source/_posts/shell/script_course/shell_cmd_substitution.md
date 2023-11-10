---
title: Shell脚本深入教程：Bash命令替换
p: shell/script_course/shell_cmd_substitution.md
date: 2020-05-19 14:19:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash命令替换

使用反引号(在波浪线的按键上)或者`$()`来执行命令替换。

命令替换是指：先执行`$()`中的命令，将命令的输出结果替换到命令行`$()`位置处。

所以，命令替换和变量替换差不多，都是在命令开始执行前执行，并将结果替换到命令行。

如：

```shell
$ ls -l $(which sh)
$ a=$(date +"%s.%N")
$ echo $a
```

![](/img/shell/1589872355594.png)

例如：

```shell
$ echo $(ls -1 /etc/m*.conf)
/etc/man_db.conf /etc/mke2fs.conf
$ echo "$(ls -1 /etc/m*.conf)"
/etc/man_db.conf
/etc/mke2fs.conf
```

例如，对目录下的mp4文件重命名，比如：

```
P1. 01.了解jQuery(1).mp4       -> 01.了解jQuery.mp4
P2. 02.jQuery的基本使用(2).mp4  -> 02.jQuery的基本使用.mp4
P3. 03.jQuery的2把利器(3).mp4   -> 03.jQuery的2把利器.mp4
P4. 04.jQuery函数的使用(4).mp4  -> 04.jQuery函数的使用.mp4
P5. 05.jQuery对象的使用(5).mp4  -> 05.jQuery对象的使用.mp4
P6. 06.基本选择器(26).mp4       -> 06.基本选择器.mp4
P7. 07.层次选择器(7).mp4        -> 07.层次选择器.mp4
P8. 08.过滤选择器(128).mp4      -> 08.过滤选择器.mp4
```

命令如下：

```shell
touch "P1. 01.了解jQuery(1).mp4"
touch "P2. 02.jQuery的基本使用(2).mp4"
touch "P3. 03.jQuery的2把利器(3).mp4"
touch "P4. 04.jQuery函数的使用(4).mp4"
touch "P5. 05.jQuery对象的使用(5).mp4"
touch "P6. 06.基本选择器(26).mp4" 
touch "P7. 07.层次选择器(7).mp4"   
touch "P8. 08.过滤选择器(128).mp4"  
for i in *.mp4 ;do
  mv "$i" "`echo $i | sed -r 's/.* ([0-9]{1,3}\..*)\([0-9]{1,3}\)\.mp4/\1.mp4/'`"
done
```
