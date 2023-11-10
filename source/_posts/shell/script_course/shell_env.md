---
title: Shell脚本深入教程：Shell环境和子Shell的概念(★★★)
p: shell/script_course/shell_env.md
date: 2020-05-19 14:17:01
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------


# Shell环境和子Shell★★★

## shell如何让命令执行

假设敲下命令行：

```shell
echo hello
```

shell是如何让echo命令执行起来的？

shell首先读取命令行、解析命令行，解析期间会发现echo是一个外部命令。解析通过后，那么就准备让echo命令执行起来。

![](/img/shell/1589869991053.png)

因为exec是调用echo程序来替换子shell进程，所以子shell进程**继承自父shell的属性都会被覆盖**，也就是说，继承自父Shell的变量已经不存在了。

如图：

![](/img/shell/1581150563172.png)

```
+--------+
| pid=7  |
| ppid=4 |
| bash   |
+--------+
    |
    | calls fork
    V
+--------+             +--------+
| pid=7  |    forks    | pid=22 |
| ppid=4 | ----------> | ppid=7 |
| bash   |             | bash   |
+--------+             +--------+
    |                      |
    | waits for pid 22     | calls exec to run echo
    |                      V
    |                  +--------+
    |                  | pid=22 |
    |                  | ppid=7 |
    |                  | echo   |
    V                  +--------+
+--------+                 |
| pid=7  |                 | exits
| ppid=4 | <---------------+
| bash   |
+--------+
    |
    | continues
    V
```

上面是Shell如何让外部命令运行的过程。但在Shell中，可执行的内容有多种。

## Shell中的可执行程序

- 外部命令：比如cat,ls  
- shell内置命令：比如cd,set,export,read  
- shell函数  
- 别名  
- shell保留关键字：time,while,if,case,do,done,function,for等  

Shell让它们执行的方式不一样：  
1. 外部命令的执行需要先fork子shell进程，然后在子shell进程中exec调用外部命令  
2. shell函数、shell内置命令、shell保留关键字都依赖于Shell进程，没有Shell进程，它们都没有意义。它们都是直接在当前Shell进程内执行的，不会创建新的子Shell进程来执行  
3. 别名会在命令行解析阶段先替换成对应的内容，然后重新执行命令行解析  

> 补充：命令优先级
>
> 当别名、shell保留关键字、shell函数、shell内置命令、外部命令的名称有冲突时，会执行谁呢？详细内容可参考：https://www.junmajinlong.com/shell/call_order/
>
> 这里只给它们的优先级：别名>保留关键字>shell函数>shell内置命令>外部命令

## Shell环境

何为环境？在Shell中这是极其重要的概念，是深入Shell必须搞懂的内容，所以请一定要重视。

**每个shell进程有一个自己的运行环境，不同的Shell进程有不同的Shell环境。Shell解析命令行、调用命令行的过程都在这个环境中完成**。

可以把一个Shell环境想象成一个盒子。

![](/img/shell/1580893793550.png)

关于环境的几个结论：
1. 调用shell程序(比如bash)时，会读取shell配置文件来初始化Shell环境。对于bash来说，是读取：  
  - /etc/profile
  - /etc/profile.d/*.sh
  - ~/.bash_profile
  - ~/.bashrc
  - /etc/bashrc
2. 环境主要体现在对环境的设置，包括但不限于的环境设置有：
  - `cd /tmp`表示设置当前shell环境的工作目录(cd命令是一个内置命令)  
  - shopt或set命令进行的shell功能设置，即打开或关闭某个功能  
      - 比如`shopt -s globstar`表示开启当前Shell环境的双星号`**`目录递归通配功能，例如`grep "declare" /etc/**/*.sh`  
  - 环境变量设置  
      - 主要用于Shell进程和其子进程之间的数据传递  
      - **子进程(不仅仅是子Shell进程)可以继承父Shell环境中的环境变量**  
      - 环境变量通常以大写字母定义，但非一定  
      - 使用bash内置命令`export`可以定义环境变量  
      - 命令前定义变量`var=value cmd`，表示定义一个专属环境变量，该环境变量只能在cmd进程环境中可访问，其它任何地方都无法访问，cmd进程退出后，var环境变量也消失  
  - `export X=xyz`表示在当前Shell环境下定义一个环境变量`X`，以便让子进程继承这个变量  
3. 每当提到shell内置命令，就要想到这个命令的作用有可能是在当前Shell环境下进行某项设置  
4. shell内置命令不会创建新进程，而是直接在当前Shell环境内部执行  
5. 内置命令`source`或`.`执行脚本时，表示在当前Shell环境下执行脚本内容，即脚本中的所有设置操作都会直接在当前Shell下设置  
6. 父Shell环境**可能**会影响子Shell环境，但子Shell环境**一定不**影响父Shell环境，比如Shell脚本中的环境变量不会粘滞到父Shell环境中  

## 何时会进入子Shell：两种子Shell

如何是不同的Shell环境？查看`$BASHPID`的值即可，如果变量的值不同，说明进入了新的Shell环境。
```shell
echo $BASHPID
```

有两种类型的子Shell：  
- (1).当前Shell进程因命令行中使用了某些特殊语法而产生的子Shell：  
  - 小括号`(cmd1;cmd2)`  
  - 管道`cmd1|cmd2`   
  - 命令替换`$(cmd)`  
  - 进程替换`<(cmd),>(cmd)`  
- (2).显式调用bash程序产生的子Shell进程：  
  - `bash -c 'cmd'`  
  - bash命令  
  - shell脚本  

第一种进入子Shell的方式，在fork产生子bash进程后不会执行exec，所以会继承父Shell很多设置，比如继承普通变量。

第二种进入子Shell的方式，在刚fork产生子bash进程时也会继承父Shell设置，但因为exec调用bash程序会替换覆盖掉子bash进程，所以这种方式只会保留环境变量。

第二种方式的子Shell环境更应该称为一个独立的进程环境，只不过这个进程是Shell进程，且和父Shell有父子进程的关系，所以也可以称为子Shell。

另外需要注意，**shell内置命令、shell函数、shell保留关键字都不会主动开启子Shell，但如果将它们放在管道前后，由于它们依赖于shell进程，所以会单独创建一个新的子shell进程提供它们的运行环境**。

比如，Shell环境1中的设置、变量，在第一种形式的子Shell环境2中都生效，在第二种形式的子Shell环境3中都不生效。

```shell
$ echo $BASHPID
$ x=10
$ echo $x
$ shopt -s globstar
$ shopt globstar 
globstar        on

# 进入第一种形式的子Shell环境
$ (echo $x)         # 10
$ (shopt globstar)
globstar        on

# 进入第二种形式的子Shell环境
$ bash
$ echo $x
$ shopt globstar
globstar        off
```