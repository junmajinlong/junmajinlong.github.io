---
title: 通过getopt设计shell脚本选项
p: shell/getopt.md
date: 2020-02-12 17:37:29
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# 通过getopt设计shell脚本选项

> man 1 getopt翻译：[https://www.junmajinlong.com/shell/getopt_translate](/shell/getopt_translate)

写shell脚本的时候，通过while、case、shift来设计脚本的命令行选项是一件比较麻烦的事，因为Unix命令行的选项和参数自由度很高，支持短选项和长选项，参数可能是可选的，选项顺序可能是无所谓的，等等。

bash下的getopt命令可以解析命令行的选项和参数，**将散乱、自由的命令行选项和参数进行改造，得到一个完整的、规范化的参数列表，这样再使用while、case和shift进行处理就简单的太多了**。

getopt有不同的版本，本文介绍的是它的增强版(enhanced)，相比传统的getopt(也称为兼容版本的getopt)，它提供了引号保护的能力。另外，除了不同版本的getopt，bash还有一个内置命令getopts(注意，有个尾随的字符s)，也用来解析命令行选项，但只能解析短选项。

要验证安装的getopt是增强版的还是传统版的，使用`getopt -T`判断即可。如果它什么都不输出，则是增强版，此时它的退出状态码为4。如果输出`--`，则是传统版的getopt，此时它的退出状态码为0。如果想在脚本中进行版本检查，可以参考如下代码：
```
getopt -T &>/dev/null;[ $? -ne 4 ] && { echo "not enhanced version";exit 1; }
```

<a name="blog1"></a>

## 1.命令行选项的那些常识

在学习getopt如何使用之前，必须先知道命令行的一些常识。这些，都可以通过getopt来实现，但有些实现起来可能会比较复杂。

**1.区分option、parameter、argument、option argument和non-option parament**

parameter和argument都表示参数，前者通常表示独立性的参数，后者通常表示依赖于其它实体的参数。parameter的含义更广，argument可以看作parameter的一种。

例如，定义函数时`function foo(x,y){CODE}`，函数的参数x和y称为parameter。调用函数并传递参数时，`foo(arg1,arg2)`中的arg1和arg2都是依赖于函数的，称为argument更合适，当然也可以称为更广泛的parameter。

再例如，一个命令行：
```
tar -zcf a.tar.gz /etc/pki
```
粗分的话，`-z`、`-c`、`-f`、`a.tar.gz`、`/etc/pki`都可以称为parameter。细分的话：  
- `-z -c -f`称为选项，即option  
- a.tar.gz是选项"-f"的**选项参数**(传递给选项的参数)，依赖于选项，称为argument更合适，更严格的称呼是option argument   
- /etc/pki既不属于选项，也不属于某个选项的参数，它称为**非选项类型的参数**，对应的名称为non-option parameter

本文要介绍的是getopt，所以只考虑命令行参数的情况。

**2.短选项和长选项以及它们的"潜规则"**

Linux中绝大多数命令都提供了短选项和长选项。一般来说，短选项是只使用一个`-`开头，选项部分只使用一个字符，长选项是使用两个短横线(即`--`)开头的。

例如`-a`是短选项，`--append`是长选项。

一般来说，选项的顺序是无所谓的，但并非绝对如此，有时候某些选项必须放在前面，必须放在某些选项的前面、后面。

一般来说，短选项：  
- 可以通过一个短横线"-"将多个短选项连接在一起，但如果连在一起的短选项有参数的话，则必须作为串联的最后一个字符。  

   例如`-avz`其实会被解析为`-a -v -z`，`tar -zcf a.tar.gz`串联了多个短选项，但`-f`选项有参数a.tar.gz，所以它必须作为串联选项的最后一个字符。  

- 短选项的参数可以和选项名称连在一起，也可以是用空白分隔。例如`-n 3`和`-n3`是等价的，数值3都是`-n`选项的参数值。  
- 如果某个短选项的参数是可选的，那么它的参数必须紧跟在选项名后面，不能使用空格分开。至于为什么，见下面的第3项。  

一般来说，长选项：  
- 可以使用等号或空白连接两种方式提供选项参数。例如`--file=FILE`或`--file FILE`。  
- 如果某个长选项的参数是可选的，那么它的参数必须使用"="连接。至于为什么，见下面的第3项。  
- 长选项一般可以缩写，只要不产生歧义即可。

例如，ls命令，以"a"开头的长选项有3个。
```
$ ls --help | grep -- '--a' 
  -a, --all                  do not ignore entries starting with .
  -A, --almost-all           do not list implied . and ..
      --author               with -l, print the author of each file
```
如果想要指定`--almost-all`，可以缩写为`--alm`；如果想要指定`--author`，可以缩写为`--au`。如果只缩写为`--a`，bash将给出错误提示，长选项出现歧义：
```
$ ls --a
ls: option '--a' is ambiguous; possibilities: '--all' '--author' '--almost-all'
Try 'ls --help' for more information.
```

**3.不带参数的选项、可选参数的选项和带参数的选项**

有不同类型的命令行选项，这些选项可能不需要参数，也可能参数是可选的，也可能是强制要求参数的。

前面说了，如果某个选项的参数是可选的，那么它的参数必须不能使用空格将参数和选项分开。如果使用空格分隔，则无法判断它的下一个元素是该选项的参数还是非选项类型的参数。

例如，`-c`和`--config`选项的参数是可选的，要向这两个选项提供参数，必须写成`-cFILE`、`--config=FILE`，如果写成`-c FILE`、`--config FILE`，那么命令将无法判断这个FILE是提供给选项的参数，还是非选项类型的参数。

一般来说，使用可选参数的情况非常少，至少我目前回忆不起来这样的命令(mysql的`-p`选项是一个)。

**4.使用`--`将选项(及它们的选项参数)与非选项类型参数进行分隔**

unix的命令行中，总是可以在非选项类型的参数之前加上`--`，表示选项和选项参数到此为止，后面的都是非选项类型的参数。

例如：
```
seq -w -- 3
seq -w -- 1 3
```
分别表示3和"1 3"是seq的非选项类型参数，而`--`前面的一定是选项或选项参数。

**5.命令行参数中的短横线开头的并不一定总是短选项，也可能是负数参数**

例如seq命令：
```
seq -w -5 -1 5
```
其中-5和-1都是负数非选项类型的参数。

**6.选项的依赖性和互斥性**

有些命令的选项是有依赖性和互斥性的。比如某个选项要和另一个选项一起使用，某个选项不能和另一个选项一起使用。

例如`--manage --remove`，只有在使用了`--manage`的前提下才能使用`--remove`，否则就应该报错。

**7.模式化(模块化)类型的选项**

很多unix命令都将选项进行模块化设计。例如ip命令，address模式、route模式、link模式等等。
```
ip addr OPTIONS
ip route OPTIONS
ip link OPTIONS 
ip neigh OPTIONS
```

**8.其他特性的选项**

有些命令还有比较个性化的选项。

比如head命令，`-n NUM`选项，即可以指定为`-3`，也可以指定为`-n 3`或`-n3`。

再比如有的命令支持逻辑运算，例如find命令的`-a、-o`选项。

<a name="blog2"></a>

## 2.getopt解析选项的工作机制

bash的getopt命令经常用在shell脚本内部或函数内部，用来解析脚本执行或函数执行时传递的选项、参数。

下面都以命令行为例解释getopt是如何解析参数的，但用来解析函数参数是一样的。

<a name="blog2.1"></a>

### 2.1 getopt选项

下面这个是最常用的getopt解析方式(有这个命令就够了)。如果要了解getopt更完整的语法，见`man getopt`。
```
getopt -o SHORT_OPTIONS -l LONG_OPTIONS -n "$0" -- "$@"

其中：  
-o SHORT_OPTIONS` 
--options SHORT_OPTIONS
getopt通过"-o"选项收集命令行传递的短选项和它们对应的参数。关于SHORT_OPTIONS的格式见下一小节。

-l LONG_OPTIONS  
--longoptions LONG_OPTIONS  
getopt通过"-l"选项收集命令行传递的长选项和它们对应的参数。可能从别人的脚本中经常看到"--long"，是等价的，前文已经解释过，长选项只要不产生歧义，是可以进行缩写的。关于LONG_OPTIONS的格式见下一小节。  

-n NAME  
getopt在解析命令行时，如果解析出错(例如要求给参数的选项没带参数，使用了无法解析的选项等)将会报告错误信息，getopt将使用该NAME作为报错的脚本名称。  

-- "$@" 
其中"--"表示getopt命令自身的选项到此结束，后面的元素都是要被getopt解析的命令行参数。这里使用"$@"，表示所有的命令行参数。注意，不能省略双引号。
```

<a name="blog2.2"></a>

### 2.2 getopt如何解析选项和参数

getopt使用`-o`或`-l`解析短、长选项和参数时，将会对每个解析到的选项、参数进行输出，然后不断放进一个字符串中。这个字符串的内容就是完整的、规范化的选项和参数。

getopt使用`-o`选项解析短选项时：  
- 多个短选项可以连在一起  
- 如果某个要解析的选项需要一个参数，则在选项名后面跟一个冒号  
- 如果某个要解析的选项的参数可选，则在选项名后面跟两个冒号  
- 例如，`getopt -o ab:c::`中，将解析为`-a -b arg_b -c [arg_c]`，arg_b是-b选项必须的，arg_c是-c选项可选的参数，"-a"选项无需参数  

getopt使用`-l`选项解析长选项时：  
- 可以一次性指定多个选项名称，需要使用逗号分隔它们  
- 可以多次使用-l选项，多次解析长选项  
- 如果某个要解析的选项需要一个参数，则在选项名后面跟一个冒号  
- 如果某个要解析的选项的参数可选，则在选项名后面跟两个冒号  
- 例如，`getopt -l add:,remove::,show`中，将解析为`--add arg_add --remove [arg_rem] --show`，其中arg_add是`--add`选项必须的，`--remove`选项的参数arg_rem是可选的，`--show`无需参数  

如果解析的是带参数的选项，则getopt生成的字符串中，会将选项的参数值作为该选项的下一个参数。如果解析的是可选参数的选项，如果为该选项设置了参数，则会将这个参数放在选项的下一个参数位置，如果没有为该选项设置参数，则会生成一个用引号包围的空字符串作为选项的下一个参数。

getopt**解析完选项和选项的参数后**，将解析非选项类型的参数(non-option parameter)。getopt为了让非选项类型的参数和选项、选项参数区分开，将在解析第一个非选项类型参数时加上一个`--`到字符串中，表示选项和选项参数到此结束，然后将所有的非选项类型参数放在这个`--`参数之后。

默认情况下，该加强版本的getopt会将所有参数值(包括选项参数、非选项类型的参数)使用引号进行包围，以便保护空白字符和特殊字符。如果是兼容版本的getopt，则不会用引号保护，所以会破坏参数解析。

看后面的示例就很容易理解了。

<a name="blog2.3"></a>

### 2.3 示例分析getopt的解析方式

例如在脚本test.sh中，下面的getopt的结果保存到变量parameters中，然后输出getopt解析完成后得到的完整参数列表。
```
#!/usr/bin/env bash

parameters=`getopt -o ab:c:: --long add:,remove::,show -n "$0" -- "$@"`
echo "$parameters"
```

如果还不知道这里的`-o`和`--long`解析了什么东西，请回头仔细再看一遍。

执行这个脚本，并给这个脚本传递一些选项和参数，这些脚本参数将被收集到`$@`，然后被getopt解析。
```
$ ./test.sh -a non-op_arg1 -b b_short_arg non-op_arg2 --rem --add /path --show -c non-op_arg3
 -a -b 'b_short_arg' --remove '' --add '/path' --show -c '' -- 'non-op_arg1' 'non-op_arg2' 'non-op_arg3'
```

首先可以看出，传递给脚本的参数都是无序的：  
- 长选项有：  
    - `--rem`：是--remove的缩写形式，它的参数是可选的，但没有为它传递参数  
    - `--add`：并设置了该选项的参数/path  
    - `--show`：没有任何参数  
- 短选项有：  
    - `-a`：它是无需参数的选项，所以它后面的non-op_arg1是一个非选项类型的参数  
    - `-b`：它是必须带参数的选项，所以b_short_arg是它的参数  
    - `-c`：它的参数是可选的，这里没有给它提供参数(前面解释过，要给参数可选的选项提供参数，短选项时，参数和选项名称必须连在一起)。  
- 非选项类型的参数有：  
    - non-op_arg1  
    - non-op_arg2  
    - non-op_arg3  

从getopt的输出结果中，可以看出：  
- 先解析选项和选项参数  
- 选项和选项参数是按照从左向右的方式进行解析的  
- 参数都使用引号包围  
- 那些参数可选的选项，当没有为它们提供参数时，将生成一个引号包围的空字符串参数  
- 解析完所有的选项和选项参数后，开始解析非选项类型的参数  
- 非选项类型的参数前面，会生成一个`--`字符串，它将选项(以及选项参数)与非选项类型的参数隔开了  

<a name="blog3"></a>

## 3.处理getopt解析的结果

getopt解析得到了完整、规范化的结果，当然要拿来应用。例如直接传递个函数，或者根据while、case、shift将选项、参数进行分割单独保存。

如果要进行分割，由于getopt的解析结果通常保存在一个变量中，要解析这个结果字符串，需要使用eval函数将变量的内容进行还原，一般来说会将其设置为一个位置参数(因为shift只能操作位置变量)。

一般来说，整个处理流程是这样的：
```
parameters=$(getopt -o SHORT_OPTIONS -l LONG_OPTIONS -n "$0" -- "$@")
[ $? != 0 ] && exit 1
eval set -- "$parameters"   # 将$parameters设置为位置参数
while true ; do             # 循环解析位置参数
    case "$1" in
        -a|--longa) ...;shift ;;    # 不带参数的选项-a或--longa
        -b|--longb) ...;shift 2;;   # 带参数的选项-b或--longb
        -c|--longc)                 # 参数可选的选项-c或--longc
            case "$2" in 
                "")...;shift 2;;  # 没有给可选参数
                *) ...;shift 2;;  # 给了可选参数
            esac;;
        --) ...; break ;;       # 开始解析非选项类型的参数，break后，它们都保留在$@中
        *) echo "wrong";exit 1;;
    esac
