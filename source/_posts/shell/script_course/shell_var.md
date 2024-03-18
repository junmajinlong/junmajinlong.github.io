---
title: Shell脚本深入教程：Bash变量
p: shell/script_course/shell_var.md
date: 2020-05-19 14:17:02
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash变量

## 变量赋值

等号`=`左右两边必须不能出现空白。当包含了特殊符号时，需要使用引号包围。使用单引号还是双引号，后面再详细解释。

```shell
var1=22            # 数值
var2="hello world" # 字符串
var3=hello world   # 错误
```

问题：`var3=hello world`为什么报错的是没有world命令，而不是赋值错误？为什么`echo = 3`不是赋值？

![](/img/shell/1589871297231.png)

其实经常会看到这种用法，比如下面的命令模式，表示cmd命令执行时设置locale环境为C，但非cmd命令则不受影响：

```shell
LC_ALL=C cmd
```

使用`unset`内置命令可以注销变量：

```shell
var="hello world"
echo $var
unset var
echo $var
```

使用`readonly`内置命令可以定义只读变量，只读变量不可修改、不可注销。

```shell
readonly xyz="helloworld"
abc="hello world"
readonley abc
```

## 变量引用

使用`$VAR`或`${VAR}`，前者是简写，后者是规范引用。

例如：

```shell
var='hello world'
echo $var
```

一定注意：变量的名称是`var`，而不是`$var`，`$var`是在引用、访问变量在内存中保存的值。

使用`{% raw %}${#VAR}{% endraw %}`获取变量VAR保存的字符长度。

```shell
a="hello world"
echo ${#a}
```

当`$VAR`引用方式会产生歧义时，就用`${VAR}`，例如：

```shell
VAR="hello"
echo $VARa      # 访问变量VARa，但它不存在
echo ${VAR}a    # 访问变量VAR，并将结果和a串联起来
```

## 间接引用变量(动态变量)

假设有变量`var=value`，当通过`${!var}`语法来引用变量时，将表示先获取变量`${var}`的值`value`，再获取变量名为`value`的值。这种语法称为变量的间接引用。

```shell
value=hello
var=value
echo ${var}  # 输出 value
echo ${!var} # 输出 hello
```

有些时候需要获取动态变量的值(比如通过传参来获取指定参数对应的变量值)，变量的间接引用将非常方便。

## 环境变量

在Shell脚本中偶尔会考虑用环境变量，但是环境的概念在Shell中很重要。

环境变量可以看作是一个全局变量，但这说法并不正确。后面我会详细解释何为Shell环境，这是Shell中非常重要的概念，到时候大家就会对环境变量有一个非常清晰的认识。

使用bash内置命令`export`可以定义一个环境变量：

```shell
# 直接定义一个新的环境变量
$ export ENV_VAR="hello world"

# 将一个已存在的普通变量导出为环境变量
$ ENV_VAR1="HELLOWORLD"
$ export ENV_VAR1
```

环境变量一般以大写字母命名。

子Shell进程可以继承父Shell中的环境变量：

```shell
x=1000
y=10000
export y
bash -c 'echo $x;echo $y'
```

使用`env`命令可以查看所有环境变量。

```shell
env
```

可以在某个命令行前设置变量，便表示**设置该命令的专属环境变量**，只有该命令进程中可访问该环境变量，其它任何地方都无法访问，且命令退出后专属环境变量消失。

```shell
# 下面等价
xyz=10 bash -c 'echo $xyz'
env xyz=100 bash -c 'echo $xyz'

# 环境变量只对cmd1有效
xyz=10 cmd1 | cmd2

# 环境变量对cmd1和cmd2都有效
xyz=10 cmd1 | xyz=10 cmd2
xyz=55 bash -c 'cmd1 | cmd2'
```

常见的环境变量：

```
HOSTNAME=control_node
USER=root
HOME=/root
SHELL=/bin/bash
HISTSIZE=1000
SSH_TTY=/dev/pts/2
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
MAIL=/var/spool/mail/root
PWD=/root
LANG=en_US.UTF-8
```

