---
title: sed修炼系列(一)：花拳绣腿入门篇
p: shell/sed1.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列和sed系列文章大纲](/shell/index#sed)**  

------

# sed修炼系列(一)：花拳绣腿入门篇

本文为花拳绣腿招式入门篇，主要目的是入门，为看懂[sed修炼系列(二)：武功心法][2]做准备。虽然是入门篇，只介绍了基本工作机制以及一些选项和命令，但其中仍然包括了很多sed的工作机制细节。对比网上各sed相关文章以及介绍sed的书籍，基本上都只介绍了sed是如何使用的，却没有"How sed Works"这种工作机制的原理性内容，最多给出一段稍微解释下。即使是非常流行的《sed & awk》也只是零零散散地介绍了一些sed工作机制细节。我想本文应能给你带来一些对sed的新认知。

<a name="blog1.1"></a>

## 基本概念

sed是一个流式编辑器程序，它读取输入流(可以是文件、标准输入)的每一行放进模式空间(pattern space)，同时将此行行号通过sed行号计数器记录在内存中，然后对模式空间中的行进行模式匹配，如果能匹配上则使用sed程序内部的命令进行处理，处理结束后，从模式空间中输出(默认)出去，并清空模式空间，随后再从输入流中读取下一行到模式空间中进行相同的操作，直到输入流中的所有行都处理完成。由此可见，sed是一个循环一个循环处理内容的。

这是sed的一个循环的过程：  
- 1.读取输入流的一行到模式空间。
- 2.对模式空间中的内容进行匹配和处理。
- 3.自动输出模式空间内容。
- 4.清空模式空间内容。
- 5.读取输入流的下一行到模式空间。

(注：(如看不懂，请跳过)如果是读取文件数据，则会每次需要的时候一次性加载一定量(比如多行)的数据到os buffer，然后sed从os buffer中一行一行读取，并不是要读一行就从磁盘文件中加载一行。另外，如果是管道或其它输入流，则直接从对应的缓存中一行一行读取。验证命令：`sed 'p;s/.*/:>filename/e;d' filename`)

上述整个循环过程中，**第2步是我们写sed命令所修改的地方，其余的几个步骤，通过命令行无法改变**。但是，sed有几个命令和选项能改变第3、4步的行为，使其输出总是输出空内容或无法清空模式空间。

sed程序的语法格式为：
```
sed OPTIONS SCRIPT INPUT_STREAM
```

其中SCRIPT部分就是所谓的sed脚本，它是sed内部命令的集合，sed中的命令有些奇特，它包含行匹配以及要执行的命令。格式为`ADDR1[,ADDR2]cmd_list`。例如，要对第2行执行删除命令，其命令为`sed 2d filename`，只输出第4行到6行，其命令为`sed -n 4,6p`。

sed的内部命令非常多，但既然"花拳绣腿篇"，当然只介绍些入门的东西。具体的行匹配方法、有哪些命令以及哪些选项稍后解释。现在的重点是sed中的循环过程。既然SCRIPT是命令的集合，于是上面的循环过程可以修改为如下：

![](/img/shell/1699322450870.png)

其中SCRIPT部分包含了sed命令行中的内部命令，**还包括两个特殊动作：自动输出和清空模式空间内容。这两个动作是一定会执行的**，只不过有些时候通过某些命令可以使其输出空内容、使其清空不了模式空间。

如果使用编程结构来描述，则大致过程如下：
```
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        execute cmd3 in SCRIPT;
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
其中while循环执行的正是SCRIPT中的所有命令，只不过一般情况下，while循环只执行一轮就退出并进入外层的for循环。于是，外层的for循环称之为"sed循环"，内层的while循环称之为"SCRIPT"循环。所以，for循环只包含了两个动作：读取下一行和执行SCRIPT循环。

其实while循环中是有continue、break甚至是exit的，分别表示回到SCRIPT的顶端(即进入下一个SCRIPT循环)、退出当前SCRIPT循环回到外层sed循环以及退出整个sed循环。显然，这不是"花拳绣腿"的内容。

最后，说明下sed命令行如何书写，其实就是写SCRIPT部分，这部分的写法比较灵活，大致有以下几种：
```
# 一行式。多个命令使用分号分隔
sed Address{cmd1;cmd2;cmd3...}

# 多个表达式时，可以使用"-e"选项，也可以不用，但使用分号分隔
sed Address1{cmd1;cmd2;cmd3};Address2{cmd1;cmd2;cmd3}...
sed  -e 'Address1{cmd1;cmd2;cmd3}' -e 'Address2{cmd1;cmd2;cmd3}' ...

# 分行写时
sed Address1{
    cmd1
    cmd2
    cmd3
}
Address2{
    cmd1
    cmd2
    cmd3
}
```
如果是写在文件中，即sed脚本，以文件名为a.sed为例。
```
#!/usr/bin/sed -f
#注释行
Address1{cmd1;cmd2...}
Address2{cmd1;cmd2...}
......
```
其中cmd部分还可以进行模式匹配，也即类似于`{% raw %} Address{{pattern1}cmd1;{pattern2}cmd2} {% endraw %}`的写法。例如，`/^abc/{2d;p}`。  

有了以上基本的大纲性知识，理解和深入sed机制就简单多了。

<a name="blog1.2"></a>
## sed选项

sed选项不算多，能用到的更没几个。
```
sed OPTIONS SCRIPT INPUT_STREAM
```
可能用到的几个选项：

`'-n'`  
默认情况下，sed将在**每轮script循环结束**时自动输出模式空间中的内容。使用该选项后可以使得这次自动输出动作输出**空内容**，而不是当前模式空间中的内容。注意，"-n"是输出空内容而不是禁用输出动作，虽然两者的结果都是不输出任何内容，但在有些依赖于输出动作和输出流的地方，它们的区别是很大的，前者有输出流，只是输出空流，后者则没有输出流。

`'-e SCRIPT'`  
前文说了，SCRIPT中包含的是命令的集合，"-e"选项就是向SCRIPT中添加命令的。可以省略"-e"选项，但如果命令行容易产生歧义，则使用"-e"选项可明确说明这部分是SCRIPT中的命令。另外，如果一个"-e"选项不方便描述所需命令集合时，可以指定多个"-e"选项。

`'-f SCRIPT-FILE'`  
指定包含命令集合的SCRIPT文件，让sed根据SCRIPT文件中的命令集处理输入流。

`'-i[SUFFIX]'`    
该选项指定要将sed的输出结果保存(覆盖的方式)到当前编辑的文件中。GNU sed是通过创建一个临时文件并将输入写入到该临时文件，然后重命名为源文件来实现的。

当当前输入流处理结束后，临时文件被重命名为**源文件**的名称。如果还提供了SUFFIX，则在重命名临时文件之前，先使用该SUFFIX修改源文件名，从而生成一个源文件的备份文件。  

**临时文件总是会被重命名为源文件名称**，也就是说输入流处理结束后，仍使用源文件名的文件是sed修改后的文件。文件名中包含了SUFFIX的文件则是最原始文件的备份。例如源文件为a.txt，`sed -i'.log' SCRIPT a.txt`将生成两个文件：a.txt和a.txt.log，前者是sed修改后的文件，a.txt.log是源a.txt的备份文件。

重命名的规则如下：如果扩展名不包含符号`*`，将SUFFIX添加到原文件名的后面当作文件后缀；如果SUFFIX中包含了一个或多个字符`*`，则每个`*`都替换为原文件名。这使得你可以为备份文件添加一个前缀，而不是后缀。如果没有提供SUFFIX，源文件被覆盖，且不会生成备份文件。

该选项隐含了"-s"选项。

`'-r'`  
使用扩展正则表达式，而不是使用默认的基础正则表达式。sed所支持的扩展正则表达式和`egrep`一样。使用扩展正则表达式显得更简洁，因为有些元字符不用再使用反斜线`\`。正则表达式见[grep命令中文手册][1]。

`'-s'`  
默认情况下，如果为sed指定了多个输入文件，如`sed OPTIONS SCRIPT file1 file2 file3`，则多个文件会被sed当作一个长的输入流，也就是说所有文件被当成一个大文件。指定该选项后，sed将认为命令行中给定的每个文件都是独立的输入流。

既然是独立的输入流，**范围定址(如`/abc/,/def/`)就无法跨越多个文件进行匹配，行号也会在处理每个文件时重置，"$"代表的也将是每个文件的最后一行**。这也意味着，如果不使用该选项，则这几个行为都是可以完成的。

示例：以sed命令"p"和"="为例，其中"p"命令用于强制输出当前模式空间中的内容，"="命令用于输出sed行号计数器当前的值，即刚被读入到模式空间中的行是输入流中的第几行。

(1).只输出a.txt中的第5行。
```
sed -n 5p a.txt
```
这里使用了"-n"选项，使得读取到模式空间的每一行都无法被输出，只有明确使用了"p"选项才能被"p"动作输出。由于只有读入的第5行内容能匹配"5"，才能被"p"输出。

其实上面的命令和`sed -n -e '5p' a.txt`是完全一样的，因为"5p"在sed解析命令行时不会产生歧义，所以可以省略"-e"选项。

(2).输出a.txt，并输出每行的行号。
```
sed '=' a.txt
```
由于要输出a.txt的内容，所以不使用"-n"选项，同时"="命令会输出每行行号。

(3).分别输出a.txt和b.txt的第5行，并分别保存到".bak"后缀的文件中。
```
sed -i'*.bak' -n '5p' a.txt b.txt
```
此处必须使用"-s"选项，否则将只会输出"a.txt+b.txt"结合后的第5行。但"-i"隐含了"-s"选项。这会生成4个文件：a.txt、b.txt和a.txt.bak、b.txt.bak。前两个是第5行内容，后两个是源文件的备份文件。

(4).使用扩展正则表达式，输出a.txt和b.txt中能包含3个以上字母"a"的行。
```
sed -r -n '/aaa+/p' a.txt b.txt
```

<a name="blog1.3"></a>
## 定址表达式

当sed将输入流中的行读取到模式空间后，就需要对模式空间中的内容进行匹配，如果能匹配就能执行对应的命令，如果不能匹配就直接输出、清空模式空间并进入下一个sed循环读取下一行。

匹配的过程称为定址。定址表达式有多种，但总的来说，其格式为`[ADDR1][,ADDR2]`。这可以分为3种方式：

![](/img/shell/1699322473668.png)

无论是ADDR1还是ADDR2，都可以使用两种方式进行匹配：行号和正则表达式。如下：

`'N'`  
指定一个行号，sed将只匹配该行。(需要注意，除非使用了"-s"或"-i"选项，sed将对所有输入文件的行连续计数。)

`'FIRST~STEP'`  
表示从第FIRST行开始，每隔STEP行就再取一次。也就是取行号满足`FIRST+(N*STEP)` (其中N>=0)的行。因此，要选择所有奇数行，使用`1~2`；要从第2行开始每隔3行取一次，使用`2~3`；要从第10行开始每隔5行取一次，使用`10~5`；而`50~0`则表示只取第50行。

`'$'`  
默认该符号匹配的是最后一个文件的最后一行，如果指定了"-i"或"-s"，则匹配的是每个文件的最后一行。总之，`$`匹配的是每个输入流的最后一行。

![](/img/shell/1699322491652.png)

`'/REGEXP/'`  
将选择能被正则表达式REGEXP匹配的所有行。如果REGEXP中自身包含了字符"/"，则必须使用反斜线转义，即`\/`。

`'/REGEXP/I'`  
和`/REGEXP/`是一样的，只不过匹配的时候不区分大小写。

`'\%REGEXP%'`  
('%'可以使用其他任意单个字符替换。)
这和上一个定址表达式的作用是一样的，只不过是使用符号"%"替换了符号"/"。当REGEXP中包含"/"符号时，使用该定址表达式就无需对"/"使用反斜线`\`转义。但如果此时REGEXP中包含了"%"符号时，该符号需要使用`\`转义。  
总之，定址表达式中使用的分隔符在REGEXP中出现时，都需要使用反斜线转义。

`'ADDR1,+N'`  
匹配ADDR1和其后的N行。

`'ADDR1,~N'`  
匹配ADDR1和其后的行直到出现N的倍数行。倍数可为随意整数倍，只要N的倍数是最接近且大于ADDR1的即可。
如`ADDR1=1,N=3`匹配1-3行，`ADDR1=5,N=4`匹配5-8行。而`1,+3`匹配的是第一行和其后的3行即1-4行。

另外，在定址表达式的后面加"!"符号表示反转匹配的含义。也就是说那些匹配的行将不被选择，而是不匹配的行被选择。

例如，以下几个定址的示例：
```
sed -n '3p' INPUTFILE
sed -n '3,5!p' INPUTFILE
sed -n '3,/^# .*/! p' INPUTFILE
sed -n '/abc/,/xyz/p' INPUTFILE
sed -n '!p' INPUTFILE   # 这个有悖常理，但确实是允许的
```

<a name="blog1.4"></a>
## sed常用命令

sed命令很多，本文的只简单介绍几个最常见的。

**此处不以命令的用法为重，而是通过这几个命令，引出sed最重要的原理和执行机制(还包括本文的第一节内容)**，并为阅读下一篇文章sed武功心法：info sed打下基础。而且理解了这些原理，再使用sed做任何操作都有理可循，遇到疑难之处也知道如何进行分析。。

**(1).强制输出命令"p"。**

该命令能强制输出当前模式空间的内容。即使使用了"-n"选项。

事实上，它们本就不冲突，因为循环过程如下：
```
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{print};        # "p" command
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
![](/img/shell/1699322512851.png)

例如，仅输出标准输入的第2行内容。
```
[root@xuexi ~]# echo -e 'abc\nxyz' | sed -n 2p
xyz
```
不加"-n"选项，在"p"输出之后，SCRIPT循环的结尾处还会被auto\_print输出一次。
```
[root@xuexi ~]# echo -e 'abc\nxyz' | sed 2p   
abc
xyz    # 这是p命令输出的结果
xyz    # 这是自动输出的结果
```
**(2).删除命令"d"。**

命令"d"用于删除**整个模式空间**中的内容，**并立即退出当前SCRIPT循环，进入下一个sed循环，即读取下一行**。

循环大致格式如下:
```
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{delete;break};     # "d" command
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
唯一需要注意的一点是立即退出当前SCRIPT循环，这意味着如果"d"命令后面还有其他的命令，则这些命令都不会执行。

例如：删除a.txt中的第5行，并保存到原文件中。
```
sed -i '5d' a.txt
```
这里不能使用重定向的方式保存，因为重定向是在sed命令执行前被shell执行的，所以会截断a.txt，使得sed读取的输入流为空，或者结果出乎意料之外。而"-i"选项则不会操作原文件，而是生成临时文件并在结束时重命名为原文件名。

删除a.sh中包含"#"开头的注释行，但第一行的`#!/bin/bash`不删除。
```
sed '/^#/{1!d}' a.sh 
```
如果"d"后面还有命令，在删除模式空间后，这些命令不会执行，因为会立即退出当前SCRIPT循环。例如：
```
echo -e 'abc\nxyz' | sed '{/abc/d;=}'
2
xyz
```
其中"="这个命令用于输出行号，但是结果并没有输出被"abc"匹配的行的行号。

**(3).退出sed程序命令"q"和"Q"。**

使用"q"和"Q"命令的作用是立即退出当前sed程序，使其不再执行后面的命令，也不再读取后面的行。因此，在处理大文件或大量文件时，使用"q"或"Q"命令能提高很大效率。它们之间的不同之处在于"q"命令被执行后还会使用自动输出动作输出模式空间的内容，除非使用了"-n"选项。而"Q"命令则会立即退出，不会输出模式空间内容。另外，可以为它们指定退出状态码，例如"q 1"。

使用了"q"和"Q"的sed循环结构大致如下：
```
# "q"命令
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{auto_print;exit};     # "q" command
        ……
        auto_print;
        remove_pattern_space;
    done
done

# "Q"命令
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{exit};      # "Q" command
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
例如，搜索脚本a.sh，当搜索到使用了"."或"source"命令加载环境配置脚本时就输出并立即退出。
```
sed -n -r '/^[ \t]*(\.|source) /{p;q}' a.sh 
```

**(4).输出行号命令"="。**

![](/img/shell/1699322556987.png)

例如，搜索出httpd.conf中"DocumentRoot"开头的行的行号，允许有前导空白字符。
```
sed -n '/^[ \t]*DocumentRoot/{p;=}' httpd.conf        
DocumentRoot "/var/www/html"
119
```
如果"="命令前没有"p"输出命令，且没有使用"-n"选项，则是输出在Document所在行的前一行，因为SCRIPT最后的自动输出动作也有输出流。

**(5).字符一一对应替换命令"y"。**

该命令和"tr"命令的映射功能一样，都是将字符进行一一替换。

例如，将a.txt中包含大写字母的YES、Yes等替换成小写的yes。
```
sed 'y/YES/yes/' a.txt
```

**(6).手动读取下一行命令"n"。**

在sed的循环过程中，每个sed循环的第一步都是读取输入流的下一行到模式空间中，这是我们无法控制的动作。但sed有读取下一行的命令"n"。

由于是读取下一行，所以它会触发自动输出的动作，于是就有了输出流。不仅如此，还应该记住的是：**只要有读取下一行的行为，在其真正开始读取之前一定有隐式自动输出的行为**。

但需注意，当没有下一行可供"n"读取时(例如文件的最后一行已经被读取过了)，将输出模式空间内容后直接退出sed程序，使得"n"命令后的所有命令都不会执行，即使是那两个隐含动作。

相应的循环结构如下：
```
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{              # "n" command
            if [ "$line" -ne "$last_line_num" ];then
                auto_print;
                remove_pattern_space;
                read next_line to pattern_space;
            else
                auto_print;
                remove_pattern_space;
                exit;
            fi
        }; 
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
注意，是先判断是否有下一行可读取，再输出和清空pattern space中的内容，所以then和else语句中都有这两个动作。
也许感觉上似乎更应该像下面这样的优化形式：
```
ADDR1,ADDR2{    # "n" command
        auto_print;
        remove_pattern_space;
        [ "$line" -ne "$last_line_num" ] && read next_line to pattern_space || exit;
}; 
```
但事实证明并非如此，证明过程在[本文结尾](#blog1.5)。此处暂不讨论这些复杂的东西，先看看"n"命令的示例。

例如，搜索a.txt中包含"redirect"字符串的行以及其下一行，并输出。
```
sed -n '/redirect/{p;n;p}' a.txt
```
再例如下面的命令。
```
echo -e "abc\ndef\nxyz" | sed '/abc/{n;=;p}' 
abc
2
def
def
xyz
```
从结果中可以分析出，"n"读取下一行前输出了"abc"，然后立即读入了下一行，所以输出的行号是2而不是1，因为这时候行号计数器已经读取了下一行，随后命令"p"输出了该模式空间的内容，输出后还有一次自动输出的隐含动作，所以"def"被输出了两次。

**(7).替换命令"s"。**

这是sed用的最多的命令。两个字就能概括其功能：替换。将匹配到的内容替换成指定的内容。

"s"命令的语法格式为：其中"/"可以替换成任意其他单个字符。
```
s/REGEXP/REPLACEMENT/FLAGS
```
它使用REGEXP去匹配行，将匹配到的那部分字符替换成REPLACEMENT。FLAGS是"s"命令的修饰符，常见的有"g"、"p"和"i"或"I"。

![](/img/shell/1699322594053.png)

REPLACEMENT中可以使用`\N`(N是从1到9的整数)进行后向引用，所代表的是REGEXP第N个括号`(...)`匹配的内容。另外，REPLACEMENT中可以包含未转义的`&`符号，这表示引用pattern space中被匹配的整个内容。需要注意，`&`是引用pattern space中的所有匹配，不仅仅只是括号的分组匹配。

例如，删除a.sh中所有`#`开头(可以包括前导空白)的注释符号`#`，但第一行`#!/bin/bash`不处理。
```
sed -i '2,$s/^[ \t]*#//' a.sh
```
为a.sh文件中的第5行到最后一行的行首加上注释符号`#`。
```
sed '5,$s/^/#/' a.sh
```
将a.sh中所有的"int"单词替换成"SIGINT"。
```
    sed 's/\bint\b/SIGINT/g' a.sh
```
将a.sh中`cmd1 && cmd2 || cmd3`的cmd2和cmd3命令对调个位置。
```
    sed 's%&&\(.*\) ||\(.*\)%\&\&\2 ||\1%' a.sh  
```
这里使用了"%"代替"/"，且在REPLACEMENT部分对`&`进行了转义，因为该符号在REPLACEMENT中时表示的是引用REGEXP所匹配的所有内容。

**(8).追加、插入和修改命令"a"、"i"、"c"。**

这3个命令的格式是"[a|i|c] TEXT"，表示将TEXT内容队列化到内存中，当有输出流或者说有输出动作的时候，半路追上输出流，分别追加、插入和替换到该输出流然后输出。追加是指追加在输出流的尾部，插入是指插入在输出流的首部，替换是指将整个输出流替换掉。"c"命令和"a"、"i"命令有一丝不同，它替换结束后立即退出当前SCRIPT循环，并进入下一个sed循环，因此"c"命令后的命令都不会被执行。

例如：
```
echo -e "abc\ndef" | sed '/abc/a xyz'
abc
xyz
def
```
其实"a"、"i"和"c"命令的TEXT部分写法是比较复杂的，如果TEXT只是几个简单字符，如上即可。但如果要TEXT是分行文本，或者包含了引号，或者这几个命令是写在`{}`中的，则上面的写法就无法实现。需要使用符号`\`来转义行尾符号，这表示开启一个新行，此后输入的内容都是TEXT，直到遇到引号或者";"开头的行时。

例如，在a.sh的`#!/bin/bash`行后添加一个注释行`# Script filename: a.sh`及一个空行。由于是追加在尾部，所以使用"a"命令。
```
sed '\%#!/bin/bash%a\# Script filename: a.sh\n' a.sh
```
"a"命令后的第一个反斜线用于标记TEXT的开始，`\n`用于添加空白行。如果分行写，或者"a"命令写在大括号`{}`中，则格式如下：
```
sed '\%#!/bin/bash%a\
# Script filename: a.sh\n
' a.sh

sed '\%#!/bin/bash%{p;a\
# Script filename: a.sh\n
;p}' a.sh
```
最后需要说的是，**这3个命令的TEXT是存放在内存中的，不会进入模式空间，因此不受"-n"选项或某些命令的影响。此外，这3个命令依赖于输出流，只要有输出动作，不管是空输出流还是非空的输出流，只要有输出，这几个命令就会半路"劫杀"**。如果不理解这两句话，这3个命令的结果有时可能会比较疑惑。

例如，"a"命令是追加在当前匹配行行尾的，但为什么下面的"haha"却插入到匹配行"def"的前面去了呢？
```
echo -e "abc\ndef\nxyz" | sed '/def/{a\
haha
;N}'

abc
haha
def
xyz
```
阅读了下面的"N"命令之后，再回头看这个示例，应该能知道为什么。在[sed修炼系列(四)：sed中的疑难杂症][4]中给出了解释。

**(9).多行模式命令"N"、"D"、"P"简单说明。**

在前面已经解释了"n"、"d"和"p"命令，sed还支持它们的大写命令"N"、"D"和"P"。

- "N"命令：读取下一行内容追加到模式空间的尾部。其和"n"命令不同之处在于："n"命令会输出模式空间的内容(除非使用了"-n"选项)并清空模式空间，然后才读取下一行到模式空间，也就是说"n"命令虽然读取了下一行到模式空间，但模式空间仍然是单行数据。而"N"命令在读取下一行前，虽然也有自动输出和清空模式空间的动作，但该命令会把当前模式空间的内容**锁住**，使得自动输出的内容为空，也无法清空模式空间，然后读取下一行追加到当前模式空间中的尾部。追加时，原有内容和新读取内容使用换行符`\n`分隔，这样在模式空间中就实现了多行数据。即所谓的"多行模式"。  另外，当无法读取到下一行时(到了文件尾部)，将直接退出sed程序，使得"N"命令后的命令不会再执行，这和"n"命令是一样的。

- "D"命令：删除模式空间中第一个换行符`\n`之前的内容，然后立即回到SCRIPT循环的顶端，即进入下一个SCRIPT循环。如果"D"删除后，模式空间中已经没有内容了，则SCRIPT循环自动退出进入下一个sed循环；如果模式空间还有剩余内容，则继续从头执行SCRIPT循环。也就是说，"D"命令后的命令不会被执行。

- "P"命令：输出模式空间中第一个换行符`\n`之前的内容。

"N"、"D"和"P"命令作用非常大，它们是绝佳的组合命令，因为借助它们能实现"窗口滑动"技术，这对于复杂的文本行操作来说大有裨益。但显然，这不是本文的内容，在[sed修炼系列(三)：sed高级应用之实现窗口滑动技术][3]中详细说明了这3个命令的功能。

此处按照惯例，还是给出它们的大致循环结构：其中"N"命令的if判断和前文的"n"一样，在[本文结尾证明](#blog1.5)。
```
# "N"命令的大致循环结构 
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{           # "N" command
            if [ "$line" -ne "$last_line_num" ];then
                lock pattern_space;
                auto_print;
                remove_pattern_space;
                unlock pattern_space;
                append "\n" to pattern_space;
                read next_line to pattern_space;
            else
                auto_print;
                remove_pattern_space;                   
                exit;
            fi
        }; 
        ……
        auto_print;
        remove_pattern_space;
    done
done

# "D"命令的大致循环结构
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{               # "D" command
            delete first line in pattern_space;
            continue;
        }; 
        ……
        auto_print;
        remove_pattern_space;
    done
done

# "P"命令的大致循环结构
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ADDR1,ADDR2{               # "P" command
            print first line in pattern_space;
        }; 
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
**(10).buffer空间数据交换命令"h"、"H"、"g"、"G"、"x"简单说明。**

sed除了维护模式空间(pattern space)，还维护另一个buffer空间：保持空间(hold space)。这两个空间初始状态都是空的。

绝大多数时候，sed仅依靠模式空间就能达到目的，但有些复杂的数据操作则只能借助保持空间来实现。之所以称之为保持空间，是因为它是暂存数据用的，除了仅有的这几个命令外，没有任何其他命令可以操作该空间，因此借助它能实现数据的持久性。

保持空间的作用很大，它和模式空间之间的数据交换能实现很多看上去不能实现的功能，是实现sed高级功能所必须的，例如"窗口滑动"。同样，这不是本文的内容。所以只简单解释这几个命令的作用：

![](/img/shell/1699322642309.png)

注意，无论是交换、追加还是覆盖，原空间的内容都不会被删除。


<a name="blog1.5"></a>
## 总结

看到这里，对sed已经有了一些概念，也许已经发现了sed的重点在于各选项和各命令是如何影响sed循环以及SCRIPT循环的。确实如此，在info sed文档中，虽然没有将这些工作机制详细描述，但各选项各命令说明中，在需要的时候都提到了这些细节，而我所做的只不过是将其系统性地描述出来、做一些深入，再给几个示例解释，并使用通俗易懂的循环结构来展示这些机制。

最后，验证前文"n"和"N"命令留下的疑问："n"和"N"命令是先判断是否还有下一行，再自动输出的。也就是证明下面两个判断语句采用前者还是后者的问题。
```
ADDR1,ADDR2{              # "n" command
    if [ "$line" -ne "$last_line_num" ];then
        auto_print;
        remove_pattern_space;
        read next_line to pattern_space;
    else
        auto_print;
        remove_pattern_space;
        exit;
    fi
}; 

ADDR1,ADDR2{              # "n" command
        auto_print;
        remove_pattern_space;
        [ "$line" -ne "$last_line_num" ] && read next_line to pattern_space || exit;
}; 
```
虽然后者看上去代码更优化，但事实上采用的是前者。要证明这一点不太容易，好在我想出了下面的方法来证明。下面的示例中使用的是"N"，它和"n"在判断逻辑上的行为是一致的。
```
[root@xuexi ~]# echo -e "abc\ndef\nxyz" | sed '/def/{a\   
haha
;N}'

abc
haha
def
xyz

[root@xuexi ~]# echo -e "abc\ndef" | sed '/def/{a\     
haha
;N}'

abc
def
haha
```
在以上两个命令中，第一个命令"haha"是插入在匹配行"def"的前面，而第二个命令则是插入在"def"的后面。似乎根据"a"命令的作用来说，第二个命令才是意料之中的结果。

首先，解释第一个命令为何"haha"会出现在匹配行"def"的前面。当sed读取的行能匹配"def"时，将队列化"haha"到内存中，并在有输出流的时候追加到输出流尾部。由于这里的输出流来自于"a"命令后的"N"命令，该命令将模式空间锁住，使得隐含动作自动输出的内容为空，但队列化的内容还是发现了这个空输出流，于是追加在这个空流的尾部。再之后，"N"将下一行读取到模式空间中，到了SCRIPT循环的结尾，再次自动输出，此时模式空间有两行："def" 和 "xyz"，这两行同时被输出。显然，在"def"被输出之前，队列化的内容已经随着空输出流而输出了。

再解释为何第二个命令的结果中"haha"在"def"之后，这也是待证明的疑问。第二个命令中，由于"def"已经是输入流的最后一行，"N"已经无法再读取下一行，于是输出当前模式空间内容并退出sed程序。假设，"n"或"N"命令是先自动输出、清空模式空间内容，再判断是否有下一行可读取的，那么在判断之前自动输出时，"N"不知道是否还有下一行，于是队列化的内容应该同第一个命令一样，插入在"def"之前。但结果却并非如此。如果先判断是否有下一行可供读取，再输出、清空模式空间，则队列化内容是跟随着"N"退出sed程序前输出的，这正符合第二个命令的结果。

[1]: /shell/grep_translate  "info grep正则翻译"
[2]: /shell/sed2   "sed武功心法(info sed翻译+注解)"
[3]: /shell/sed3   "sed修炼系列(三)：sed高级应用之实现窗口滑动技术"
[4]: /shell/sed4#blog1.5  "命令a和命令N的纠葛"