done
```

需要注意，getopt解析既可以放在脚本中解析命令行参数，也可以放在某个函数中解析函数参数。

<a name="blog4"></a>

## 4.getopt的两种扫描模式

getopt提供了两种扫描模式，只要在getopt的短选项前加上加号或负号，就能指定两种扫描模式，即`getopt -o [+-]SHORT_OPTS`。

- `+`扫描模式：只要解析完选项、选项参数，解析到第一个非选项类型的参数后，就会停止解析，它会将所有没有解析的内容都当作非选项类型参数。所以这种情况下，非选项类型的参数都必须放在尾部，而不能放在某个待解析选项的前面。这种模式在区别负数和短选项时，非常有用。  
- `-`扫描模式：会按照原始位置参数解析，并保留原始位置。这种模式一般用不上，因为破坏了getopt的优势：让选项完整、规范化。  

例如，对于命令行参数`-w -s -5 3 -2`，要将-5识别为-s的参数，3和-2为非选项类型的参数，则：
```
$ set -- -w -s -5 3 -2  # 设置位置参数
$ getopt -o +s:w -n "$0" -- "$@"
 -w -s '-5' -- '3' '-2'      # 解析结果
```

注意，上面的-5是被解析成了-s的参数，而不是选项或非选项类型的参数，因为-s选项必须要指定一个参数。

上面的3必须不能是负数，因为**getopt必须先扫描到一个正常的非选项型参数，才能将它后面的所有负数都当作非选项型参数**。至于如何将`-w -s -5 -3 -2`中的-3和-2都解析为非选项型参数，目前我也不知道。

使用`-`扫描模式：
```
$ set -- 3 -w 4 -s -5 a 3
$ getopt -o -s:w -n "$0" -- "$@"
 '3' -w '4' -s '-5' 'a' '3' --    # 解析结果
