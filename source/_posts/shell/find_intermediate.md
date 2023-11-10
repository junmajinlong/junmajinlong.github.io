---
title: Linux find命令运行机制详解
p: shell/find_intermediate.md
date: 2019-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# Linux find命令运行机制详解

find用于搜索文件或目录，功能非常强大。该工具是findutils包提供的，该包中还包括一个老版本的oldfind工具，以及另一个非常强大的xargs命令，在搜索文件时，如果还要对搜索的文件进行后续的处理，一般都会结合xargs来实现。但本文并不过多涉及xargs，如果要了解xargs用法，见我的另一篇关于xargs的总结[xargs的原理剖析及用法详解](/shell/xargs)，目前在网上暂时还没找到比这篇文档更详细的xargs说明。

find搜索是从磁盘进行指定目录开始扫描，而不是从数据库搜索。


# find基础示例

因为内容有点多，所以放进单独一篇文章中了。参见[find常用用法示例](/shell/find_usage)，建议在阅读本文之前一看，相信其中一些用法不会让你失望的。



# find理论部分

```
find [path...] [expression_list]
```

expression分为三种：options、test、action。对于多个表达式，find是**从左向右处理的**，所以表达式的前后顺序不同会造成不同的搜索性能差距。

*find首先对整个命令行进行语法解析，并应用给定的options，然后定位到搜索路径path下开始对路径下的文件或子目录进行表达式评估或测试，评估或测试的过程是按照表达式的顺序从左向右进行的(此处不考虑操作符的影响)，如果最终表达式的表达式评估为true，则输出(默认)该文件的全路径名*。

对于find来说，一个非常重要的概念：find的搜索机制是根据表达式返回的true/false决定的，每搜索一次都判断一次是否能确定最终评估结果为true，只有评估的最终结果为true才算是找到，并切入到下一个搜索点。

## expression operators

操作符控制表达式运算方式。确切的说，是控制expression中的options/tests/actions的运算方式，无论是options、tests还是actions，它们都可以给定多个，例如`find /tmp -type f -name "*.log" -exec ls '{}' \; -print`，该find中给定了两个test，两个action，它们之间从前向后按顺序进行评估，所以如果想要改变运算逻辑，需要使用操作符来控制。

注意，理解and和or的评估方式非常重要，很可能写在and或or后面的表达式不起作用，而导致跟想象中的结果不一样。

![](/img/referer.jpg)

下面的操作符优先级从高到低。

| expression_operators | 含义                                                         |
| -------------------- | ------------------------------------------------------------ |
| ( expr )             | 优先级最高。为防止括号被shell解释(进入子shell)，所以需要转义，即\(...\) |
| ! expr               | 对expr的true和false结果取反。同样需要使用引号包围            |
| -not expr            | 等价于"! expr"                                               |
| expr1 expr2          | 等同于and操作符                                              |
| expr1 -a expr2       | 等同于and操作符                                              |
| expr1 -and expr2     | 首先要求expr1为true，然后expr2以expr1搜索的结果为基础继续检测，然后再返回检测值为true的文件。因为expr2是以expr1结果为基础的，所以如果expr1返回false，则expr2直接被忽略而不会进行任何操作 |
| expr1 -o expr2       | 等同于or操作符                                               |
| expr1 -or expr2      | 只有expr1为假时才评估expr2                                   |
| expr1 , expr2        | 逗号操作符表示列表的意思，expr1和expr2都会被评估，但expr1的true或false是被无视的，只有expr2的结果才是最终状态值 |

关于and和or操作符，一定要明确的是and后的表达式操作的对象是前面表达式的结果，而or操作符的对象则是前面评估后为假时文件。

例如：

```shell
find /tmp -type f -name "*.log"
```

它是一个and操作符，-name表达式是在-type筛选的结果基础上再匹配文件名的。但如果是：

```shell
find /tmp -type f -o -name "*.log"
```

则`-type f`返回真的文件直接执行它后面默认的-print，而不满足-type f的才会去被-name评估，所以返回结果中即有任意普通文件，也有任意log文件，但两者同名的文件只返回一次。

