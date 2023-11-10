---
title: Shell脚本深入教程：Bash流程控制语句
p: shell/script_course/shell_flow_control.md
date: 2020-05-19 14:20:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash流程控制语句

## if语句

```shell
if test-commands; then
  consequent-commands;
[elif more-test-commands; then
  more-consequents;]
[else alternate-consequents;]
fi
```

test-commands既可以是test测试或`[]、[[]]`测试，也可以是任何其它命令，test-commands用于条件测试，它只判断命令的退出状态码是否为0，为0则为true。

例如：

```shell
if [ "$a" ];then echo '$a' is not none;else echo '$a' undefined or empty;fi

if [ ! -d ~/.ssh ];then
  mkdir ~/.ssh
  chown -R $USER.$USER ~/.ssh
  chmod 700 ~/.ssh
fi

if grep 'junmajinlong' /etc/passwd &>/dev/null;then
  echo 'User "junmajinlong" already exists...'
elif grep 'malongshuai' /etc/passwd &>/dev/null;then
  echo 'User "malongshuai" already exists...'
else
  echo 'you should create user,exit...'
  exit 1
fi
```

## case

case常用于确定的分支判断。比如：

```shell
while [ "$1" ];do
  case "$1" in
    start)
      echo start;;
    stop)
      echo stop;;
    restart)
      echo restart;;
    reload | force-reload)
      echo reload;;
    status)
      echo status;;
    *)
      echo $"Usage: $0 {start|stop|status|restart|reload|force-reload}"
      exit 2
  esac
done
```