```
可以看到，上面的所有参数位置都是保持原样的，且将分隔符号`--`补在了最尾部。

<a name="blog5"></a>

## 5.如何实现命令行选项的各种个性功能

在前面[命令行选项的那些常识](#blog1)中介绍了几种有"个性"的选项功能，包括：  
- 选项依赖：例如`-a`或`--add`要依赖于`-m`或`--manage`选项  
- 选项互斥：例如`-a`或`--add`与`-r`或`--remove`是互斥的  
- 识别负数参数：例如`-w -5 -3 5`，其中-5和-3不是短选项，而是负数参数
- 模式化选项：例如`script_name MODE OPTIONS`的MODE部分，可以是manage模式(`--manage,-m`)，也可以使用add模式(`--add,-a`)  
- 选项参数替代选项：例如`head -n 3`可以替换为`head -3`  

这里介绍下用getopt解析参数后实现它们的思路。

在getopt解析完成后，假设返回结果保存到了`$parameters`变量中。

**1.选项依赖性**

这个其实很好实现，只需使用grep对`$parameters`变量进行筛选一下即可。

例如实现依赖性，只需：
```
{ echo "$parameters" | grep -E '\-\-add|\-a ' | grep -E '\-\-manage|\-m '; } &>/dev/null
[ $? -ne 0 ] && exit
```

**2.选项互斥性**

要实现互斥性，只需：
```
or_op=`echo "$parameters" | grep -Eo '\-\-add|\-a | \-\-remove|\-r ' | wc -l`
[ "$or_op" = "2" ] && exit
```

**3.识别负数参数**

前面解释过，getopt提供了两种扫描模式，只要使用`+`扫描模式，就能轻松区别负数参数和短选项。

**4.模式化选项**

一般来说，模式化选项都是命令行的第一个参数。所以，只需将`$parameter`中`--`后面的第一个非选项类型的参数提取出来，就是所谓的模式了。当然，还得对这个参数进行一些判断，避免它不是模式参数。

例如，要提供addr、show、route三种模式，那么其它的非选项类型参数值都不应该是模式参数。
```
eval set -- "$parameters"
while true ; do
    case "$1" in
            ...
        --) 
            shift
            [ "$x" = "addr" -o "$x" = "route" -o "$x" = "show" ] && MODE=$1
            shift
            break ;;
        *) echo "wrong";exit 1;;
    esac
