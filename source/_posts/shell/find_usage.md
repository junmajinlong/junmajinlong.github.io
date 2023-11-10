---
title: Linux find命令常用用法示例
p: shell/find_usage.md
date: 2019-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# Linux find命令常用用法示例

在此处只给出find的基本用法示例，都是平时我个人非常常用的搜索功能。如果有不理解的部分，则看后面的[find运行机制详解](/shell/find_intermediate)对于理论的说明，也建议在看完这些基本示例后阅读一遍理论说明，它是本人翻译自find的man文档并加上了个人的理解。另外，在该理论说明结束后，还有find深入用法示例和分析。

## (1). 最基础的打印操作

find命令默认接的命令是`-print`，它默认以`\n`将找到的文件分隔。可以使用`-print0`来使用`\0`分隔，这样就不会分行了。但是一定要注意，`-print0`针对的是`\n`转`\0`，如果查找的文件名本身就含有空格，则find后`-print0`仍然会显示空格文件。所以`-print0`实现的是`\n`转`\0`的标记，可以使用其他工具将`\0`标记替换掉，如xargs，tr等。

```shell
$ mkdir /tmp/a
$ touch /tmp/a/{1..5}.log
$ find /tmp/a   # 等价于find /tmp/a -print，表示搜索/tmp/a目录
/tmp/a
/tmp/a/4.log
/tmp/a/2.log
/tmp/a/5.log
/tmp/a/1.log
/tmp/a/3.log

$ find /tmp/a -print0    
/tmp/a/tmp/a/4.log/tmp/a/2.log/tmp/a/5.log/tmp/a/1.log/tmp/a/3.log
```

## (2). 文件名搜索

常用的两个是`-name`和`-path`。

`-name`可以对文件的basename进行匹配，`-path`可以对文件的dirname+basename。查找的文件名最好使用引号包围，可以配合通配符进行查找。

```shell
$ find /tmp -name "*.log"
/tmp/screen.log
/tmp/x.log
/tmp/timing.log
/tmp/a/4.log
/tmp/a/2.log
/tmp/a/5.log
/tmp/a/1.log
/tmp/a/3.log
/tmp/b.log
```

但不能在-name的模式中使用"/"，除非文件名中包含了字符"/"，否则将匹配不到任何东西，因为-name只对basename进行匹配。例如，想要匹配/tmp目录下某包含字符a的目录下的log文件。

```shell
$ find /tmp -name '*a*/*.log'
find: warning: Unix filenames usually don't contain 
slashes (though pathnames do). That means that 
'-name ‘*a*/*.log’' will probably evaluate to false
all the time on this system.  You might find the 
'-wholename' test more useful, or perhaps '-samefile'.
Alternatively, if you are using GNU grep, you could
use 'find ... -print0 | grep -FzZ ‘*a*/*.log’'.
```

所以想要在指定目录下搜索某目录中的某文件，应该使用-path而不是-name。

```shell
$ find /tmp -path '*a*/*.log'
/tmp/abc/axyz.log
```

注意，配合通配符[]时应该注意是基于字符顺序的，大小写字母的顺序是a-z --> A-Z，指定[a-z]表示小写字母a-z，同理[A-Z]，而[a-zA-Z]和[a-Z]都表示所有大小写字母。当然还可以指定[a-A]表示a-z外加一个A。

字母的处理顺序较容易理解，关于数字的处理方法，见下面的示例。

```shell
$ ls
11.sh  1.sh  22.sh  2.sh  3.sh

$ find -name "[1-2].sh"
./2.sh
./1.sh

$ find -name "[1-23].sh"
./2.sh
./3.sh
./1.sh

$ touch 0.sh
$ find -name "[1-20].sh"
./2.sh
./0.sh
./1.sh

$ find -name "[1-22-3].sh"
./2.sh
./3.sh
./1.sh
```

从上面结果可以看出，其实[]只能匹配单个字符，[0-9]表示0-9的数字，[1-20]表示[1-2]外加一个0，[1-23]表示[1-2]外加一个3，[1-22-3]表示[1-2]或[2-3]，迷惑点就是看上去是大于10的整数，其实是两个或者更多的单个数字组合体。也可以用这种方法表示多种匹配：[1-2,2-3]。

![](/img/referer.jpg)


## (3). 根据文件类型搜索：-type

一般需要搜索的文件类型就只有普通文件(f)，目录(d)，链接文件(l)。

例如，搜索普通文件类的文件，且名称为a开头的sh文件。

```shell
$ find /tmp -type f -name "a*.sh"
```

搜索目录类文件，且目录名以a开头。

```shell
$ find /tmp -type d -name "a*"
```

## (4). 根据文件的时间戳搜索

最基础的时间戳包括：-atime/-mtime/-ctime。

例如搜索/tmp下3天内修改过内容的sh文件，因为是文件内容，所以不考虑搜索目录。

```shell
$ find /tmp -type f -mtime -3 -name "*.sh"
```

## (5). 根据文件大小搜索：-size

例如搜索/tmp下大于100K的sh文件。

```shell
$ find /tmp -type f -size +100k -name '*.sh'
```

## (6). 根据权限搜索：-perm

例如搜索/tmp下所有者具有可读可写可执行权限的sh文件。

```shell
$ find /tmp -type f -perm -0700 -name '*.sh'
```

## (7). 搜索空文件