case用法基本要求：
- 每个小分句中的pattern部分都使用括号『()』包围，只不过左括号『(』不是必须的  
- 每个小分句的pattern支持**通配模式匹配，可使用『|』分隔多个通配模式**，表示满足其中一个模式即可  
  - 例如`([yY]|yY][eE][sS]])`表示即可以输入单个字母的y或Y，还可以输入yes三个字母的任意大小写格式  
- 最后一般会定义一个能匹配其它任意条件的默认分支，即`(*)`  
- 除最后一个分支外，每个分支都建议以`;;`结尾，但还支持其它结尾符号：`;&`或`;;&`，这三个结尾符号分别表示：
  - `;;`结尾符号表示小分句执行完成后立即退出case语句  
  - `;&`表示继续执行下一个小分句的命令体，而无需进行匹配动作，并由此小分句的结尾符号来决定是否继续操作下一个小分句  
  - `;;&`表示继续向后(不止是下一个，而是一直向后)匹配小分句，如果匹配成功，则执行对应小分句中的command部分，并由此小分句的结尾符号来决定是否继续向后匹配  

例如：

```shell
set -- y
case "$1" in
    ([yY]|[yY][eE][sS])
        echo yes;&
    ([nN]|[nN][oO])
        echo no;;
    (*)
        echo wrong;;
esac
yes
no

set -- y
case "$1" in
    ([yY]|[yY][eE][sS])
        echo yes;;&
    ([nN]|[nN][oO])
        echo no;;
    (*)
        echo wrong;;
esac
yes
wrong
```

## for循环

有两种for循环结构：

```shell
# 成员测试类语法
for i [ [in [words …] ] ; ] do commands; done

# C语言for语法
for (( expr1;expr2;expr3 ));do cmd_list;done
```

成员测试类的for循环中，in关键字后是默认使用空格分隔的一个或多个元素，for循环时，每次从in关键字后面取一个元素并赋值给i变量。

例如：

```shell
$ for i in 1 2 "3 4";do echo $i;done
1
2
3 4
```

如果省略in words，则等价于`in "$@"`，即迭代位置参数。例如：

```shell
set -- a b c
for i do echo $i;done
for i;do echo $i;done
```

C语言型的for语法中，expr1是初始化语句，expr2是循环终点条件判断语句，expr3是每轮循环后执行的语句，一般用来更改条件判断相关的变量。

```shell
for ((i=1;i<=3;++i));do echo $i;done
1
2
3
```

对于成员测试类的语法，两点需要注意：  
1. 命令行解析时，路径扩展的过程在单词分割过程之后  
2. 迭代的元素中包含了空白  

```shell
touch "aa aaa.txt"
touch "bb bbb.txt"
for i in *.txt;do	echo $i;done 
for i in $(ls *.txt);do echo $i;done
(IFS=$'\n';for i in $(ls *.txt);do echo $i;done)
```

现在记住结论，后面介绍命令行解析的时候再做解释。

## while循环

```shell
while test_cmd_list;do cmd_list;done
```

while循环，开始时会测试`test_cmd_list`，如果测试的退出状态码为0，则执行一次循环体语句`cmd_list`，然后再测试`test_cmd_list`，一直循环，直到测试退出状态码非0，循环退出。

例如：

```shell
let i=1,sum=0;
while [ $i -le 10 ];do 
  let sum=sum+i
  let ++i
done
```

还有until循环语句，但在Shell中用的很少。

while循环经常会和read命令一起使用，read是Bash的内置命令，可用来读取文件，通常会按行读取：每次读一行。

例如：
```shell
cat /etc/fstab | while read line;do
  let num+=1
  echo $num: $line
done
```

上面的命令行中，首先cat进程和while结构开始运行，while结构中的read命令从标准输入中读取，也就是从管道中读取数据，每次读取一行，因为管道中最初没有数据，所以read命令被阻塞处于数据等待状态。当cat命令读完文件所有数据后，将数据放入到管道中，于是read命令从管道中每次读取一行并将所读行赋值给变量line，然后执行循环体，然后继续循环，直到read读完所有数据，循环退出。

但注意，管道两边的命令默认是在子Shell中执行的，所以其设置的变量在命令执行完成后就消失。换句话说，在父Shell中无法访问这些变量。比如上面的num变量是在管道的while结构中设置的，除了在while中能访问该变量，其它任何地方都无法访问它。

如果想要访问while中赋值的变量，就不能使用管道。如果是直接从文件读取，可使用输入重定向，如果是读取命令产生的数据，可使用进程替换。
```shell
while read line;do
  let num1+=1
  echo $num1: $line
done </etc/fstab
echo $num1

while read line;do
  let num2+=1
  echo $num2: $line
done < <(grep 'UUID' /etc/fstab)
```

## select选项选择

select可提供选项给用户选择。

```shell
select name [ in word ] ; do cmd_list ; done
```

`in word`部分就是展示给用户的各个选项，如果省略，则等价于`in "$@"`。当用户输入其所选择的项后，对应项的内容保存到name变量，用户输入的内容保存到REPLY变量中。

注：REPLY变量一般是序号值，但用户可以不按常理出牌，随意输入，所以REPLY保存的不一定是序号。

另外，用户做出选择后select会执行相关命令，执行完命令后会再次让用户选择。所以，应该在命令尾部使用break命令来终止select。

例如：

```shell
select fname in cat dog sheep mouse;do
  echo your choice: \"$REPLY\) $fname\"
  break
done
1) cat
2) dog
3) sheep
4) mouse
#? 3                      # 在此选择序号3
your choice: "3) sheep"   # 将输出序号3对应的内容
```

## continue、break、return、exit

```
exit [n]
退出当前shell，在脚本中应用则表示退出整个脚本。其中数值n表示退出状态码。

break [n]
退出整个循环，包括for、while、until和select语句。其中数值n表示退出的循环层次。

continue [n]
退出当前循环进入下一次循环。n表示继续执行向外退出n层的循环。默认n=1，表示继续当前层的下一循环，n=2表示继续上一层的下一循环。

return [n]
退出整个函数。n表示函数的退出状态码。
```

唯一需要注意的是return，它并非只能用于function内部，绝大多数人都有这样的误解。如果return用在function之外，但在source命令的执行过程中，则直接停止该执行操作，并返回给定状态码n(如果未给定，则为0)。如果return在function之外，且不在source的执行过程中，则这是一个错误用法。

为什么要让return单独作用于source命令？如果了解source的特性『在当前shell而非子shell执行指定脚本中的代码』的话，就能理解为什么会这样。

比如设计一个脚本，它可以在当前Shell命令行下激活几个代理相关的变量，还能注销这些代理变量：

```shell
proxy="http://127.0.0.1:8118"
function active_proxy() {
  export http_proxy=$proxy
  export https_proxy=$proxy
  export ftp_proxy=$proxy
  export no_proxy=localhost
}

case $1 in 
  set) active_proxy;;
  unset) unset http_proxy https_proxy ftp_proxy no_proxy;;
  *) return 0
esac
```

因为变量要在当前Shell下生效，所以应该使用source命令去执行脚本：

```shell
source proxy.sh set
source proxy.sh unset
```

但如果没有给脚本传递参数(比如将该脚本放在/etc/profile.d/目录下，自动调用该脚本时是不会传参的)，或者传递了其它参数，将直接停止source。但如果将上面的return改成exit，则直接退出当前Shell。

**下面是return的另一个技巧**：当在Linux系统中写下很多脚本后，很可能会将脚本进行分类和组织，一部分脚本是用于执行的，一部分是用于source的。但判断某脚本是被source还是被bash执行，不是一件简单的事。但由于`return`只能在函数内或在source的脚本内使用，这使得判断当前脚本是被source执行还是被bash直接当脚本执行变得非常容易。

下面我给出了bash脚本内判断当前脚本是否被source的两种方案：

```bash
# 方案一：直接使用return进行判断
# 如果不是source加载，则return报错
# 在子shell中处理报错信息，并让子shell返回退出状态码0
(return 0 2>/dev/null) && source_flag=1 || source_flag=0

# 方案二：使用$0和$BASH_SOURCE[0]
# 在a.sh脚本中source b.sh，那么b.sh中的：
#   $0 = a.sh， $BASH_SOURCE[0]=b.sh
[ "$0" != "$BASH_SOURCE" ] && source_flag=1 || source_flag=0
```