done
```

**5.选项参数替代选项**

就以`-n3`和`-3`为例，它的通用格式是`-n NUM`和`-NUM`。这个并不好实现，我能想到的方法是将这个`-NUM`先从`$@`中筛选出来，然后赋值。
```
NUM=`echo "$@" | grep -Eo "\-[0-9]+"`
ARGS=`echo "$@" | sed -nr 's!(.*)-[0-9]+(.*)!\1\2!'p`
eval set -- "$ARGS"
```

<a name="blog6"></a>

## 6.使用getopt设计shell脚本选项示例

这里提供一个和seq命令功能相同的脚本seq.sh，然后设计这个脚本的选项。

先看一下seq命令的各个选项说明：
```
seq [OPTION]... LAST                  # 语法1
seq [OPTION]... FIRST LAST            # 语法2
seq [OPTION]... FIRST INCREMENT LAST  # 语法3

选项：
-s, --separator=STRING
使用指定的STRING分隔各数值，默认值为"\n"u

-w, --equal-width
使用0填充在前缀使所有数值长度相同

--help
显示帮助信息并退出

--version
输出版本信息并退出
```

以下是脚本内容：和seq相比，只有两个问题：第一个起点数值FIRST不能为负数；不支持小数功能。其它功能完全相同
```
#!/usr/bin/env bash
###########################################################
#  author     : 骏马金龙                                   #
#  blog       : http://www.cnblogs.com/f-ck-need-u/       #
###########################################################