空文件可以是没有任何内容的普通文件，也可以是没有任何内容的目录。

例如搜索目录中没有文件的空目录。

```shell
$ find /tmp -type d -empty
```

## (8). 搜索到文件后并删除

例如搜索到/tmp下的".tmp"文件然后删除。

```shell
$ find /tmp -type f -name "*.tmp" -exec rm -rf  '{}'  \;
```

![](/img/referer.jpg)

## (9). 搜索指定日期范围的文件

例如搜索/test下2017-06-03到2017-06-06之间修改过的文件。

```shell
$ find /test -type f -newermt 2017-06-03 -a ! -newermt 2017-06-06
```

或者，创建两个临时文件，并用touch修改这两个文件的修改时间，然后`find -newer`去参照这两个文件。

```shell
$ touch -m -d 2017-06-03 tmp1.txt
$ touch -m -d 2017-06-06 tmp2.txt
$ find /test -type f -newer tmp1.txt -a ! -newer tmp2.txt
```

不过这样会把tmp2.txt也搜索出来，因为newer搜索的是比xxx文件更新，取反则表示更旧或时间相同。

## (10). 并行加速搜索

有时候，想要搜索的内容并不知道在哪里，这时我们会从根"/"开始搜索，这样的搜索速度可能会稍微长那么一点点。为了加速搜索，使用xargs的并行功能。例如，搜索"/"下的所有"Find.pm"结尾的文件：

```shell
ls --hide proc / | xargs -i -P 0 find /{} -type f -name "*Find.pm"
```

可以使用time命令看看cpu利用率：

```shell
$ /usr/bin/time bash -c 'ls --hide proc / \
    | xargs -i -P 0 find /{} -type f -name "*Find.pm" \
    | sort'
/perlapp/perl-5.26.2/cpan/Pod-Parser/lib/Pod/Find.pm
/perlapp/perl-5.26.2/ext/File-Find/lib/File/Find.pm
/usr/share/perl5/vendor_perl/Pod/Find.pm
/usr/share/perl5/File/Find.pm
0.04user 0.25system 0:00.19elapsed 149%CPU (0avgtext+0avgdata 5492maxresident)k
1.0inputs+0outputs (0major+12685minor)pagefaults 0swaps
```

## (11). 获取文件绝对路径

当find结合管道，而管道后的命令很可能想要获取到搜索到的文件的绝对路径，或者说是全路径。而问题是，当find的搜索路径是相对路径时，搜索出来的显示结果也是以相对路径显示的。

```shell
$ mkdir /tmp/test
$ touch /tmp/test/{a,b,c}.png
$ find .
.
./a.png
./b.png
./c.png
```

想要获取全路径，方式有很多种：

```shell
# 搜索前先pwd

$ find $(pwd)
/tmp/test
/tmp/test/a.png
/tmp/test/b.png
/tmp/test/c.png
```

```shell
# 或使用$PWD环境变量
$ find $PWD
/tmp/test
/tmp/test/a.png
/tmp/test/b.png
/tmp/test/c.png
```

```shell
# 执行readlink，它不仅解析软链接，也可以使用-f选项解析普通文件
$ find . -exec readlink -f {} \;
/tmp/test
/tmp/test/a.png
/tmp/test/b.png
/tmp/test/c.png
```

```shell
# 使用bash的波浪号扩展 `~+`
$ find ~+
/tmp/test
/tmp/test/a.png
/tmp/test/b.png
/tmp/test/c.png
```

![](/img/referer.jpg)

## (12). 获取文件名部分(basename)

find的`-printf`选项有很多修饰符功能，对于处理路径方面的修饰符有`%f、%p、%P`，其中`%f`是获取basename（去除所有路径前缀），`%p`是获取路径自身，一般用不上，`%P`是获取除了find搜索路径的剩余部分。

首先，想要获取basename，建议使用`%f`。

```shell
$ mkdir /tmp/test/test1
$ touch /tmp/test/test1/{x,y,z}.png
$ find /tmp/test -printf "%f\n"
test
a.png
b.png
c.png
test1
x.png
y.png
z.png
```

再看使用`%P`的效果，结果是去掉了find搜索路径`/tmp/test`部分。当搜索路径只有一层(即没有子目录)时，它也可以用来获取basename。

```shell
$ find /tmp/test -printf "%P\n"

a.png
b.png
c.png
test1
test1/x.png
test1/y.png
test1/z.png
```

## (13). 从结果中排除目录自身

find搜索目录时，总是会将搜索路径自身也包含到搜索结果中。想办法排除它是必须的。

排除的方法是，加上一个`-path`选项并取反，`-path`的参数和find的搜索路径参数必须一致。

```shell
$ find /tmp/test ! -path /tmp/test
/tmp/test/a.png
/tmp/test/b.png
/tmp/test/c.png
/tmp/test/test1
/tmp/test/test1/x.png
/tmp/test/test1/y.png
/tmp/test/test1/z.png
```

```shell
$ find . ! -path .
./a.png
./b.png
./c.png
./test1
./test1/x.png
./test1/y.png
./test1/z.png
```

另一种更好的方法是加上`-atime +0`选项或者加上`-mindepth 1`：
```shell
$ find . -atime +0  # 或者 find . -mindepth 1
./a.png
./b.png
./c.png
./test1
./test1/x.png
./test1/y.png
./test1/z.png
```