特别要注意的是PATH环境变量，它决定了Shell调用命令时的搜索路径。经常会设置PATH环境变量，特别是应用在定时任务的Shell脚本。

```shell
PATH=/usr/local/mysql/bin:$PATH
```

比如，编译安装或直接解压安装程序时，通常会将PATH写入到`/etc/profile.d/*.sh`下：

```shell
echo 'PATH=/usr/local/mysql/bin:$PATH' >/etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh
```

## 位置参数和特殊变量

### 位置参数

Shell脚本运行时可能需要一些选项或参数。

比如，某脚本test.sh内容:

```shell
#!/bin/bash
cat $1
touch $2
```

这表示先读取并输出第一个参数表示文件内容，然后touch第二个参数表示的文件。执行时：

```shell
chmod +x test.sh
./test.sh /etc/fstab /tmp/touchme.log
```

脚本执行时的选项(本例没有选项)和参数都是脚本的位置参数，位置参数使用`$1,$2,$3,...`引用。

`$0`比较特殊，表示脚本名或当前Shell名称。

**位置参数是相对于每个Shell进程而言的**。对于Shell脚本来说，位置参数就是执行脚本时的参数部分。当处于Shell环境内时，也可以使用`set --`设置当前Shell环境位置参数：

```shell
set -- a b c
echo $1
echo $@
echo $#
set --   # 清空当前Shell的位置参数
echo $#
```

### 特殊变量

Shell有一些特殊变量，这些变量由Shell自身动态维护，不允许用户手动修改。

为了让这些特殊变量更直观，我使用变量引用而不直接使用特殊变量名。

```
$1,$2,...,$N：脚本的位置参数
$0：shell或shell脚本的名称  
$*：扩展为位置参数，"$*"会将所有位置参数一次性包围引起来，"$*"等价于"$1_$2_$3..."
$@：扩展为位置参数，"$@"会将每个位置参数单独引起来，"$@"等价于"$1" "$2" "$3"...
$#：位置参数的个数
$$：当前Shell的进程PID，在某些子Shell(如小括号()开启的子Shell)下，会被继承。如果可以，建议使用$BASHPID替代$$
$?：最近一个前台命令的退出状态码  
$!：最近一个后台命令的进程PID
$-：当前Shell环境的一些特殊设置，比如是否交互式
$_：最近一个前台命令的最后一个参数（还有其它情况，该变量用的不多，所以不追究了）
```

关于`$* "$*" $@ "$@"`的区别，参见如下shell脚本测试：

```shell
#!/bin/bash

echo '$*----------':
for i in $*;do echo "<$i>";done

echo '"$*--------"':
for i in "$*";do echo "<$i>";done

echo '$@----------':
for i in $@;do echo "<$i>";done

echo '"$@--------"':
for i in "$@";do echo "<$i>";done
```

执行：

```shell
$ chmod +x position_parameters.sh
$ ./position_parameters.sh a b 'c d'
$*---------:
<a>
<b>
<c>
<d>
"$*--------":
<a b c d>
$@----------:
<a>
<b>
<c>
<d>
"$@--------":
<a>
<b>
<c d>
```


## shift踢掉位置参数

Bash内置命令shift专门用来踢位置参数。在为Shell脚本设计选项、参数时，都会用到shift。

```shell
shift [N]
```

踢掉前N个位置参数，如果没有指定N参数，则默认N=1，即一次踢掉一个位置参数。

踢掉位置参数后，后面的位置参数会向前移动。

```shell
$ set -- a b c d e f
$ echo ${#@}
$ echo $1
a
$ shift 2
$ echo ${@}
c d e f
$ echo ${1}
c
```

例如，自定义ping命令的选项、参数。

```shell
#!/bin/bash
while [ ${#@} -gt 0 ];do
  case "$1" in 
    -c|--count)
      count=$2
      shift 2
      ;;
    -t|--timeout)
      timeout=$2
      shift 2
      ;;
    -a|--ip)
      ip=$2
      shift 2
      ;;
    *)
      echo "wrong options or arguments"
      exit 1
  esac
done

ping -c $count -W timeout $ip
```