usage(){
cat <<'EOF'
Usage: $0 [OPTION]... LAST
  or:  $0 [OPTION]... FIRST LAST
  or:  $0 [OPTION]... FIRST INCREMENT LAST
EOF
}

# getopt的版本是增强版吗
getopt -T &>/dev/null;[ $? -ne 4 ] && { echo "not enhanced version";exit 1; }

# 参数解析
parameters=`getopt -o +s:w --long separator:,equal-width,help,version -n "$0" -- "$@"`
[ $? -ne 0 ] && { echo "Try '$0 --help' for more information."; exit 1; }

eval set -- "$parameters"

while true;do
    case "$1" in
        -w|--equal-width) ZERO_PAD="true"; shift ;;
        -s|--separator) SEPARATOR=$2; shift 2 ;;
        --version) echo "$0 version V1.0"; exit ;;
        --help) usage;exit ;;
        --)
            shift
            FIRST=$1
            INCREMENT=$2
            LAST=$3
            break ;;
        *) usage;exit 1;;
    esac
done


# 用于生成序列数
function seq_func(){

    # 是否要使用printf填充0位？
    [ "x$1" = "xtrue" ] && zero_pad="true" && shift
    
    # 设置first、step、last
    if [ $# -eq 1 ];then
        first=1
        step=1
        last=$1
    elif [ $# -eq 2 ];then
        first=$1
        step=1
        last=$2
    elif [ $# -eq 3 ]; then
        first=$1
        step=$2
        last=$3
    else
        echo "$FUNCNAME: ARGS wrong..."
        exit 1
    fi
    
    # 最后一个要输出的元素及其长度，决定要填充多少个0
    last_output=$[ last - ( last-first ) % step ]
    zero_pad_len=`[ ${#last_output} -gt ${#first} ] && echo ${#last_output} || echo ${#first}`

    # 生成序列数
    if [ "x$zero_pad" = "xtrue" ];then
        # 填充0
        if [ $step -gt 0 ];then
            # 递增，填充0
            for((i=$first;i<=$last;i+=$step)){
                [ $last_output -eq $i ] && { printf "%0${zero_pad_len}i\n" "$i";return; }
                printf "%0${zero_pad_len}i " $i
            }
        else
            # 递减，填充0
            for((i=$first;i>=$last;i+=$step)){
                [ $last_output -eq $i ] && { printf "%0${zero_pad_len}i\n" "$i";return; }
                printf "%0${zero_pad_len}i " $i
            }
        fi
    else
        # 不填充0
        if [ $step -gt 0 ];then
            # 递增，不填充0
            for((i=$first;i<=$last;i+=$step)){
                [ $last_output -eq $i ] && { printf "%i\n" "$i";return; }
                printf "%i " $i
            }
        else
            # 递减，不填充0
            for((i=$first;i>=$last;i+=$step)){
                [ $last_output -eq $i ] && { printf "%i\n" "$i";return; }
                printf "%i " $i
            }
        fi
    fi
}

# 指定输出分隔符
: ${SEPARATOR="\n"}

# 输出结果
seq_func $ZERO_PAD $FIRST $INCREMENT $LAST | tr " " "$SEPARATOR"
```

上面解析选项的脚本缺陷在于无法解析FIRST为负数的情况，例如`./seq.sh -w -5 3`将报错。但可以写为标准的`./seq.sh -w -- -5 -3`语法。