总之，and和or对于find的影响比较大，后面会专门用一节[彻底搞懂operator和action](#operator_action)来详细解释。另外，在后文find深入用法示例中还有一个关于"[忽略目录](#prune)"的例子，它很好的解释了操作符的行为。

## expression-options

options总是返回true。除了"-daystart"，options会影响所有指定的test表达式部分，哪怕是test部分写在options的前面。这是因为options是在命令行被解析完后立即处理的，而test是在检测到文件后才处理的。对于"-daystart"这个选项，它们仅仅影响写在它们后面的test部分，因此，建议将任何options部分写在expression的最前面，若不如此，会给出一个警告信息。

```
---------------------------------------------------------
-daystart：指定以每天的开始(凌晨零点)计算关于天的时间，用于改变
         : 时间类(-amin,-atime,-cmin,-ctime,-mmin和-mtime)
         ：的计算方式。默认天的计算是从24小时前计算的。例如，当前
         : 时间为5月3日17:00，要求搜索出2天内修改过的文件，默认
         ：搜索文件的起点是5月1日17:00，如果使用-daystart，则搜
         : 索文件的起点是是5月1日00:00。
         ：注意，该选项只会影响写在它后面的test表达式。
---------------------------------------------------------
-depth   ：搜索到目录时，先处理目录中文件(子目录)，再处理目录本身。
         : 对于"-delete"这个action，它隐含"-depth"选项。
---------------------------------------------------------
-maxdepth levels：指定tests和actions作用的最大目录深度，只能为
                : 非负整数。可以简单理解为目录搜索深度，但并非如
                : 此。当前path目录的层次为1，所以若指定
                ：-maxdepth 0将得不到任何结果。
---------------------------------------------------------
-mindepth levels：tests和actions不会应用于小于指定深度的目录，
                : "-mindepth 1"表示应用于所有的文件。
---------------------------------------------------------
-ignore_readdir_race：当无法用stat检测文件信息时(如无权限)会给
                    : 出错误信息，如要忽略该信息，可使用该选项。
---------------------------------------------------------
-warn：忽略警告信息。
```

![](/img/referer.jpg)

## expression-tests

find解析完命令行语法之后，开始搜索文件，在搜索过程中，每次检测到的文件都会被test expression进行测试，符合条件的将被保留。

数值部分可以设置为以下3种模式：n可以是小数。
```
+n：大于n
-n：小于n
 n：精确的等于n
```

对于文件大小而言，文件的大小是精确的，指定100KB，则比对的值必定是100KB。此时(+ -)n和字面意思是一样的。

![](/img/shell/733013-20170613011411275-598720664.jpg)

但对于时间而言，时间是有时间段的，例如指定前第四天，第四天也整整占用了一天，所以(+ -)n和文件大小的计算方法是不一样的。

![](/img/shell/733013-20170613011414478-1962876114.jpg)

find在计算以天数为单位的时间时，默认会转换为24小时制，除非同时指定了`-daystart`这个选项，这在前面已经解释过了。例如当前时间为5月3号17:00，那么计算`atime +1`的时候，真正计算的是`24*1=24`小时之前，又因为前48小时到前24小时之间都属于前一天内，所以其搜索的一天前是5月1号17:00以前被访问过，若指定了`-daystart`，则计算的是5月2号00:00之前被访问过。(所以，+n -n的计算，用汉语来描述是非常容易理解的，+n表示的是n天前，n是前n天(或者说是前第n天)，-n是n天内)

具体的选项如下：

```
-type X：根据文件类型来搜索 
      b：块设备文件
      c：字符设备文件
      d：目录
      p：命名管道文件(FIFO文件)
      f：普通文件
      l：符号链接文件，即软链接文件
      s：套接字文件(socket)
```

【文件大小或内容类测试条件】

```
-size n[cwbkMG]：根据文件大小来搜索，可以是(+ -)n，单位可以是：
            · b：512字节的(默认单位)
            · c：1字节的
            · w：2字节
            · k：1024字节
            · M：1024k
            · G：1024M
empty：空文件，对于目录来说，则是空目录
```

【文件名或路径名匹配类测试条件】

`-name pattern`  
文件的basename(不包括其前导目录的纯文件名)能被通配符模式的pattern匹配到。由于前导目录
被移除，所以find对包含"/"的pattern是绝对不可能匹配到内容的，例如`-name a/b`的结果
一定是空且会给出警告信息，若要匹配这样的文件，可考虑使用"-path"或"-name b"。需注意，在
find中的通配元字符"\*"、"?"和"[]"是能够匹配以点开头的文件的，之所以要在此说明这一点，是
因为在bash中，这些通配元字符默认是无法匹配"."开头的文件的，例如"cp ~/\* /tmp"不会把隐藏文
件也拷贝走。若要忽略一个目录及其内的文件，可配合"-prune"，它会跳过整个目录而不对此目录
做任何检查。注意pattern要用引号包围防止被shell解释  

`-iname pattern`   
不区分大小的"-name"  

`-path pattern`  
文件名能被通配符模式的pattern匹配到。此模式下，通配元字符"\*"、"?"和"[]"不认为字符"/"或"."是特殊字符，也就是说这两个字符也在通配范围内，所以能匹配这两个字符。  
例如`find . -path "./sr\*sc`可以匹配到名为"./src/misc"的目录。find会将"-path"的pattern与文件的dirname和basename的结合体进行比较，由于dirname和basename的结合体不包含尾随"/"，所以若pattern中指定了尾随"/"是不可能匹配到任何东西的。例如：`find /tmp -path "/tmp/ab*/"`，实际上它会给出警告信息，提示pattern以"/"结尾。使用"-path"的时候，一定要注意"-path"后指定的路径起点属于`find path expression`的path内，例如：`find /bar -path /foo/bar/myfile -print`不可能匹配到任何东西。若要忽略目录及其内文件，可配合"-prune"，它会跳过整个目录而不对此目录做检查。如`find . -path ./src/emacs -prune -o -print`将跳过对目录"./src/emacs"的检查。注意pattern要用引号包围防止被shell解释

`-ipath pattern`  
不区分大小写的"-path"  

`-regex pattern`  
文件名能被正则表达式pattern匹配到的文件。正则匹配会匹配整个路径，例如要匹配文件名为"./fubar3"的文件，可以使用`.*bar.`或`.*b.*3`，但不能是`f.*r3`，默认find使用的正则类型是Emacs正则，但可以使用-regextype来改变正则类型

`-iregex pattern`   
不区分大小写的"-regex"

【权限类测试条件】

`-perm mode`  
精确匹配给定权限的文件。"-perm g=w"将只匹配权限为0020的文件。当然，也可以写成三位数字的权限模式

`-perm -mode`  
匹配完全包含给定权限的文件，这是最可能用上的权限匹配方式。例如给定的权限"-0766"，则只能匹配"N767"、"N777"和"N776"这几种权限的文件，如果使用字符模式的权限，则必须指定u/g/o/a，例如"-perm -u+x,a+r"表示至少所有人都有读权限，且所有者有执行权限的文件


`-perm /mode`  
匹配任意给定权限位的权限，例如"-perm /640"可以匹配出600,040,700,740等等，只要文件权限的任意位能包含给定权限的任意一位就满足

`-perm +mode`  
由于某些原因，此匹配模式被替换为"-perm /mode"，所以此模式已经废弃

`-executable`  
具有可执行权限的文件。它会考虑acl等的特殊权限，只要是可执行就满足。它会忽略掉-perm的测试

`-readable`   
具有可读权限的文件。它会考虑acl等的特殊权限，只要是可读就满足。它会忽略掉-perm的测试

`-writable`   
具有可写权限的文件。它会考虑acl等的特殊权限，只要是可写就满足。它会忽略掉-perm的测试(不是writeable)

【所有者所属组类测试条件】
```
-gid n      ：gid为n的文件
-group gname：组名为gname的文件
-uid n      ：文件的所有者的uid为n
-user uname ：文件的所有者为uname，也可以指定uid
-nogroup    ：匹配那些所属组为数字格式的gid，
            : 且此gid没有对应组名的文件
-nouser     ：匹配那些所有者为数字格式的uid，
            : 且此uid没有对应用户名的文件
```

【时间戳类测试条件】
```
-anewer file：atime比mtime更接近现在的文件。
            : 即，文件修改过之后被访问过
-cnewer file：ctime比mtime更接近现在的文件
-newer  file：比给定文件的mtime更接近现在的文件。
-newer[acm]t TIME：atime/ctime/mtime比时间戳TIME更新的文件
-amin  n：文件的atime在范围n分钟内改变过。
        : 注意，n可以是(+-)n，如-amin +3表示在3分钟以前
-cmin  n：文件的ctime在范围n分钟内改变过
-mmin  n：文件的mtime在范围n分钟内改变过
-atime n：文件的atime在范围24*n小时内改变过
-ctime n：文件的ctime在范围24*n小时内改变过
-mtime n：文件的mtime在范围24*n小时内改变过
-used  n：最近一次ctime改变n天范围内，atime改变过的文件，
        : 即atime比ctime晚n天的文件，可以是(+-)n
```

【软硬链接类测试条件】

```
-samefile name：找出指定文件同indoe的文件，即其硬链接文件
-inum  n：inode号为n的文件，可用来找出硬链接文件。
        : 但使用"-samefile"比此方式更方便
-links n：有n个软链接的文件
```

【杂项测试】

```
-false：总是返回false，这选项有奇用
-true ：总是返回true，这选项有奇用
```

![](/img/referer.jpg)

## expression-actions

actions部分一般都是执行某些命令，或实现某些功能。这部分是find的command line部分。

`-delete`  
删除文件，如果删除成功则返回true，如果删除失败，将给出错误信息。"-delete"动作隐含"-depth"

`-exec command ;`  
注意有个分号";"结尾，该action是用于执行给定的命令。如果命令的返回状态码为0则该action返回true。command后面的所有内容都被当作command的参数，直到分号";"为止，其中参数部分使用字符串"{}"时，它表示find找到的文件名，即在执行命令时，"{}"会被逐一替换为find到的文件名，"{}"可以出现在参数中的任何位置，只要出现，它都会被文件名替换。注意，分号";"需要转义，即"\;"，如有需要，可以将"{}"用引号包围起来

`-ok command ;`  
类似于-exec，但在执行命令前会交互式进行询问，如果不同意，则不执行命令并返回false，如果同意，则执行命令，但执行的命令是从/dev/null读取输入的

`-print`  
总是返回true。这是默认的action，输出搜索到文件的全路径名，并尾随换行符"\n"。由于在使用"-print"时所有的结果都有换行符，如果直接将结果通过管道传递给管道右边的程序，应该要考虑到这一点：文件名中有空白字符(换行符、制表符、空格)将会被右边程序误分解，如文件"ab c.txt"将被认为是ab和c.txt两个文件，如不想被此分解影响，可考虑使用"-print0"替代"-print"将所有换行符替换为"\0"

`-printf`  
输出格式太多，所以具体用法见man文档

`-print0`  
总是返回true。输出搜索到文件的全路径名，并尾随空字符"\0"。由于尾随的是空字符，所以管道传递给右边的程序，然后只需对这个空字符进行识别分隔就能保证文件名不会因为其中的空白字符被误分解

`-prune `  
不进入目录，所以可用于忽略目录，但不会忽略普通文件。没有给定-depth时，总是返回true，如果给定-depth，则直接返回false，所以-delete(隐含了-depth)是不能和-prune一起使用的

`-ls`  
总是返回true。将找到的文件以"ls -dils"的格式打印出来，其中文件的size部分以KB为单位

一定要注意，**action是可以写在tests表达式前面的，它并不一定是在test表达式之后执行**。

例如：

```shell
$ find /tmp -print -type f -name "*.txt" 
/tmp 
/tmp/userfile 
/tmp/passwdfile 
/tmp/testdir 
/tmp/testdir/a.log 
/tmp/a.txt 
/tmp/time.sh 
/tmp/b.txt 
/tmp/abc 
/tmp/abc/axyz.log 
/tmp/about.html
```

它将输出/tmp下所有文件，而不是输出满足`-type f -name "*.txt"`的文件，因为-print总是返回true，它是第一个表达式，所以直接输出整个目录，后面的表达式`-type f -name "*.txt"`虽然在-print之后也被评估了，但是却没有对应的action，所以它们的匹配行为并没显示出来。如果在表达式`-type f -name "*.txt"`之后加上另外一个action，那么这个action将只针对匹配到的文件进行操作。

```shell
[root@server2 tmp]# find /tmp -print -type f -name "*.txt" -ls
/tmp
/tmp/userfile
/tmp/passwdfile
/tmp/testdir
/tmp/testdir/a.log
/tmp/a.txt
69617151 4 -rw-r--r-- 1 root root 1758 Jun 8 03:50 /tmp/a.txt
/tmp/time.sh
/tmp/b.txt
101409594 4 -rw-r--r-- 1 root root 68 Jun 8 08:02 /tmp/b.txt
/tmp/abc
/tmp/abc/axyz.log
/tmp/about.html
```

可以看到结果中，只有满足条件的两个文件才执行了ls命令。

还可以写多个action，写在哪里要注意它们的执行顺序。其实，不推荐使用除了"-print"、"-print0"和"-prune"外其他所有的action，如有需要，应该配合xargs命令来实现目的。

![](/img/referer.jpg)

<a name="blog1.3"></a>
<a name="operator_action"></a>

# 彻底搞懂operator和action的关系

find的逻辑是很严格的，但是如果没有深究过，可能会出现不容易理解的偏差。

下面，我对and和or操作符，并以-print这个action为例来详细说明它们的关系。在测试的时候，find的"-D debug"选项非常好用，它能让我们知道find是按照什么逻辑处理各个选项的。debug有几种类型，这里选择"-D rates"这种调试类型。

首先要明确一个结论，**and的优先级高于or**。

```
expr1 -a expr2：-a的逻辑是只有当expr1为真时才评估expr2
expr1 -o expr2：-o的逻辑是只有当expr1为假时才评估expr2
expr1 -o expr2 -a expr3：and的优先级高于or，
                       : 等价于expr1 -o (expr2 -a expr3)
```

例如，` -name "*.log" -o -name "*.txt"`，搜索到1.txt文件时，`*.log`评估结果为假，于是评估`*.txt`，评估为真，于是整个-o逻辑返回真。

第二个结论：**如果find评估完所有表达式后发现没有action(-prune这个action除外)，则在最末尾加上-print作为默认的action**。注意，这个默认的action是在评估完所有表达式后加上的。且还需注意，如果只有-prune这个action，它还是会补上-print。

以下是准备数据：

```shell
rm -rf /tmp/*
touch {1,2,3,4}.txt {a,b}.log
```

## 测试"and"操作符

现在先测试"expr1 -a expr2"中的"-a"。

```shell
[root@xuexi tmp]# find /tmp -type f -a -name "*.log"
/tmp/a.log
/tmp/b.log
```

用"-D rates"来看调试结果：

```shell
[root@xuexi tmp]# find -D rates /tmp -type f -a -name "*.log"
/tmp/a.log
/tmp/b.log
Predicate success rates after completion:
 ( -name *.log [0.8] [2/12=0.166667] -a [0.95] [2/12=0.166667] [need type] -type f [0.95] [2/2=1]  ) -a [0.76] [2/12=0.166667] -print [1] [2/2=1]
```

将最后一行进行简化，删掉所有中括号中的内容，得到的结果是：

```
( -name *.log -a -type f ) -a -print
```

所以，上面的语句执行逻辑是：

```shell
find /tmp ( -name *.log -a -type f ) -a -print
```

可见，find自动补上了-print这个默认的action，且逻辑是评估-name，在-name为真的基础上再评估-type，最后在-type为真的基础上评估-print，也就是将log后缀且是普通文件的文件打印出来。

这里的-name为什么被修改到了-type的前面去？这是因为find自带优化功能，它会评估各个表达式的开销，将开销小的表达式放在前面，以便优化搜索。但要注意的是，优化后的表达式和我们给出的find命令行的结果不会有任何出入，它们在逻辑上是相同的，只是更换了一些表达式的前后位置。

然后，按照前面的结论，猜测下面3个find语句的逻辑：

```shell
find /tmp -type f -a -name "*.log" -print
find /tmp -type f -print -a -name "*.log"
find /tmp -type f -print -a -name "*.log" -print
```

上面3个find都给出了action，所以find不会再补上默认的action：-print。

所以，上面3个语句分别等价于下面3个语句：

```shell
find /tmp ( -name *.log -a -type f ) -a -print
find /tmp ( -type f -a -print ) -a -name *.log
find /tmp (( -type f -a -print ) -a -name *.log ) -a -print
```

唯一需要注意的是，有时候某些expr后面没有action，这部分的expr将不会做任何处理，正如上面第二个语句，`-name *.log`后并不会补上-pinrt，所以对于find来说，`-a -name *.log`这个条件完全是多余的表达式。

以下是上述3个find命令的输出结果以及分析。

```shell
[root@xuexi tmp]# find /tmp -type f -a -name "*.log" -print
/tmp/a.log
/tmp/b.log
```

-type f评估后得到所有普通文件，-name再在普通文件的基础上筛选文件名后缀为".log"的文件，最后评估-print输出".log"文件。

```shell
[root@xuexi tmp]# find /tmp -type f -print -a -name "*.log"
/tmp/a.log
/tmp/1.txt
/tmp/2.txt
/tmp/3.txt
/tmp/4.txt
/tmp/b.log
```

-type f评估后得到所有普通文件，然后在普通文件的基础上评估-print，它直接将所有普通文件都输出，再在输出的文件的基础上，评估-name，得到log文件，但之后没有action，所以评估得到log文件的结果作废。

```shell
[root@xuexi tmp]# find /tmp -type f -print -a -name "*.log" -print
/tmp/a.log
/tmp/a.log
/tmp/1.txt
/tmp/2.txt
/tmp/3.txt
/tmp/4.txt
/tmp/b.log
/tmp/b.log
```

-type f评估后得到所有普通文件，然后在普通文件的基础上评估-print，它直接将所有普通文件都输出，再在输出的文件的基础上，评估-name，得到log文件，再对得到的log文件评估-print，它将再次输出log文件，所以结果中将输出两次log文件。

![](/img/referer.jpg)

## 测试"or"操作符

有了上面的基础，再理解or操作符就简单多了。

直接猜下面4个命令的等价语句：

```shell
find /tmp -type f -o -name "*.log"
    --> find /tmp ( -type f -o -name *.log ) -a -print

find /tmp -type f -o -name "*.log" -print
    --> find /tmp -type f -o ( -name *.log -a -print )

find /tmp -type f -print -o -name "*.log"
    --> find /tmp ( -type f -a -print ) -o -name *.log

find /tmp -type f -print -o -name "*.log" -print
    --> find /tmp ( -type f -a -print ) -o ( -name *.log -a -print )
```

显然，上面第三个语句中的`-o -name *.log`是多余的。

以下是这4个命令的输出结果以及分析。

```shell
[root@xuexi tmp]# find /tmp -type f -o -name "*.log"
/tmp/a.log
/tmp/1.txt
/tmp/2.txt
/tmp/3.txt
/tmp/4.txt
/tmp/b.log
```

-type f评估后得到所有普通文件，但因为只有-o前面的表达式为假时才会评估-o后面的表达式，所以将会对非普通文件评估`-name *.log`，由于本示例中/tmp下没有非普通文件，所以-name将得不到结果，于是评估得到普通文件，最后在普通文件的基础上评估-print，即输出普通文件。

```shell
[root@xuexi tmp]# find /tmp -type f -o -name "*.log" -print
```

-type f评估后得到所有普通文件，然后对非普通文件评估`(-name *.log -a -print)`，由于没有非普通文件，所以这个括号评估结果为空。由于-type f后面没有给action，所以评估得到普通文件的结果也作废。因此，这个find命令什么都不会输出。

如果在/tmp下创建一个以".log"为后缀的目录，则该find命令会输出这个目录。

```shell
[root@xuexi tmp]# find /tmp -type f -print -o -name "*.log"
/tmp/a.log
/tmp/1.txt
/tmp/2.txt
/tmp/3.txt
/tmp/4.txt
/tmp/b.log
```

-type f评估后得到所有普通文件，然后直接输出，然后再对未输出的文件评估-name，也就是找出".log"结尾的非普通文件，但即使如此，后面也没有action，所以这里的-name是多余的。所以该命令等价于 find /tmp -type f -print 。

注意，-name评估的对象是未被-print输出的文件，而不是未被-type评估的文件。虽然在本示例中是等价的，但如果-print前还有-a条件，如`find /tmp -type f -a "*.txt" -print -o -name "*.log"`，这里的-name评估的对象是未被输出的文件，也就是`(-type f -a -name .txt)`的取反，而不是"-type f"的取反，所以log结尾的普通文件也会被-name评估，但因为没有后续的action，评估的结果会作废。

```shell
[root@xuexi tmp]# find /tmp -type f -print -o -name "*.log" -print
/tmp/a.log
/tmp/1.txt
/tmp/2.txt
/tmp/3.txt
/tmp/4.txt
/tmp/b.log
```

-type f评估后得到所有普通文件，然后直接输出，然后对非普通文件评估`(-name *.log -a -print)`，由于本处示例中没有以".log"结尾的非普通文件，所以这个括号的评估结果为空。但如果在/tmp下创建一个以".log"为后缀的目录，则该find命令会也输出这个目录。

# find深入用法示例

**(1). 指定目录的搜索深度**

例如搜索/etc目录下的".conf"文件，但不搜索任何子目录。

```shell
find /etc -maxdepth 1 -type f -name "*.conf"
```

注意，"-depth"、"-maxdepth"和"-mindepth"是find的option表达式，所以它们写在任意位置都会对test和action产生影响，所以它的位置可随意书写。

**(2). 指定目录的处理顺序：-depth**

"-depth"选项用于改变find使得它先搜索目录中的文件，然后才处理目录本身。默认情况下，find查找文件和目录的顺序是：假如/tmp下有a文件、b和c两个目录，b和c目录中都有一些文件和目录。

> 先查找/tmp本身
> 再查找/tmp下的各个文件，如a
> 再按序查找/tmp下的第一个目录，如b
> 再查找b中的文件。
> 查找/tmp下的下一个目录，如c
> 查找c中的文件

如果b和c目录中还有其他目录，则依照上面的处理规则进行处理。

即`/tmp --> /tmp中的文件 --> /tmp中第一个目录 --> 第一个目录中的文件 --> 第一个文件中的目录 --> … --> /tmp中第二个目录 --> 第二个目录中的文件 --> … --> /tmp中的第三个目录 --> …/tmp中的最后一个目录`。

如下面的图所示。

![](/img/shell/733013-20170613011415118-919879032.jpg)

如果指定了-depth，则先处理目录中的文件，再处理目录本身。在Linux一切皆文件，子目录也是文件。

例如，下面在/tmp目录下新建一个目录/tmp/tmp，并在子/tmp下创建文件a，目录b和c，其中分别有一些文件。使用-depth参数运行find。

```shell
[root@xuexi tmp]# touch a
[root@xuexi tmp]# mkdir b c
[root@xuexi tmp]# touch ./b/{1..3}.log ./c/{1..3}.sh
[root@xuexi tmp]# find /tmp/tmp -depth
/tmp/tmp/b/2.log
/tmp/tmp/b/1.log
/tmp/tmp/b/3.log
/tmp/tmp/b
/tmp/tmp/c/2.sh
/tmp/tmp/c/3.sh
/tmp/tmp/c/1.sh
/tmp/tmp/c
/tmp/tmp/a
/tmp/tmp
```

上面的find工作过程是：先找到/tmp/tmp，准备处理/tmp/tmp下的文件a、b、c(目录也是文件)，在这里的处理顺序是b、c、a(为什么这样的顺序我也不知道)，处理b的时候发现它是一个目录，转去处理b目录里的文件，处理完b中的文件后处理b目录，再处理c，发现是目录，转去处理c中的文件，处理完c中的文件后处理a文件。

![](/img/referer.jpg)

需要注意的是，不一定总是先处理目录和目录中的内容。在同一个目录下的文件是有处理顺序的。例如下面在/tmp/tmp下添加一个.x隐藏文件，再测试"-depth"。

```shell
[root@xuexi tmp]# touch .x            
[root@xuexi tmp]# find /tmp/tmp -depth
/tmp/tmp/b/2.log
/tmp/tmp/b/1.log
/tmp/tmp/b/3.log
/tmp/tmp/b
/tmp/tmp/.x   #   # 提前于c目录被查找到
/tmp/tmp/c/2.sh
/tmp/tmp/c/3.sh
/tmp/tmp/c/1.sh
/tmp/tmp/c
/tmp/tmp/a
/tmp/tmp
```

可以看到.x隐藏文件是在中途被处理的。

最后看看不加"-depth"的搜索顺序。

```shell
[root@xuexi tmp]# find /tmp/tmp 
/tmp/tmp
/tmp/tmp/b
/tmp/tmp/b/2.log
/tmp/tmp/b/1.log
/tmp/tmp/b/3.log
/tmp/tmp/.x
/tmp/tmp/c
/tmp/tmp/c/2.sh
/tmp/tmp/c/3.sh
/tmp/tmp/c/1.sh
/tmp/tmp/a
```



<a name="prune"></a>

**(3). 忽略搜索的结果：-prune**

一般只考虑忽略目录，不考虑忽略文件，一般忽略文件时会给出警告。再者，忽略某种文件可以使用其他条件来忽略。忽略目录后，Linux将直接不处理忽略的位置。

-prune需要放在定义忽略表达式的后面，之所以会如此，请看示例并考虑true和false进行理解。因为要忽略的是目录，所以一般都只和"-path"配合，而不跟"-name"配合，实际上和-name配合的时候会给出警告信息。

例如，查找/tmp中的".log"文件，但排除/tmp子abc目录内的".log"文件。

```shell
$ find /tmp -path "/tmp/abc" -prune -o -name "*.log"     
/tmp/testdir/a.log
/tmp/abc          # 注意这个被忽略的目录也在结果内
/tmp/a.log
/tmp/axyz.log
/tmp/xyz/axyz.log    
```

为何这里使用"-o"操作符而不是"-a"操作符呢？先将上述find表达式进行分解。第一个表达式-path "/tmp/abc"可以搜索出/tmp/abc文件，它可能是目录，也可能是文件，但无论它是什么文件，该表达式返回的总是true；第二个表达式是action类的"-prune"，它和第一个表达式是"expr1 expr2"这样的操作符格式，它等价于and逻辑关系，所以-path "/tmp/abc" -prune表示不进入匹配到的/tmp/abc目录(若abc为文件，则给出警告信息)，由于下一个表达式是使用"-o"连接的，所以到此为止就确定了该目录/tmp/abc，且是不进入的，由于返回true，-prune会将结果输出出来。

```shell
$ find /tmp -path "/tmp/abc" -prune 
/tmp/abc
```

继续，目的是搜索出非abc目录下的log文件，它的表达式应该是`-name "*.log"`。但由于前面返回的是true，如果使用"-a"选项连接前后，则表示后面的表达式的操作对象是/tmp/abc，但前面的逻辑是不进入/tmp/abc，所以将得不到任何结果。

```shell
$ find /tmp -path "/tmp/abc" -prune -a -name "*.log" | wc -l 
0
```

而使用"-o"连接，则表示`-name "*.log"`的操作对象是path部分(即/tmp目录)除了/tmp/abc的其余文件，它将找出/tmp下所有的log文件，但由于前面明确表示了不进入/tmp/abc目录，所以就实现了忽略/tmp/abc目录的作用。

它的等价命令如下：

```shell
find /tmp ( ( -path "/tmp/abc" -a -prune ) -o -name "*.log" ) -a print
```

应该注意到了，上面的/tmp/abc目录也出现在结果中了，但是它并非log文件。这是因为-prune的副作用，find认为-prune不是一个完整的action，它会补上-print。如果想要完完全全的忽略该目录，则可以使用下面的方式，在-prune之后加上-false强制使得该段表达式返回false，这样该段结果就不会被输出。

```shell
$ find /tmp -path "/tmp/abc" -prune -false -o -name "*.log"
/tmp/testdir/a.log
/tmp/a.log
/tmp/axyz.log
/tmp/xyz/axyz.log
```

-prune的一个弱点是不适合通过通配符来忽略目录，因为通配符出来的很可能导致非预期结果。

```shell
$ find /tmp -path "/tmp/a*" -prune -o -name "*.log"   
/tmp/testdir/a.log
/tmp/a.txt
/tmp/abc
/tmp/about.html
/tmp/a.log
/tmp/axyz.log
/tmp/xyz/axyz.log
```

![](/img/referer.jpg)

所以想要忽略多个目录，最好的方法是多次使用-path，但要注意它们的逻辑顺序。例如，搜索/tmp下所有log文件，但忽略/tmp/abc和/tmp/xyz两个目录中的log文件。

```shell
$ find /tmp \( -path '/tmp/abc' -o -path '/tmp/xyz' \)
/tmp/abc
/tmp/xyz
```

所以完整的写法是：

```shell
$ find /tmp \( -path /tmp/abc -o -path /tmp/xzy \) -prune -o -name "*.log"
/tmp/testdir/a.log
/tmp/abc
/tmp/a.log
/tmp/axyz.log
/tmp/xyz
```

**(4). 不显示待搜索目录本身**

在find显示结果的时候，如果没有表达式过滤掉目录本身，那么目录本身也会被显示出来，但是很多时候这时不必要的。一般用于只显示一级目录下所有的文件，但不包括目录本身。

```shell
$ find ~ -maxdepth 1
/root # 搜索目录本身也被显示出来
/root/.bash_history
/root/.lesshst
/root/install.log
/root/.viminfo
/root/install.log.syslog
/root/.tcshrc
/root/.lftp
/root/.cshrc
/root/raid.sh
/root/.bashrc
/root/anaconda-ks.cfg
/root/.bash_profile
/root/bin
/root/.bash_logout
```

要过滤掉目录本身，方式也很简单，多使用一个匹配表达式即可。

```shell
# 或者 find ~ -maxdepth 1 -mindepth 1
# 或者 find ~ -maxdepth 1 -atime +0
$ find ~ -maxdepth 1 ! -name root
/root/.bash_history
/root/.lesshst
/root/install.log
/root/.viminfo
/root/install.log.syslog
/root/.tcshrc
/root/.lftp
/root/.cshrc
/root/raid.sh
/root/.bashrc
/root/anaconda-ks.cfg
/root/.bash_profile
/root/bin
/root/.bash_logout
```

**(5). 搜索指定目录下非空文件**

"-empty"测试条件用于测试文件是否非空，对于目录而言则是目录为空目录。一般可用于在搜索时，排除空文件或空目录，所以一般和"!"或"-not"一起使用。

例如，搜索/tmp下非空文件和非空目录。

```shell
[root@xuexi tmp]# find /tmp/tmp ! -empty
```



