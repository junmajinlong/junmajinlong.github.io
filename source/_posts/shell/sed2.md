---
title: sed命令中文手册(info sed翻译+译注解释)
p: shell/sed2.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列和sed系列文章大纲](/shell/index#sed)**  

------

# sed命令中文手册(info sed翻译+译注解释)

1. 本文中的提到GNU扩展时，表示该功能是GNU为sed提供的(即GNU版本的sed才有该功能)，一般此时都会说明：如果要写具有可移植性的脚本，应尽量避免在脚本中使用该选项。
2. 本文中的正则表达式几乎和grep中支持的一样。但还是有少数几个是sed自身才能解析的表达式。因此本译文中只对这些sed自身才支持的正则表达式做翻译，其余sed支持的通用性正则表达式见我翻译的[grep命令中文手册](https://www.junmajinlong.com/shell/grep_translate)。
3. 此外，除了正则表达式部分，还有些地方没有进行翻译，因为个人觉得几乎用不上，没必要翻译。但为了保持文章的完整性，仍给出了原文内容。
4. 个人建议：如只想了解sed的简单用法，不建议阅读本译文；但如果想深入学习或掌握sed，info sed是最佳选择，很多处理机制在书上(即使是最为流行的《sed & awk》)都没有深入说明。本文为我第二次翻译，两次翻译过程中收获都极大。
5. 译文中有些地方加上了`"(注：)"`，为本人自行加入，助于理解和说明，非原文内容。PS：直到翻译完，才发现加了很多很多我个人注释，尽管是一篇译文，但也舍不得删掉这些感想。如果这些"注："有碍观感，还望各位能体谅。
6. 学习sed的过程中，推荐使用"[sedsed: http://aurelio.net/projects/sedsed/](http://aurelio.net/projects/sedsed/)"调试工具，这对于分析sed处理过程以及pattern space、hold space有很大帮助。但极少数情况下，它的结果可能并不如想象中那样准确。
7. 最后，本文篇幅太大，难免有遗漏的地方，所以如果发现了哪里有错误或有疑问的地方，盼请指出。同时，也期待与各位交流sed的使用。

[1 简介](#blog1)   
[2 调用方式](#blog2)   
[3 sed程序](#blog3)   
　　[3.1 sed是如何工作的](#blog3.1)  
　　[3.2 sed定址：筛选行的方式](#blog3.2)  
　　[3.3 正则表达式一览](#blog3.3)  
　　[3.4 sed常用命令](#blog3.4)  
　　[3.5 sed的s命令](#blog3.5)  
　　[3.6 比较少用的sed命令](#blog3.6)  
　　[3.7 大师级的sed命令(sed标签功能)](#blog3.7)  
　　[3.8 GNU sed特有的命令](#blog3.8)   
　　[3.9 GNU对正则表达式的反斜线扩展](#blog3.9)  
[4 一些简单的示例脚本](#blog4)  
[5 GNU sed的限制和优点](#blog5)  
[6 学习sed的其他资源](#blog6)  
[7 Bugs说明(建议看)](#blog7)  

<a name="blog1"></a>
## 1. Introduction(简介) 

sed是一个流式编辑器。流式编辑器用于对输入流(文件或管道传递的数据)执行基本的文本转换操作。在某些方面上，sed有点类似于脚本化编辑的编辑器(如`ed`)，但sed只能**通过一次输入流**，因此它的效率要更高。但sed区别于其他类型编辑器的地方在于它能筛选过滤管道传递的文本数据。  
(注：sed只能通过一次输入流，意味着每次的输入流只能处理一次，因此像`(sed -n '2{p;q}';sed -n 3{p;q}) <filename`这样的命令中，第二个sed语句读取的输入流是空流。)

<a name="blog2"></a>
## 2. Invocation(调用方式)
通常，会使用下面的方式来调用sed：  
```
sed SCRIPT INPUTFILE...
```
完整的调用格式为：

    sed OPTIONS... [SCRIPT] [INPUTFILE...]

如果不指定INPUTFILE或者指定的INPUTFILE为"-"，sed将从标准输入中读取输入流并进行过滤。`SCRIPT`是第一个非选项参数，sed仅在没有使用"-e"或"-f script_file"选项时才将其当作是script部分而非输入文件。

可以使用下面的命令行选项来调用sed：

`'--version'`  
输出sed的版本号并退出。

`'--help'`  
输出sed命令行的简单帮助信息并退出。

`'-n'`  
`'--quiet'`  
`'--silent'`  
默认情况下，sed将在**每轮script循环结束**([How 'sed' works: Execution Cycle](#blog3.1))时自动输出模式空间中的内容。该选项禁止自动输出动作，这种情况下，只有显式通过"p"命令来产生对应的输出。

`'-e SCRIPT'`  
`'--expression=SCRIPT'`  
向SCRIPT中添加命令集，让sed根据这些命令集来处理输入流。(注：其实就是指定pattern和对应的command)

`'-f SCRIPT-FILE'`  
`'--file=SCRIPT-FILE'`  
指定包含command集合的script文件，让sed根据script文件中的命令集处理输入流。

`'-i[SUFFIX]'`  
`'--in-place[=SUFFIX]'`  
该选项指定要将sed的输出结果保存(覆盖的方式)到当前编辑的文件中。GNU sed是通过创建一个临时文件并将输入写入到该临时文件，然后重命名为源文件来实现的。

该选项隐含了"-s"选项。

当到达了文件的尾部，临时文件被重命名为**源文件**的名称。如果还提供了SUFFIX，则在重命名临时文件之前，先使用该SUFFIX先修改源文件名，从而生成一个文件备份。  
(注：**临时文件总是会被重命名为源文件名称**，也就是说sed处理完全结束后，仍使用源文件名的文件是sed修改后的文件。文件名中包含了SUFFIX的文件则是最原始文件的备份。例如源文件为a.txt，`sed -i'.log' SCRIPT a.txt`将生成两个文件：a.txt和a.txt.log，前者是sed修改后的文件，a.txt.log是源a.txt的备份文件。)

规则如下：如果扩展名不包含符号`*`，将SUFFIX被添加到原文件名的后面当作文件后缀；如果SUFFIX中包含了一个或多个字符`*`，则每个`*`都替换为原文件名。这使得你可以为备份文件添加一个前缀，而不是后缀，甚至可以将此备份文件放在在其他已存在的目录下。

如果没有提供SUFFIX，源文件被覆盖，且不会生成备份文件。

`'-l N'`  
`'--line-length=N'`  
为"l"命令指定默认的换行长度。N=0意味着完全不换行的长行，如果不指定，则70个字符就换行。

`'--posix'`  
GNU 'sed' includes several extensions to POSIX sed.  In order to simplify writing portable scripts, this option disables all the extensions that this manual documents, including additional commands.  Most of the extensions accept 'sed' programs that are outside the syntax mandated by POSIX, but some of them (such as the behavior of the 'N' command described in *note Reporting Bugs::) actually violate the standard.  If you want to disable only the latter kind of extension, you can set the 'POSIXLY_CORRECT' variable to a non-empty value.

`'-b'`  
`'--binary'`  
This option is available on every platform, but is only effective where the operating system makes a distinction between text files and binary files.  When such a distinction is made--as is the case for MS-DOS, Windows, Cygwin--text files are composed of lines separated by a carriage return _and_ a line feed character, and 'sed' does not see the ending CR.  When this option is specified, 'sed' will open input files in binary mode, thus not requesting this special processing and considering lines to end at a line feed.

`'--follow-symlinks'`  
该选项只在支持符号连接的操作系统上生效，且只有指定了"-i"选项时才生效。指定该选项后，如果sed命令行中指定的输入文件是一个符号连接，则sed将对该符号链接的目标文件进行处理。默认情况下，禁用该选项，因此不会修改链接的源文件。

`'-r'`  
`'--regexp-extended'`  
使用扩展正则表达式，而不是使用默认的基础正则表达式。sed所支持的扩展正则表达式和`egrep`一样，使用扩展正则表达式显得更简洁，因为有些元字符不用再使用反斜线`\`，但这是GNU扩展功能，因此应避免在可移植性脚本中使用。

`'-s'`  
`'--separate'`  
默认情况下，sed会将命令行中指定文件的所有行当作一个长输入流。此选项为GNU sed扩展功能，指定该选项后，sed将认为命令行中给定的每个文件都是独立的输入流。因此**此时范围定址(如`/abc/,/def/`)无法跨越多个文件，行号也会在处理每个文件时重置，`$`代表的是每个文件的最后一行，"R"命令调用的文件将绕回到每个文件的开头**。

`'-u'`  
`'--unbuffered'`  
使用尽量少的空间缓冲输入和输出行。(该选项在某些情况下尤为有用，例如输入流的来源是`"tail -f`时，指定该选项将可以尽快返回输出结果。)

`'-z'`  
`'--null-data'`  
`'--zero-terminated'`  
以空串符号"\0"而不是换行符"\n"作为输入流的行分隔符。

如果未给定"-e"、"-f"、"--expression"或"--file"选项，则命令行中的第一个非选项参数将被认为是SCRIPT被执行。

如果命令行中在上述选项后还加了任意参数，则都将被当作输入文件。其中"-"表示的是标准输入流。如果没有给定任何输入文件，则默认从标准输入中读取输入里流。

<span id="blog3"><span>
## 3. 'sed' Programs(sed程序)

sed程序由一个或多个sed命令组成(注：请勿将sed这个工具理解为sed命令，而应该看作是一个包含很多命令的程序，所以后文中"sed命令"表示的是sed中的子命令，而非sed本身这个命令)，这些命令通过"-e"、"-f file"传递，或者当没有指定这两个选项时，通过第一个非选项参数传递。这些传递的命令集合称为sed的执行脚本(SCRIPT)，命令的执行顺序按照命令行中给定的顺序依次执行。

在SCRIPT或SCRIPT-FILE中的多个命令可以使用分号";"进行分隔，或者换行书写。但有些命令，由于它们的语法问题，导致不支持使用分号作为命令分隔符，因此只能换行书写，除非它们是SCRIPT或SCRIPT-FILE中的最后一个命令。允许命令的前面有空白字符。

每个sed命令由地址或范围地址，随后紧跟单字符代表的命令名称组成，其中地址部分可以省略。


<a name="blog3.1"></a>
### 3.1 How 'sed' Works(sed是如何工作的)

sed维护两个数据缓冲空间：一直处于活动状态的模式空间(pattern space)和辅助性的保持空间(hold space)。这两个空间初始时都为空。

sed通过执行下面的循环来操作输入流中的每一行：
首先，sed读取输入流中的一行，移除该行的尾随换行符，并将其放入到pattern space中。然后对pattern space中的内容执行SCRIPT中的sed命令，每个sed命令都可以关联一个地址：地址是一种条件判断代码，只有符合条件的行才会执行相应的命令。当执行到SCRIPT的尾部时，除非指定了"-n"选项，否则pattern space中的内容将写到标准输出流中，并添加回尾随的换行符。然后进入下一个循环开始处理输入流中的下一行。

除非指定了特殊的命令(例如"-D")，否则pattern space中的内容在SCRIPT循环结束后会被删除。但是hold space会保留其中的内容(参见sed命令'h', 'H', 'x', 'g', 'G'以获知两个buffer空间是如何互相移动数据的)。

(

注：也就是说，sed程序工作时包含内外两个循环：内循环是SCRIPT循环，循环的过程是对模式空间中的内容执行SCRIPT中的命令集合，还包括一个隐含的自动输出动作用于输出模式空间的内容到标准输出中；外循环是sed循环，读取下一行到模式空间中，并执行SCRIPT循环，SCRIPT循环结束后再读取下一行到模式空间中。使用编程结构描述，结构如下：
```
for ((line=1;line<=last_line_num;++line))
do
    read $line to pattern_space;
    while pattern_space is not null
    do
        execute cmd1 in SCRIPT;
        execute cmd2 in SCRIPT;
        ……
        auto_print;
        remove_pattern_space;
    done
done
```
一般情况下，SCRIPT循环都只执行一轮就退出并进入外层sed循环，因为执行完一次SCRIPT循环，pattern space就清空了，但有些特殊命令(如"D"命令)会进入多行模式，使得SCRIPT循环结束时将数据锁在pattern space中不输出也不清空，甚至在SCRIPT循环还没结束时就强行进入下一轮SCRIPT循环，在《sed & awk》一书中，这种行为被称之为"回到SCRIPT的顶端"。其实就相当于在上面的while循环结构中加上了"continue"关键字。此外还有命令(如"d")可以直接退出SCRIPT循环进入下一个sed循环，就像是在while循环中加上了"break"一样。甚至还有直接退出sed循环的命令(只有2个这样的命令："q"和"Q")，就像是加上了"exit"一样。

`auto_print`和`remove_pattern_space`动作是隐含动作，分别称之为自动输出和清空pattern space。"-n"选项禁用的自动输出不是禁止隐含动作auto_print，而是让其输出空内容(它们是有区别的)，并执行remove_pattern_space。只要没有指定"-n"选项，在SCRIPT循环结束时一定输出pattern space中的全部内容并清空pattern space，只不过某些特殊的命令可以将pattern space中的内容锁住使auto_print输出空内容，也使得remove_pattern_space移除不了pattern space的内容。  
之所以要特地指出"输出空内容"并标明它和禁止输出动作，是因为某些命令(如"a"、"i"、"c"命令)依赖于输出流，没有输出动作就没有输出流，这些命令就无法正确完成。

这几个循环、几个动作的细节非常重要，虽不影响后文单个选项或命令的理解，但如果将命令结合，有时候的结果很可能会出人意料，而sed难就难在命令的合理组合。例如下面的命令，"a"命令本来是要将xyz插入到aaa所在行后面的，但结果却插在了aaa行的前面。
```
echo -e "aaa\nbbb" | sed '/aaa/{a\
> xyz
> ;N}'
xyz
aaa
bbb
```
)

<a name="blog3.2"></a>
### 3.2 Selecting lines with 'sed'(sed定址：筛选行的方式)

(注：sed SCRIPT中的命令由单字符代表的命令和地址组成，地址的作用是匹配当前正在处理的行，如果能匹配成功，就执行与该地址相关联的命令。地址由定址表达式决定，定址的结果可能是单行，可能是一个范围，省略定址表达式时表示所有行。)

可以使用下面任意一种方式进行定址：

`'NUMBER'`  
指定一个行号，sed将仅只匹配该行。(需要注意，除非使用了"-s"或"-i"选项，sed将对所有输入文件的行连续计数。)

`'FIRST~STEP'`  
这是GNU sed的功能，FIRST和STEP都是非负数。该定址表达式表示从第FIRST行开始，每隔STEP行就再取一次。也就是取行号满足`FIRST+(N*STEP)` (其中N>=0)的行。因此，要选择所有的奇数行，使用`1~2`；要从第2行开始每隔3行取一次，使用`2~3`；要从第10行开始每隔5行取一次，使用`10~5`；而`50~0`则表示只取第50行。

`'$'`  
该符号匹配的是最后一个文件的最后一行，如果指定了"-i"或"-s"，则匹配的是每个文件的最后一行。  
(注：总之，`$`匹配的是每个输入流的最后一行)

`'/REGEXP/'`  
该定址表达式将选择那些能被正则表达式REGEXP匹配的所有行。如果REGEXP中自身包含了字符"/"，则必须使用反斜线进行转义，即`"\/"`。

空的正则表达式"//"表示引用最后一个正则表达式匹配的结果(命令"s"中的正则匹配部分如果是空正则"//"，则一样如此处理)。注意，正则表达式的修饰符(即下文中的"I"和"M"，以及s命令中的修饰符"i"、"I"和"M")是在正则表达式编译完成之后(注：进行数据匹配时)才生效的，因此，这些修饰符和空正则一起使用时无效(注：会报错，提示"cannot specify modifiers on empty regexp")。

(注：这里的修饰符特指定址时可用的修饰符，即I和M。命令s也有修饰符"i"、"I"和"M"。  
如：`sed '/hold/Is//gogogo/g'`能成功，因为第一个定址正则表达式的修饰符"I"的对象是非空集"hold"，第二个正则模式"//"没有指定"i"、"I"或"M"修饰符，所以成功。但`sed '/hold/Is//gogogo/gi'`会报错，因为在正则编译结束后还未开始进行匹配的时候，第二个正则表达式中的修饰符"i"的对象是空集。`sed '/hold/I{//Mp}'`和`sed '/hold/I{//Ip}'`也都是失败的，原因都是第二个正则的修饰符对象是空集。

`'\%REGEXP%'`  
('%'可以使用其他任意单个字符替换。)

这和上一个定址表达式的作用是一样的，只不过是使用符号"%"替换了符号"/"。当REGEXP中包含"/"符号时，使用该定址表达式就无需对"/"使用反斜线`\`转义。但如果此时REGEXP中包含了"%"符号时，该符号需要使用`\`转义。  
总之，定址表达式中使用的分隔符在REGEXP中出现时都需要使用反斜线转义。

`'/REGEXP/I'`  
`'\%REGEXP%I'`  
正则表达式的修饰符"I"是GNU的扩展功能，表示REGEXP在匹配时不区分大小写。

<a name="M_modifier"></a>
`'/REGEXP/M'`  
`'\%REGEXP%M'`  
正则表达式的修饰符"M"是GNU的扩展功能，这可以让sed直接匹配多行模式下(multi-line mode)的位置。该修饰符使得正则表达式的元字符"^"和"$"匹配分别匹配每个新行后的空字符和新行前的空字符。还有另外两个特殊的字符序列(`` \` ``和`\'`，分别是反斜线加反引号，反斜线加单引号)，它们总是匹配buffer空间中的起始和结束位置。此外，元字符点"."不能匹配多行模式下的换行符。  
(注：在单行模式下，使用M和不使用M是没有区别的，但在多行模式下各符号匹配的位置将和通常的正则表达式元字符匹配的内容不同，各符号的匹配位置如下图所示)  
![](/img/shell/733013-20170905100348101-2017368622.png)


如果没有给定地址，则会匹配输入流中的所有行，如果给定了单地址的定址表达式，则这些行会在模式空间中被匹配条件进行匹配。

可以使用逗号分隔的两个地址表达式(ADDR1,ADDR2)来描述范围地址。范围地址表示从符合第一个地址条件的行开始匹配，直到符合第二个地址的行结束。

(注：需要注意，无论行是否符合地址匹配条件，它们都会被读入pattern space，只不过读入的行如果不符合地址匹配条件，将直接执行SCRIPT循环中的隐含动作，并结束SCRIPT循环继续读取输入流的下一行)

如果第二个定址表达式是正则表达式REGEXP，将从符合第一个定址表达式的行开始逐行检查，以匹配范围定址的结束行：范围定址的范围总是会跨越至少两行(当然，如果输入流结束的情形除外)。

如果第二个定址表达式是一个小于或等于第一个表达式代表的行号值，则只会匹配第一个表达式搜索到的那一行。  
(注：即如果范围地址的结束行在起始行的前面，则只会匹配起始行。)

GNU sed还支持以下几种特殊的两地址格式，这些都是GNU的扩展功能：

`'0,/REGEXP/'`  

使用行号0作为起始地址也是支持的，就像此处的`0,/REGEXP/`，这时sed会尝试对第一行就匹配REGEXP。换句话说，`0,/REGEXP/`和`1,/REGEXP/`基本是相同的。但以行号0作为起始行时，如果第一行就能被ADDR2匹配，范围搜索立即就结束，因为第二个地址已经搜索到了；如果以行号1作为起始行，会从第二行开始匹配ADDR2，直到匹配成功。

注意，0地址只有在这一种情况下才是有意义的，实际上并没有第0行，在其他任何时候使用行号0都会给出错误提示。

`'ADDR1,+N'`  
匹配ADDR1和其后的N行。

`'ADDR1,~N'`  
匹配ADDR1和其后的行直到出现N的倍数行，倍数可为随意整数倍。
(注：可以是任意倍，只要N的倍数是最接近且大于ADDR1的即可。如`ADDR1=1,N=3`匹配1到3行，`ADDR1=5,N=4`匹配5-8行。而`1,+3`匹配的是第一行和其后的3行即1-4行。)

在地址的后面追加"!"符号指定反转匹配的含义。意思是，如果"!"跟在范围定址的后面，则那些匹配的行将不被选择，而是不匹配的行被选择。同样可以适用于单个定址甚至于有悖常理的空定址。

(注：例如，`sed -n '3!p' INPUTFILE`，`sed -n '3,5!p' INPUTFILE`，甚至是`sed -n '!p' INPUTFILE`)


(注：

- **sed采用计数器计算，每读取一行，计数器加1，直到最后一行。因此在读到最后一行前，sed是不知道这次输入流中总共有多上行，也不知道最后一行是第几行。`$`符号表示最后一行，它只是一个特殊的标识符号。当sed读取到输入流的尾部时，sed就会为该行打上该标记。无法使用`$`参与数学运算，例如无法使用`$-1`表示倒数第二行，因为sed在读到最后一行前，它并不知道这是倒数第二行，此时也还没打`$`标记，因此`$-1`是错误的定址表达式。另一方面，在本译文的最开始就说明了，sed只能通过一次输入流，这意味着已经读取过的行无法再次被读取，所以sed不提供往回取数据的定址表达式，上面的几个定址表达式也确实证明了这一点**。事实上，sed中根本就无法使用"-"做减法或取负数，因为语法不支持。

- 范围匹配的是pattern space中的内容，对于`regexp1,regexp2`这样的范围定址自然容易理解，但如果是`num1,num2`这样的范围定址，它可能和想象中的匹配方式不一样。**每读取一行，sed内部的行号计数器都会"+1"，所以行号值总是记录在内存中。当进行行号匹配时，其本质是和内存中当前计数器的值进行匹配，如果当前计数器的值在范围内，则能被匹配，否则无法匹配**。因此，从hold space回到pattern space的行或者被修改的行甚至是被删除过的行，不管这一行的内容在pattern space中是否存在，只要当前计数器值在范围内，就能匹配。例如多行模式：

```
echo -e "abc\nfgh" | sed -n 'N;1p'
```

该命令不输出任何内容。虽然N读取下一行后，pattern space两行中的第一行一直原封不动的放在那，没做任何处理，但"1p"根本就无法匹配这一行，因为当前计数器的值为2。

)。


<a name="blog3.3"></a>
### 3.3 Overview of Regular Expression Syntax(正则表达式一览)

(注：sed解析、编译和匹配正则表达式的引擎和grep的引擎一样，因此本文基本不做翻译，可以参考我对info grep的译文：[grep命令中文手册](https://www.junmajinlong.com/shell/grep_translate)。)

To know how to use 'sed', people should understand regular expressions ("regexp" for short).  A regular expression is a pattern that is matched against a subject string from left to right.  Most characters are "ordinary": they stand for themselves in a pattern, and match the corresponding characters in the subject.  As a trivial example, the
pattern

    The quick brown fox

matches a portion of a subject string that is identical to itself.  The power of regular expressions comes from the ability to include alternatives and repetitions in the pattern.  These are encoded in the pattern by the use of "special characters", which do not stand for themselves but instead are interpreted in some special way.  Here is a brief description of regular expression syntax as used in 'sed'.

`'CHAR'`  
A single ordinary character matches itself.

`'*'`  
Matches a sequence of zero or more instances of matches for the preceding regular expression, which must be an ordinary character, a special character preceded by '\', a '.', a grouped regexp (see below), or a bracket expression.  As a GNU extension, a postfixed regular expression can also be followed by '\*'; for example, 'a\*\*' is equivalent to 'a\*'.  POSIX 1003.1-2001 says that '\*' stands for itself when it appears at the start of a regular expression or subexpression, but many nonGNU implementations do not support this and portable scripts should instead use '\\\*' in these contexts.

`'\+'`  
As '*', but matches one or more.  It is a GNU extension.

`'\?'`  
As '*', but only matches zero or one.  It is a GNU extension.

`'\{I\}'`  
As '*', but matches exactly I sequences (I is a decimal integer; for portability, keep it between 0 and 255 inclusive).

`'\{I,J\}'`  
Matches between I and J, inclusive, sequences.

`'\{I,\}'`  
Matches more than or equal to I sequences.

`'\(REGEXP\)'`  
Groups the inner REGEXP as a whole, this is used to:

- Apply postfix operators, like `'\(abcd\)*'`: this will search for zero or more whole sequences of 'abcd', while `'abcd*'` would search for 'abc' followed by zero or more occurrences of 'd'.  Note that support for `'\(abcd\)*'` is required by POSIX 1003.1-2001, but many non-GNU implementations do not support it and hence it is not universally portable.

- Use back references (see below).

`'.'`  
Matches any character, including newline.

`'^'`  
Matches the null string at beginning of the pattern space, i.e. what appears after the circumflex must appear at the beginning of the pattern space.

In most scripts, pattern space is initialized to the content of each line (\*note How 'sed' works: Execution Cycle.).  So, it is a useful simplification to think of '^#include' as matching only lines where '#include' is the first thing on line--if there are spaces before, for example, the match fails.  This simplification is valid as long as the original content of pattern space is not modified, for example with an 's' command.

'^' acts as a special character only at the beginning of the regular expression or subexpression (that is, after '\(' or '\|'). Portable scripts should avoid '^' at the beginning of a subexpression, though, as POSIX allows implementations that treat '^' as an ordinary character in that context.

`'$'`  
 It is the same as '^', but refers to end of pattern space.  '$' also acts as a special character only at the end of the regular expression or subexpression (that is, before '\)' or '\|'), and its use at the end of a subexpression is not portable.

`'[LIST]'`  
`'[^LIST]'`  
Matches any single character in LIST: for example, '[aeiou]' matches all vowels.  A list may include sequences like 'CHAR1-CHAR2', which matches any character between (inclusive) CHAR1 and CHAR2.

 A leading '^' reverses the meaning of LIST, so that it matches any single character _not_ in LIST.  To include ']' in the list, make it the first character (after the '^' if needed), to include '-' in the list, make it the first or last; to include '^' put it after the first character.

 The characters `'$', '*', '.', '[', and '\'` are normally not special within LIST.  For example, `'[\*]'` matches either `'\' or '*'`, because the '\' is not special here.  However, strings like '[.ch.]', '[=a=]', and '[:space:]' are special within LIST and represent collating symbols, equivalence classes, and character classes, respectively, and '[' is therefore special within LIST when it is followed by '.', '=', or ':'.  Also, when not in 'POSIXLY_CORRECT' mode, special escapes like '\n' and '\t' are recognized within LIST.  \*Note Escapes::.

`'REGEXP1\|REGEXP2'`  
Matches either REGEXP1 or REGEXP2.  Use parentheses to use complex alternative regular expressions.  The matching process tries each alternative in turn, from left to right, and the first one that succeeds is used.  It is a GNU extension.

`'REGEXP1REGEXP2'`  
Matches the concatenation of REGEXP1 and REGEXP2.  Concatenation binds more tightly than '\|', '^', and '$', but less tightly than the other regular expression operators.

`'\DIGIT'`  
Matches the DIGIT-th '\(...\)' parenthesized subexpression in the regular expression.  This is called a "back reference". Subexpressions are implicity numbered by counting occurrences of '\(' left-to-right.

`'\n'`  
匹配换行符。

`'\CHAR'`  
Matches CHAR, where CHAR is one of `'$', '*', '.', '[', '\', or '^'`. Note that the only C-like backslash sequences that you can portably assume to be interpreted are '\n' and `'\\'`; in particular '\t' is not portable, and matches a 't' under most implementations of 'sed', rather than a tab character.


注意，sed支持的正则表达式是贪婪匹配的。例如，从行左向行右匹配时，如果满足匹配的匹配字符有多个，则会取最长的。(注：例如字符串'abbbc'，给定正则表达式`ab\*`，满足该正则的字符串序列有"a"，"ab"，"abb"以及"abbb"，由于是贪婪匹配，它会取最长的，即"abbb")

示例：

`'abcdef'`  
匹配字符串"abcdef"。

`'a*b'`  
匹配0或多个a，但后面需要跟上字符b，例如'b'或'aaaaab'。

`'a\?b'`  
匹配"b"或"ab"。

`'a\+b\+'`  
匹配一个或多个a，且后面需要有一个或多个b，所以"ab"是最短的匹配序列，其他的如'aaaab'、'abbbbb'或'aaaaaabbbbbbb'都能被匹配。

`'.*'`  
`'.\+'`  
都匹配所有字符；但是，第一个匹配任何字串(包含空字串)，而第二个只匹配包含至少一个字符的字串(空格也是字符)。  
(注：即第一种会匹配所有，包括null字符，第二种不能匹配null字串。例如：`sed -n '/.*/p'`会匹配空行。`sed -n '/.\+/p'`不会匹配空行，这是匹配非空行的一种简便方法。)

`'^main.*(.*)'`  
匹配以"main"开头的行，且该行还包括括号。

`'^#'`  
匹配以"#"开头的行。

`'\\$'`  
匹配以反斜线结尾的行。

`'\$'`  
匹配包含美元符号的行。

`'[a-zA-Z0-9]'`  
在C字符集环境下，这匹配任意ASCII的字母和数字。

`'[^ tab]\+'`  
(此处tab代表一个制表符)匹配不包含空格和制表符的行。通常用于匹配整个单词。

`'^\(.*\)\n\1$'`  
匹配相邻两行完全相同的情况。

`'.\{9\}A$'`  
匹配9个任意字符，后面还带一个字母A。

`'^.\{15\}A'`  
匹配以任意15个字符开头但第16个字符是A的行。

<a name="blog3.4"></a>
### 3.4 Often-Used Commands(sed常用命令)

如果要掌握sed，下面几个命令几乎是必须掌握的。

`'#'`  
不能跟在定址表达式后。

以`#`开头的行表示注释行。

如果要考虑可移植性，需要注意有些sed可能仅支持一条单行注释，且此时`#`必须是SCRIPT中的第一个字符。

注意：如果SCRIPT中的前两个字符是`#n`，则表示强制开启选项"-n"，而非注释。如果想要开启一个注释行来注释以字母"n"开头的字符串，那么需要使用大写字母N代替，或者`#`和"n"之间加上一个或多个空白字符。

`'q [EXIT-CODE]'`  
该命令只能在单定址表达式下使用。

表示立即退出sed，而不再处理后续的命令和输入流。注意，退出sed前，当前pattern space中的内容会被输出，除非使用了"-n"选项。返回一个退出状态码是GNU sed的扩展功能。(注：后文还有一个"Q"命令，和"q"命令作用相同，但退出sed前不输出当前pattern space中的内容。)

`'d'`  
删除模式空间中的内容，并立即进入下一个sed循环(注：即以"break"的方式直接退出当前SCRIPT循环，并读取输入流中的下一行，再执行SCRIPT循环)。


`'p'`  
输出当前模式空间的内容(到标准输出中)。该命令一般只和"-n"选项一起使用。

`'n'`  
输出模式空间的内容(除非自动输出功能被禁止)，然后读取下一行到模式空间中替换其中的内容。如果输入流中没有下一行供"n"读取(注：即此前已经读取了输入流的最后一行)，sed将直接退出，不会执行SCRIPT中后续的命令。

(

注：以下面两个命令为例：其中"i"命令为插入字符串并输出。

```
[root@xuexi ~]# echo -e "line1\nline2\nline3\nline4" | sed -n 'n;p;i aaa'
line2
aaa
line4
aaa
[root@xuexi ~]# echo -e "line1\nline2\nline3" | sed -n 'n;p;i aaa'
line2
aaa
```

第二个命令读取了第三行到pattern space后执行n命令，但此时输入流中已经没有下一行供读取，于是直接结束，链后续的"p"命令都没有执行。

)

`'{ COMMANDS }'`  
使用大括号将一系列命令组合成一个命令组。在某些时候特别有用，例如相对某一匹配行或某一范围中的行做多次处理时。

<a name="blog3.5"></a>
### 3.5 The 's' Command(sed的s命令)

sed的"s"命令(该命令用于字符串替换)的语法格式为`"s/REGEXP/REPLACEMENT/FLAGS"`。"s"命令中的字符"/"可以统一被替换成任意其他单个字符(注：在定址正则表达式中斜杠也可以被替换成其他字符，但需要在第一个被替换字符前加上反斜线转义，例如`\%REGEXP%`，而s命令中替换时无需加反斜线)。如果斜线字符"/"(或被替换后的其他字符)需要出现在REGEXP或REPLACEMENT中，必须使用反斜线进行转义。

sed的"s"命令算的上是sed程序中最重要的命令，它有很多不同的选项。但它的基本概念很简单："s"命令使用REGEXP匹配pattern space中的内容，如果匹配成功，则匹配成功的那部分字符串被替换为REPLACEMENT。

REPLACEMENT中可以使用`"\N"`(N是从1到9的整数)进行后向引用，所代表的是REGEXP第N个括号`\(...\)`中匹配的内容。另外，REPLACEMENT中可以包含未转义的"&"符号，这表示引用pattern space中被匹配的整个内容(注：是pattern space中的所有匹配，不仅仅只是括号的分组匹配)。最后，GNU sed为REPLACEMENT还提供了一些"反斜线加单字母"的特殊字符序列，如下：

`'\L'`  
将REPLACEMENTE转换成小写，直到遇到了`\U`或`\E`。

`'\l'`  
将REPLACEMENTE中下一个字符转换成小写。

`'\U'`  
将REPLACEMENTE转换成大写，直到遇到了`\L`或`\E`。

`'\u'`  
将REPLACEMENT中下一个字符转换成大写。

`'\E'`  
停止`\L`和`\E`开启的大小写转换。

(

注：使用方法如下示例

    shell> echo hello | sed 's/e/ \Uyour\Lname /'
    h YOURname llo

)

当使用了"g"修饰符时，大小写转换不会从一个正则匹配事件扩展到另一个正则匹配事件。例如，如果pattern space中的内容为"a-b-"，执行下面的命令：

```
 s/\(b\?\)-/x\u\1/g
```

得到的输出为"axxB"。当替换第一个"-"时，`"\u"`只影响`"\1"`代表的空值。当替换"b-"时，由于`"\1"`代表的是`b-`，所以第一个字符"b"会被替换为"B"，但添加到pattern space中的"x"仍然为小写。

另一方面，`"\l"`和`"\u"`会影响REPLACEMENT中的空引用的后一个字符。例如：模式空间内容为`"a-b-"`，执行下面的命令：

```
 s/\(b\?\)-/\u\1x/g
```

将使用"X"(大写)替换"-"，使用"Bx"替换`b-`。如果这样的结果不是你想要的结果，可以在此例中的`"\1"`后加上`"\E"`防止"x"被转换。

如果想要在最后的REPLACEMENT中包含字面符号`"\"`、`"&"`或换行符，需要在REPLACEMENT中这些字符之前加上反斜线。

sed的"s"命令可以接0或多个如下列出的修饰符(FLAGS)：

`'g'`  
使用REPLACEMENT替换所有被REGEXP匹配的内容，而非第一次被匹配到的。

(注：**sed任何动作都是在pattern space进行的，字符串替换也如此。**不加"g"修饰符时，将只pattern space中第一个被匹配到的内容，加上"g"，将替换pattern space中所有被匹配到的内容，因此如果是多行工作模式，第二行或更多行只要能被匹配都会被替换。)


`'NUMBER'`  
为一个整数值N。表示只替换第N个被匹配到的内容。

注意：POSIX标准中没有说明既指定"g"又指定"NUMBER"修饰符时会如何处理，并且当前各种sed的实现也没有标准说法。对于GNU sed而言，定义如下：忽略第NUMBER个匹配前的所有匹配，然后从第NUMBER个匹配开始向后重新匹配并替换所有匹配成功的内容。

`'p'`  
替换动作完成后打印新的模式空间中的内容。

注意：当既指定"p"又指定"e"命令时，它们的前后顺序不同，会得到两种不同结果。一般来说，`"ep"`这种顺序得到的结果可能是所期望的结果，但另一种顺序`"pe"`对调试很有用。出于这个原因，当前版本的GNU sed特地解释了"p"命令在"e"命令前(或后)时，将在"e"命令生效前(或后)输出内容，因为"s"命令的每个修饰符一般都只展示一次结果。虽然这种行为已经写入文档，但未来可能会改变。

`'w FILE-NAME'`  
该子命令表示将模式空间的内容写入到指定的文件FILE-NAME中。

GNU sed支持两个特殊的FILE-NAME：`"/dev/stderr"`和`"/dev/stdout"`，分别表示写入到标准错误和标准输出中。

`'e'`  
该命令允许通过管道将shell命令的执行结果直接传递到pattern space中。当替换动作完成后，将搜索pattern space，发现的命令会被执行，并且执行结果覆盖到当前pattern space中。它会禁用尾随换行符，并且如果待执行命令中包含了NULL字符，该命令将不被定义为命令，即表示它是普通字符而不是命令所以不执行。这是GNU sed的功能。

(

注："s"命令的"e"修饰符似乎只有搜索pattern space并找出其中的命令并执行的功能，没有选项描述中第一句话所述的功能。但"e"命令有该功能，例如：

```
echo -e "good\nbad" | sed 'e echo haha'
haha
good
haha
bad
```

不讨论e命令的用法，关于"s"命令的"e"修饰符的用法如下：

文件ttt.txt的内容如下：

```
[root@xuexi tmp]# cat ttt.txt 
ls /tmp
haha
excute a command
```

将第二行的"haha"替换成一个命令后使用"e"修饰符，那么在替换完成后会查找模式空间，如果找到了可以执行的命令就执行。

```
[root@xuexi tmp]# sed 's/haha/du -sh \/tmp/ge' ttt.txt 
ls /tmp        # 注意这一行没有执行
18M     /tmp   # 说明执行了du -sh /tmp的命令
excute a command
```

注意到第一行虽然也是可以执行的命令，但是却没有执行，因为"e"是"s"命令的修饰符，需要成功匹配"haha"的行才能符合"e"修饰符。  
注意模式空间中的内容是一行一行的，命令将把整行内容作为命令行。例如，如果只将excute替换成`du -sh /tmp`，那么模式空间中的内容将是`du -sh /tmp a command`，它会对/tmp和当前目录下的"a和"command"目录进行`du -sh`统计，但很可能"a"或"command"目录根本不存在，这时就会报错。

```bash
[root@xuexi tmp]# sed 's%excute%du -sh /tmp%ge' ttt.txt
ls /tmp
haha
du: cannot access 'command': No such file or directory  # 不存在command目录所以错误
18M      /tmp
4.0K     a      # 当前目录下正好存在a目录，所以统计了信息
```

并且如果替换后找到的命令不在行首(有前导空白没有影响)，将出现错误，因为是将整行作为命令来执行的。

```bash
[root@xuexi tmp]# sed 's%command%du -sh /tmp%ge' ttt.txt         
ls /tmp
haha
sh: excute: command not found
```

所以更保险的方法是对整行进行替换。

```bash
[root@xuexi tmp]# sed 's%^excute.*%du -sh /tmp%ge' ttt.txt 
ls /tmp
haha
18M     /tmp
```

当然，如果想要执行第一行的`ls /tmp`命令也很简单，只需匹配这一行。也可以在此命令上进行命令扩展。如下面将/tmp替换后，模式空间的内容是`ls -ld /tmp;du -sh /tmp`，所以会执行这两条命令。

```bash
[root@xuexi tmp]# sed 's!/tmp!-ld /tmp;du -sh /tmp!ge' ttt.txt       
dr-xr-xr-x. 17 root root 8736768 Oct 25 14:11 /tmp
18M     /tmp
haha
excute a command
```

)

`'I'`  
`'i'`  
这两个修饰符作用相同，表示在REGEXP匹配的时候忽略大小写。这是GNU扩展功能。

`'M'`  
`'m'`  
和前文定址表达式中所述的M修饰符作用一致。见[M修饰符](#M_modifier)。


(注：除了上述明确的修饰符外，还有一个特殊的不算修饰符的符号`\`，当它写在"s"命令的REPLACEMENT的每个行尾时，表示新转义当前命令行的行尾，也就是开启一个新行。例如：

```bash
echo 'abcdef' | sed 's/^.*$/\
&\
/'
```

这表示在abcdef这一行内容前后分别加一个空行，也就是将abcdef这一行嵌入到空行中间。所以可将其理解为换行符`\n`，但必须注意，如果转义新行后有内容，则此新行必须再次使用反斜线终止该行，因为它的行尾已经被转义了。例如：

```bash
echo 'abcdef' | sed 's/^.*$/\
&\
xyz\
/'

echo 'abcdef' | sed 's/^.*$/&\
/'
```

其实个人觉得使用`\n`方便多了，即容易理解又容易控制。例如上面最后一个命令改为：`echo 'abcdef' | sed 's/^.*$/&\n/'`。在后文的好几个例子中都是用了转义行尾的技巧，所以此处提上一提。

)

<a name="blog3.6"></a>
### 3.6 Less Frequently-Used Commands(比较少用的sed命令)

虽然可能用的不如前面所述的命令频繁，但这些小命令有时候非常有用。

`'y/SOURCE-CHARS/DEST-CHARS/'`  
(y命令中的斜线"/"可以统一被替换成其他单个字符。)

转换pattern space中能被SOURCE-CHARS匹配的字符为DEST-CHARS，且是一一对应地转换。

斜线"/"(或被替换为的其他字符)、反斜线`\`和换行符可以出现在SOURCE-CHARS或DEST-CHARS中，但需要使用反斜线进行转义。(转义后的)SOURCE-CHARS和DEST-CHARS中字符的数量必须完全相同。

`'a\'`  
`'TEXT'`  
是GNU sed的扩展功能，该命令可以在两个地址格式的定址表达式后使用。

队列化该命令后的文本内容(最后一个反斜线`\`是文本结束符，在输出时该符号会被移除)，并在当前SCRIPT循环结束时输出，或从输入流中读取下一行时输出。(注：本质是：只要"有读取下一行"的动作就会触发队列化内容的输出，不止是"a"、还有"i"和"c"以及"r"等，另外，除了sed循环的第一个动作可以读取下一行，命令"N"和"n"都会读取下一行，因此它们也会触发这些队列化内容的输出)

输入的文本内容中如果要出现反斜线字符，需要进行转义，即`\\`。

作为一个GNU扩展功能，如果命令"a"和换行符之间存在非空白字符`\`序列，则此反斜线会开启新行，并且反斜线后的第一个非空白字符作为下一行的第一个字符。这同样适用于下面的"i"和"c"命令。

(注：**命令"a"为尾部追加插入，其动作是将输入文本队列化在内存中，它不会进入模式空间，而是隐含动作自动输出模式空间内容时，在半路追上stdout并将队列化的文本追加到其尾部，并同时输出。下面的"i"和"c"命令所操作都是stdout，都不会进入pattern space。也就是说，这3个命令操作的文本内容不受任何sed其他选项和命令控制。**如"-n"选项无法禁止该输入，因为"-n"禁止的是pattern space的自动输出，同时，如果SCRIPT中这3个命令后还有后续的命令序列，则这些命令也无法匹配和操作这些文本内容，例如"p"命令不会输出这些队列化文本，因为它不是pattern space中的内容。  
**这是sed程序中少见的几个在pattern space外处理数据的动作**。具体用法如下两个示例：

```bash
[root@xuexi tmp]# cat set2.txt
carrot
cookiee
gold
[root@xuexi tmp]# sed '/carrot/a\iron\nsteel' set2.txt  # 换行插入
carrot
iron
steel
cookiee
gold
[root@xuexi tmp]# sed '/carrot/a\iron\  # iron作为第一行，其后跟\继续输入下一行
> steel' set2.txt         # 直到遇到结束引号，队列化文本才结束
carrot
iron
steel
cookiee
gold
```

)

`'i\'`  
`'TEXT'`  
是GNU的扩展功能，该命令可以在两个地址格式的定址表达式后使用。

立即输出该命令后的文本(除了最后一行，每一行都以反斜线`\`结尾，但在输出时会自动移除)。

(注："i"命令同"a"命令一样，只不过是在stdout流的头部加上队列化的文本，并同时输出。它同样不进入pattern space，操作的也是stdout流)

`'c\'`  
`'TEXT'`  
删除从模式空间输出的内容，并在最后一行处(如果没有指定定址选项则每一行)输出"c"命令后队列化文本(除了最后一行，每行都以`\`结尾，输出时会移除)。该命令处理完后会开始新的sed循环，因为模式空间的内容将会被删除。  
(注："c"即change，表示使用队列化文本修改输出流，虽说是修改，但实际上是替换模式空间的输出流并输出。它和"i"、"a"两个命令不一样，它"冒充"了输出流输出后就立即进入下一个sed循环，使得SCRIPT中后续的命令不会执行，而"i"、"a"则不会退出循环，而是会继续回到SCRIPT循环中执行后续的命令)

(注：虽然这3个命令用的远不如"s"命令频繁，但我个人认为，理解了这3个命令，就理解了大半sed的工作机制。)

`'='`  
是GNU扩展功能，可以在两个地址格式的定址表达式后使用。

该命令用于输出当前正在处理的输入流中的行号(行号后会加上尾随换行符)。  
(注：这是将sed内部保存的行号计数器的值输出，是内存中的内容，因此不进入pattern space，不受"-n"选项影响。)

`'l N'`  
将模式空间中的内容以一种明确的形式打印：非打印字符（和`\`字符）被打印成C语言风格的转义形式；长行被分割后打印，使用`\`字符来表示分割的位置；每一行的结尾被标记上"$"符。

N指定了行分割时期望的行长度(即多少字符换行)，长度指定为0表示不换行。如果忽略N，则使用默认长度(默认70)。N参数是GNU sed的扩展功能。

`'r FILENAME'`  
GNU扩展功能，该命令接受两个地址的定址表达式。

读取filename中的内容并按行队列化，然后在当前SCRIPT循环的最后或者读取下一行前插入到output流中。注意，如果filename无法被读取，它将被当成一个空文件而不会产生任何错误提示。

作为GNU sed的扩展功能，支持特殊的filename值：/dev/stdin，表示从标准输入中读取内容。

(注：

- "r"命令的功能和"a"命令的作用是完全一样的，连工作机制都完全相同，除了"r"命令是从文件中读取队列化文本，而"a"读取的是命令行中提供的。  

- 从"r"命令之后到换行符或sed的引号之间的所有内容都作为"r"的文件参数。因此r命令之后如果还有命令，应当分行写。  

- "r"命令是一次队列化filename中的所有行，后文还有一个"R"命令是每次队列化filename中的一行，它们都受输出流的影响，只要有输出流就追加到输出流中，并开始下一轮的队列化过程。并非sed每从输入流中读取一行就队列化一次)

)

`'w FILENAME'`  
将模式空间的内容写入到filename中，作为一个GNU sed扩展，它支持两种特殊的filename值：/dev/stderr和/dev/stdout，分别表示将模式空间的内容写到标准错误和标准输出中。

在输入流的第一行被读入前，filename文件将被创建(或截断，即清空)；所有引用相同filename的"w"命令(包括替换命令"s"后的"w"修饰符)不会先关闭filename文件再打开该文件来写入。

(注：

- 即sed脚本中使用了"w"命令后该filename文件对sed而言一直是处于打开状态的，直到sed退出才关闭。多次使用引用相同文件的"w"命令会向该打开文件中写入，因此多个"w"写入filename时是追加写入的方式，但如果"w"打开的文件已存在，则会先截断该文件，即后续追加式的"w"输出会覆盖原文件。
- "w"命令不会影响pattern space的输出，它就像tee命令一样，一份输出到屏幕，一份重定向到filename中。

)

`'D'`  
如果pattern space中未包含换行符，则像"d"命令中提到的一样进入下一个sed循环。否则删除pattern space中第一个换行符之前的所有内容，并重新回到SCRIPT的顶端重新对pattern space中剩下的内容进行处理，而不会读取输入流中的下一行。

(注：换句话说，D命令总是删除pattern space中第一个换行符前的内容，如果删除后pattern space中还有内容剩下，则直接回到SCRIPT顶端重新对剩下的这部分内容执行命令，即以"continue"的方式强制进入下一个SCRIPT循环，如果pattern space中没有剩下内容，则直接退出当前SCRIPT循环，并进入下一个sed循环。)

`'N'`  
在当前pattern space中的尾部加上一个换行符，并读取输入流的下一行追加到换行符后面。如果输入流中没有下一行供"N"读取，则直接退出sed循环。

(注：无论是sed循环的第一步、还是n命令或是N命令，它们读取下一行到pattern space时总会移除行尾换行符。但N命令在读取下一行前在当前pattern space的尾部加上了一个`\n`，这使得pattern space中有了两行，如果一个SCRIPT中有多次N命令还可能有更多行。因此N命令是sed进入多行工作模式的方法。但要注意，虽然称为多行模式，但在pattern space中加入换行符并非真的换行，在pattern space中仍是一行显示的，因为它能被匹配到，且在计算字符数量时会被计入，它仅仅只是一个特殊符号，只不过在输出时会换行显示)

`'P'`  
输出pattern space中第一个换行符`\n`前面的内容。

(注：即输出pattern space中的第一行)

`'h'`  
使用pattern space中的内容替换hold space中的内容。
(注：替换后，pattern space内容仍然保留，不会被清空)

`'H'`  
在hold space的尾部追加一个换行符，并将pattern space中的内容追加到hold space的换行符尾部。  
(注：追加后，pattern space内容仍然保留，不会被清空)

`'g'`  
使用hold space中的内容替换pattern space中的内容。

`'G'`  
在当前pattern space的尾部追加一个换行符，并将hold space中的内容追加到pattern space的换行符尾部。  
(注：这是另一个进入多行模式空间的方法。这就有了一个特殊的用法，`sed 'G' filename`可以在filename的每一行后添加一个空行，这也是很多人产生误解的地方，认为hold space的初始内容为空行，追加到pattern space就成了空行。其实并非如此，因为无论是pattern space还是hold space在初始时都是空的，G命令在pattern space的尾部加上了换行符，并将hold space中的空内容追加到换行符后，使得这成了一个空行。)

`'x'`  
交换pattern space和hold space的内容。


<a name="blog3.7"></a>
### 3.7 Commands for 'sed' gurus(大师级的sed命令)

在大多数情况下，使用这些命令意味着你可能可以使用像"awk"或"perl"等的工具更好地达到目的。但是偶尔有人会坚持使用'sed'，这些命令可能会使得脚本变得非常复杂。

(注：虽说标签功能很少使用，但有时候确实非常有用，它在sed脚本内部实现了简单的编程结构体)

`':LABEL'`  
不可接在定址表达式后。

设置一个分支标签LABEL供"b"、"t"或"T"(注："T"命令在后文的[GNU sed扩展](#blog3.8)中说明)命令跳转。除此功能，无任何作用。  
(注：冒号和LABEL之间不能有空格，虽然原文中有空格，但这是错误的)

`'b LABEL'`  
无条件跳转到标签LABEL上。如果LABEL参数省略，则跳转到SCRIPT的尾部准备进入下一个sed循环，即跳转到隐含动作`auto_print`的前面。

`'t LABEL'`  
如果对最近读入的行有"s"命令的替换成功了，则跳转到LABEL标签处。如果LABEL参数省略，则跳转到SCRIPT的尾部准备进入下一个sed循环，即跳转到隐含动作`auto_print`的前面。(注：该命令和"T LABEL"相反，"T"是在"s"替换不成功时跳转到指定标签。)

(注：另外，"t"的跳转条件是有"s"命令替换成功，不是最近一个"s"命令替换成功，所以如果有多个"s"命令，即使"t"命令最近的"s"命令不成功，则仍会跳转，直到一个跳转循环内都没有替换成功的才终止跳转。例如：

```bash
`echo 'abcdef' | sed ':x;s/haha/yyy/;s/def/haha/;s/yyy/zzz/;tx'`
abczzz
```

第一轮，被读取的输入行在第一个"s"和第3个"s"命令失败，但第二个成功，所以仍会执行跳转，于是进入第二轮。在第二轮，第一个"s"和第三个"s"替换成功，所以继续跳转，于是进入第三轮。在第三轮中，所有"s"都失败，所以不再跳转。

)

(注：篇幅原因，使用一个示例简单解释一下标签和跳转命令的使用。  
1. 使用":LABEL"设置好标签后，就可以使用"b LABEL"、"t LABEL"或"T LABEL"命令跳转到该标签。  
2. 跳转到某标签的意思是开始执行该标签后的命令。  
3. 如果"b"、"t"或"T"命令省略了参数LABEL，则跳转到隐藏标签，即自动输出pattern space动作`auto_print`的前面。  

假设有如下test.txt文件，该文件中空白处都是一个个的空格。目的是多个连续的空格替换成圆圈"○"，有几个空格就替换成几个圆圈，但初始时只有一个空格的不进行替换(即第一行的第一个空格和最后一行的最后一个空格不替换)。

```bash
[root@xuexi ~]# cat test.txt
 sleep       sleep
  sleep      sleep
   sleep     sleep
    sleep    sleep
     sleep   sleep
      sleep  sleep
       sleep sleep
```

可以使用下面简单的命令来实现：

```bash
[root@xuexi ~]# sed 's/  /○○/g;s/○ /○○/g' test.txt
 sleep○○○○○○○sleep
○○sleep○○○○○○sleep
○○○sleep○○○○○sleep
○○○○sleep○○○○sleep
○○○○○sleep○○○sleep
○○○○○○sleep○○sleep
○○○○○○○sleep sleep
```

如果使用标签功能，就可以变相地实现sed命令组的循环和条件判断功能。例如使用"b"标签跳转功能来实现，语句如下：

```bash
[root@xuexi ~]# sed '
:replace;
s/  /○○/;
/  /b replace;
s/○ /○○/g' test.txt
```

该语句首先定义了一个标签`replace`，在其后是第一个要执行的命令，作用是将两个空格替换成两个圆圈的"s"命令，随后是"b"标签跳转语句，表示如果行中有两个连续的空格，就使用"b"命令跳转到`replace`标签处，于是再次执行"s"命令，替换结束后再次回到`/  /b replace`命令，于是这样就可以对每一输入行实现循环替换动作，直到最后行中没有连续空格。但此时可能还会有一个多余的空格跟在圆圈后面，于是就能满足`s/○ /○○/g`命令的替换条件。

同样，还可以使用"t"标签完成上述要求。

```bash
[root@xuexi ~]# sed '
:replace;
s/  /○○/;
t replace;
s/○ /○○/g' test.txt
 sleep○○○○○○○sleep
○○sleep○○○○○○sleep
○○○sleep○○○○○sleep
○○○○sleep○○○○sleep
○○○○○sleep○○○sleep
○○○○○○sleep○○sleep
○○○○○○○sleep sleep
```

其实"t"标签更可能用于实现像shell中case的条件判断功能。例如：

```bash
sed '{
s/abc/ABC/;
t;
s/opq/OPQ/;
t;
s/xyz/XYZ/}' filename
```

这表示如果能替换什么成功，就跳转到尾部结束，是多选一的情况。

)

<a name="blog3.8"></a>
### 3.8 Commands Specific to GNU 'sed'(GNU sed特有的命令)

这些命令是GNU sed特有的命令，因此必须谨慎使用它们，并且sed脚本没有可移植性的要求。

`'e [COMMAND]'`  
该命令允许将shell命令行的输出结果插入到pattern space中。不给定参数COMMAND时，"e"命令将搜索pattern space并执行发现的命令，并将命令的执行结果替换到pattern space中(注：类似于shell下的命令替换功能)。

如果指定了"COMMAND"参数，"e"命令将解析它并认为它是一个shell命令，并在(子)shell环境下执行后将结果发送到输出流中。

无论是否指定了COMMAND参数，如果待执行命令中包含了空字符，则都不被认为是一个命令。

注意，不像"r"命令，"e"命令的结果是立即输出的；而"r"命令需要延迟到当前SCRIPT循环结束时才会输出。

`'F'`  
输出当前处理的输入流的文件名(输入完后会换行)。

`'L N'`  
这是一个失败的GNU扩展功能，无视。

`'Q [EXIT-CODE]'`  
该命令只能在单个定址表达式下使用。

该命令和命令"q"相同，除了它不会输出pattern space中的内容(注："q"命令在退出前会输出当前pattern space中的内容)。同样，像"q"那样可以提供一个退出状态码。

该命令非常有用，因为要完成这个显而易见的简单功能的唯一替代方法是使用'-n'选项(这可能会使脚本不必要地复杂化)或使用以下代码片段，但这会浪费时间读取整个文件且没有任何效果可见：  
(注：即使使用了"-n"选项，性能也不如"q"和"Q"，因为它仍会读完整个输入流，而"q | Q"是立即退出sed程序。)

```bash
:eat
$d       Quit silently on the last line
N        Read another line, silently
g        Overwrite pattern space each time to save memory
b eat
```

`'R FILENAME'`  
每次读取FILENAME中的一行并队列化在内存中，其余的像"r"选项一样，在每个SCRIPT循环结束时追上stdout流并追加在其尾部。如果FILENAME无可读取或到达了文件尾部，将不会给出任何报错信息。

和"r"命令一样，它也支持特殊的FILENAME值：/dev/stdin，表示从标准输入中读取数据。


(注："R"命令是每次队列化filename中的一行，而"r"命令是一次队列化filename中的所有行。它们都受输出流的影响，有一次输出就追加到输出流中，并开始下一轮的队列化过程。并非sed每从输入流中读取一行就队列化一次)

`'T LABEL'`  
和"t LABEL"相反，该命令是仅当"s"命令未完成替换时跳转到标签LABEL。如果LABEL参数省略，则跳转到SCRIPT的尾部准备进入下一个sed循环，即跳转到隐含动作`auto_print`的前面。

`'v VERSION'`  
该命令除了让sed程序在不支持GNU sed功能的时候失败不做任何事。此外，可以指定一个版本号，表示你的sed脚本必须要高于此版本的sed程序才能运行，例如指定"4.0.5"。默认指定的版本号为"4.0"，因为这是第一个支持该命令的版本。

`'W FILENAME'`  
将pattern space中第一个换行符之前的内容写入到filename中。除此之外，和"w"命令完全相同。

`'z'`  
该命令用于清空pattern space。可以认为它等价于`"s/.*//"`，但却效率更高，且可以在输入流中存在无效多字节序列(例如UTF-8时，一个字符占用多个字节)的情况下工作。POSIX要求这样的序列无法使用"."进行匹配，因此没有任何方法既可以清空sed的buffer空间，又能保证sed脚本的可移植性。

<a name="blog3.9"></a>
### 3.9 GNU Extensions for Escapes in Regular Expressions(GNU对正则表达式的反斜线扩展)

直到目前位置，我们只遇到了`"\^"`格式的转义，它告诉sed不要将脱字符`^`解释为特殊元字符，而是解释为一个普通的字面符号。再例如，`"\*"`匹配的是单个`*`字符，而不是匹配0或多个反斜线字符。

本小节介绍另一种类型的转义。如下：

`'\a'`  
匹配一个BEL字符，即一个"蜂鸣警告"(ASCII 7)。

`'\f'`  
匹配一个分页符(ASCII 12)。

`'\n'`  
匹配一个换行符(ASCII 10)。

`'\r'`  
匹配一个回车符(ASCII 13)。

`'\t'`  
匹配一个水平制表符(ASCII 9)。

`'\v'`  
匹配一个垂直制表符(ASCII 11)。

`'\cX'`  
Produces or matches 'CONTROL-X', where X is any character. The precise effect of `\cX` is as follows: if X is a lower case letter, it is converted to upper case.  Then bit 6 of the character (hex 40) is inverted.  Thus `\cz` becomes hex 1A, but `\c{` becomes hex 3B, while `\c;` becomes hex 7B.

`'\dXXX'`  
匹配十进制ASCII值为XXX的字符。

`'\oXXX'`  
匹配八进制ASCII值为XXX的字符。

`'\xXX'`  
匹配十六进制ASCII值为XX的字符。

`'\b'` (回退删除键)无法实现，因为`"\b"`用于匹配单词的边界。

以下转义序列匹配特殊的字符类，且只在正则表达式中有效：

`'\w'`  
匹配任意单词组成字符。单词的组成成分包括：字母、数字和下划线。其他任意字符都不是单词的一部分。

`'\W'`  
匹配人非单词字符。即匹配除了字母、数字和下划线的任意字符。

`'\b'`  
匹配单词的边界位置。

`'\B'`  
匹配非单词的边界位置。

``"\`"``  
此为sed独有正则表达式。匹配pattern space中的开头。在多行模式下，这和`"^"`是不同的。

`"\'"`  
此为sed独有正则表达式。匹配pattern space中的尾部。在多行模式时，这和`"$"`是不同的。


(注：关于最后两个正则表达式，它们是sed才可以使用的。在多行模式下，它们和`^`、`$`的区别，见[M修饰符](#M_modifier)。)


<a name="blog4"></a>
## 4. Some Sample Scripts(一些简单的示例脚本)

这里有一些sed示例脚本，可以引导你成为大师级的sed使用者。

(注：说实话，下面的17个示例看的我怀疑人生，真的没想到sed功能强大如斯，但也很难)

- **一些吸引人的示例:**
   - [Centering lines](#blog4.1)  
   - [Increment a number](#blog4.2)  
   - [Rename files to lower case](#blog4.3)  
   - [Print bash environment](#blog4.4)  
   - [Reverse chars of lines](#blog4.5)  
- **模仿一些标准的工具:**
   - [tac    :  Reverse lines of files](#blog4.6)  
   - [cat -n :  Numbering lines](#blog4.7)  
   - [cat -b :  Numbering non-blank lines](#blog4.8)  
   - [wc -c  :  Counting chars](#blog4.9)  
   - [wc -w  :  Counting words](#blog4.10)  
   - [wc -l  :  Counting lines](#blog4.11)  
   - [head   :  Printing the first lines](#blog4.12)  
   - [tail   :  Printing the last lines](#blog4.13)  
   - [uniq   :  Make duplicate lines unique](#blog4.14)  
   - [uniq -d:  Print duplicated lines of input](#blog4.15)  
   - [uniq -u:  Remove all duplicated lines](#blog4.16)  
   - [cat -s :  Squeezing blank lines](#blog4.17)  

<a name="blog4.1"></a>
### 4.1 Centering Lines(文本中心线)

该脚本将所有行都划分中心线，每行最多80个字符，中心线的位置指定为第40个字符处。如果要改变行宽度，则下面的`'\{...\}'`中的数字必须更改，同时需要在脚本的第一段中添加或删除适当个数的空格。

注意此处是如何使用buffer空间的命令来分隔两部分的，这是一个通用技巧。

(注：如果行的内容原本就超过81个字符，则这些行的行尾会被截断)

```bash
 #!/usr/bin/sed -f

 # Put 80 spaces in the buffer
 1 {
   x
   s/^$/          /
   s/^.*$/&&&&&&&&/ 
   x
 }

 # del leading and trailing spaces
 y/tab/ / 
 s/^ *//  
 s/ *$//   

 # add a newline and 80 spaces to end of line
 G

 # keep first 81 chars (80 + a newline)
 s/^\(.\{81\}\).*$/\1/

 # \2 matches half of the spaces, which are moved to the beginning
 s/^\(.*\)\n\(.*\)\2/\2\1/
```

(注：从此脚本可知，虽在pattern space中加上了换行符，但其实不是真的换行，它在pattern space中仍是一行，只不过在输出时换行符使结果换了行。因此，N命令进入多行模式也是一样的。同时，换行符是一个字符，能被匹配，也能被替换掉)

<a name="blog4.2"></a>
### 4.2 Increment a Number(数值增长)

该脚本是sed中少数几个演示了如何做算术计算的示例。这确实很有可能，但只能手动实现。

为了增长一个数值，只需要为该数的末尾加上1，并替换原末尾即可。但有一种例外：当末尾数位9时，其前面一位也必须增长，如果这还是9，则还需增长，直到无需进位时才结束。

Bruno Haible提供的解决方案非常聪明，因为它只使用了一个buffer空间。它是这样工作的：将末尾9替换成下划线，然后使用多个"s"命令增长最后一个数字(下划线前面的数字也属于最后一个数字)，最后将所有的下划线替换为0。  

(注：下面的`td`和`tn`是`t d`和`t n`的简写。)

```bash
 #!/usr/bin/sed -f

 /[^0-9]/ d

 # replace all trailing 9s by _ (any other character except digits, could
 # be used)
 :d
 s/9\(_*\)$/_\1/
 td

 # incr last digit only.  The first line adds a most-significant
 # digit of 1 if we have to add a digit.

 s/^\(_*\)$/1\1/; tn
 s/8\(_*\)$/9\1/; tn
 s/7\(_*\)$/8\1/; tn
 s/6\(_*\)$/7\1/; tn
 s/5\(_*\)$/6\1/; tn
 s/4\(_*\)$/5\1/; tn
 s/3\(_*\)$/4\1/; tn
 s/2\(_*\)$/3\1/; tn
 s/1\(_*\)$/2\1/; tn
 s/0\(_*\)$/1\1/; tn

 :n
 y/_/0/
```

(注：例如为该sed脚本传递一个个位数8，由于该数据是以8结尾的，因此将一直执行到`s/8\(_*\)$/9\1/;tn`将8增长为9，并跳转到标签n，标签n后的命令不用对其处理，所以直接输出9。假如传递的数字是3999，则`:d;s/9\(_*\)$/_\1/;td`将一直循环，首先替换成`"399_"`，再循环一次替换成`"39__"`，最后一次循环替换成`"3___"`，由于此时结尾没9了，且是以3结尾，于是继续向下直到`s/3\(_*\)$/4\1/; tn`，它会将`"3___"`替换成`"4___"`，并跳转到标签n，`y/_/0/`将把下划线替换成0得到4000，最后输出结果为4000。

<a name="blog4.3"></a>

### 4.3 Rename Files to Lower Case(重命名文件名为小写)

这是一个非常奇怪的sed应用。转换文件名文本，并转换其为shell命令，然后将它们提供给shell。

该应用的主体是其中的sed脚本部分，它会重新将名称的小写字母映射为大写，甚至还检查重新映射后的名称是否和原始名称相同。注意脚本是如何使用shell变量和正确的引号进行参数化的。

```bash
 #! /bin/sh
 # rename files to lower/upper case...
 #
 # usage:
 #    move-to-lower *
 #    move-to-upper *
 # or
 #    move-to-lower -R .
 #    move-to-upper -R .
 #

 help()
 {
         cat << eof
 Usage: $0 [-n] [-R] [-h] files...

 -n      do nothing, only see what would be done
 -R      recursive (use find)
 -h      this message
 files   files to remap to lower case

 Examples:
        $0 -n *        (see if everything is ok, then...)
        $0 *

        $0 -R .

 eof
 }

 apply_cmd='sh'
 finder='echo "$@" | tr " " "\n"'
 files_only=

 while :
 do
     case "$1" in
         -n) apply_cmd='cat' ;;
         -R) finder='find "$@" -type f';;
         -h) help ; exit 1 ;;
         *) break ;;
     esac
     shift
 done

 if [ -z "$1" ]; then
         echo Usage: $0 [-h] [-n] [-r] files...
         exit 1
 fi

 LOWER='abcdefghijklmnopqrstuvwxyz'
 UPPER='ABCDEFGHIJKLMNOPQRSTUVWXYZ'

 case 'basename $0' in
         *upper*) TO=$UPPER; FROM=$LOWER ;;
         *)       FROM=$UPPER; TO=$LOWER ;;
 esac

 eval $finder | sed -n '

 # remove all trailing slashes
 s/\/*$//

 # add ./ if there is no path, only a filename
 /\//! s/^/.\//

 # save path+filename
 h

 # remove path
 s/.*\///

 # do conversion only on filename
 y/'$FROM'/'$TO'/

 # now line contains original path+file, while
 # hold space contains the new filename
 x

 # add converted file name to line, which now contains
 # path/file-name\nconverted-file-name
 G

 # check if converted file name is equal to original file name,
 # if it is, do not print anything
 /^.*\/\(.*\)\n\1/b

 # escape special characters for the shell
 s/["$'\\]/\\&/g

 # now, transform path/fromfile\n, into
 # mv path/fromfile path/tofile and print it
 s/^\(.*\/\)\(.*\)\n\(.*\)$/mv "\1\2" "\1\3"/p

 ' | $apply_cmd
```


<a name="blog4.4"></a>
### 4.4 Print 'bash' Environment(打印"bash"环境)

该脚本去除了'set'命令中的shell function的定义。

```bash
 #!/bin/sh

 set | sed -n '
 :x

 # if no occurrence of "=()" print and load next line
 /=()/! { p; b; }
 / () $/! { p; b; }

 # possible start of functions section
 # save the line in case this is a var like FOO="() "
 h

 # if the next line has a brace, we quit because
 # nothing comes after functions
 n
 /^{/ q

 # print the old line
 x; p

 # work on the new line now
 x; bx
 '
```

<a name="blog4.5"></a>
### 4.5 Reverse Characters of Lines(反转行中的字符)

该脚本可用反转行中字符的位置，它使用了一次性移动两个字符的技术，因此它的速度比直观看上去的更快。

注意标签定义语句前面的"tx"命令，它用来重置"t"标签跳转的条件，免得被第一个"s"命令影响而跳转。

```bash
 #!/usr/bin/sed -f

 /../! b

 # Reverse a line.  Begin embedding the line between two newlines
 # (注：下面的"s"命令使用了转义行尾的技巧，其实和"sed /^.*$/\n&\n/"是等价的)
 s/^.*$/\
 &\
 /

 # Move first character at the end.  The regexp matches until
 # there are zero or one characters between the markers
 tx
 :x
 s/\(\n.\)\(.*\)\(.\n\)/\3\2\1/
 tx

 # Remove the newline markers
 s/\n//g
```

(注：例如`echo abcde | sed -f file.sed`，传入的是"abcde"，首先判断它不是单字符，否则将跳转到尾部，然后将"abcde"嵌入到两个相同的特殊字符中，此处采用的是嵌入到两空行中，为了解释方便，假设这里的特殊字符为`#`符号，于是得到`#abcde#`，随后一个`t x`命令用于重置跳转条件，避免第一个s命令的成功状态影响跳转条件。再执行第二个s命令，该命令将两个`#`前后的字符反转，第一轮循环得到`e#dcb#a`，随后跳转再次替换，得到`ed#c#ba`，再次跳转，但这一次替换不成功。最后将所有的`#`删除，即得到最终结果"edcba")

<a name="blog4.6"></a>

### 4.6 Reverse Lines of Files(反转文件中的行)

这是一个没什么用但却很有意思的脚本，是模仿unix命令。就像`tac`一样工作。

注意，非GNU sed执行该脚本时很容易内存溢出。

```bash
 #!/usr/bin/sed -nf

 # reverse all lines of input, i.e. first line became last, ...

 # from the second line, the buffer (which contains all previous lines)
 # is *appended* to current line, so, the order will be reversed
 1! G

 # on the last line we're done -- print everything
 $ p

 # store everything on the buffer again
 h
```

<a name="blog4.7"></a>
### 4.7 Numbering Lines(打印行号)

该脚本和`cat -n`的功能一样。当然，这个脚本没什么用，两个原因：第一，有人使用C语言实现了该功能，第二，下面这个shell脚本可以实现相同的目的却更快。

```bash
 #! /bin/sh
 sed -e "=" $@ | sed -e '
   s/^/      /
   N
   s/^ *\(......\)\n/\1  /
 '
```

它首先使用sed来输出行号，然后通过"N"两行一分组并进行调整。当然，这个脚本并没有下面这个脚本的引导意义多。

这个算法同时使用了两个buffer空间，使得每一行可以尽快被打印出来然后丢弃。而行号被分离，使得数字改变后可以放入一个buffer空间，而没有改变的数字在另一个buffer空间；改变的数字使用一个"y"命令来修改。于是行号被存储在hold space，在下一个迭代中被使用上。

```bash
 #!/usr/bin/sed -nf

 # Prime the pump on the first line
 x
 /^$/ s/^.*$/1/

 # Add the correct line number before the pattern
 G
 h

 # Format it and print it
 s/^/      /
 s/^ *\(......\)\n/\1  /p

 # Get the line number from hold space; add a zero
 # if we're going to add a digit on the next line
 g
 s/\n.*$//
 /^9*$/ s/^/0/

 # separate changing/unchanged digits with an x
 s/.9*$/x&/

 # keep changing digits in hold space
 h
 s/^.*x//
 y/0123456789/1234567890/
 x

 # keep unchanged digits in pattern space
 s/x.*$//

 # compose the new number, remove the newline implicitly added by G
 G
 s/\n//
 h
```

<a name="blog4.8"></a>
### 4.8 Numbering Non-blank Lines(打印非空白行行号)

模仿的是`cat -b`，它不会计算空行的行号。

在此脚本中和上一个脚本相同的部分没有给出注释，于此处可知，对于sed脚本来说，给注释是多么重要的事。

```bash
 #!/usr/bin/sed -nf

 /^$/ {
   p
   b
 }

 # Same as cat -n from now
 x
 /^$/ s/^.*$/1/
 G
 h
 s/^/      /
 s/^ *\(......\)\n/\1  /p
 x
 s/\n.*$//
 /^9*$/ s/^/0/
 s/.9*$/x&/
 h
 s/^.*x//
 y/0123456789/1234567890/
 x
 s/x.*$//
 G
 s/\n//
 h
```

<a name="blog4.9"></a>
### 4.9 Counting Characters(计算字符个数)

该脚本展示了sed另一种做数学计算的方式。此处我们必须可以增大到一个极大的数值，因此要成功实现这一点可能不那么容易。

解决的方法是将数字映射为字母，这是sed的一种算盘功能实现。"a"表示个位数，"b"表示十位数，以此类推。如前所见，我们仍然在hold space中保存总数。

在最后一行上，我们将算盘上的值转换为十进制数字。为了能适应多种情况，这里使用了循环而不是一大堆的"s"命令：首先转换个位数，然后从数值中移除字母"a"，然后滚动字母使得各个字母都转换成对应的数值。

```bash
 #!/usr/bin/sed -nf

 # Add n+1 a's to hold space (+1 is for the newline)
 s/./a/g
 H
 x
 s/\n/a/

 # Do the carry.  The t's and b's are not necessary,
 # but they do speed up the thing
 t a
 : a;  s/aaaaaaaaaa/b/g; t b; b done
 : b;  s/bbbbbbbbbb/c/g; t c; b done
 : c;  s/cccccccccc/d/g; t d; b done
 : d;  s/dddddddddd/e/g; t e; b done
 : e;  s/eeeeeeeeee/f/g; t f; b done
 : f;  s/ffffffffff/g/g; t g; b done
 : g;  s/gggggggggg/h/g; t h; b done
 : h;  s/hhhhhhhhhh//g

 : done
 $! {
   h
   b
 }

 # On the last line, convert back to decimal

 : loop
 /a/! s/[b-h]*/&0/
 s/aaaaaaaaa/9/
 s/aaaaaaaa/8/
 s/aaaaaaa/7/
 s/aaaaaa/6/
 s/aaaaa/5/
 s/aaaa/4/
 s/aaa/3/
 s/aa/2/
 s/a/1/

 : next
 y/bcdefgh/abcdefg/
 /[a-h]/ b loop
 p
```

<a name="blog4.10"></a>

### 4.10 Counting Words(计算单词个数)

该脚本和前一个差不多，每个单词都转换成一个单独的字母"a"(上一个脚本中是每一个字母转换成一个字母"a")。

有趣的是，"wc"程序对`wc -c`计算字符时做了优化，使得计算单词的速度要比计算字符的速度慢很多。与之相反，这个脚本的瓶颈在于算术，因此单词计算的速度要快的多(因为只需维护少量的数值计算)

此脚本中和上一个脚本的共同部分没有给注释，这再次说明了sed脚本中注释的重要性。

```bash
 #!/usr/bin/sed -nf

 # Convert words to a's
 s/[ tab][ tab]*/ /g
 s/^/ /
 s/ [^ ][^ ]*/a /g
 s/ //g

 # Append them to hold space
 H
 x
 s/\n//

 # From here on it is the same as in wc -c.
 /aaaaaaaaaa/! bx;   s/aaaaaaaaaa/b/g
 /bbbbbbbbbb/! bx;   s/bbbbbbbbbb/c/g
 /cccccccccc/! bx;   s/cccccccccc/d/g
 /dddddddddd/! bx;   s/dddddddddd/e/g
 /eeeeeeeeee/! bx;   s/eeeeeeeeee/f/g
 /ffffffffff/! bx;   s/ffffffffff/g/g
 /gggggggggg/! bx;   s/gggggggggg/h/g
 s/hhhhhhhhhh//g
 :x
 $! { h; b; }
 :y
 /a/! s/[b-h]*/&0/
 s/aaaaaaaaa/9/
 s/aaaaaaaa/8/
 s/aaaaaaa/7/
 s/aaaaaa/6/
 s/aaaaa/5/
 s/aaaa/4/
 s/aaa/3/
 s/aa/2/
 s/a/1/
 y/bcdefgh/abcdefg/
 /[a-h]/ by
 p
```

<a name="blog4.11"></a>
### 4.11 Counting Lines(统计行数)

就像`wc -l`一样，统计行数。

```bash
 #!/usr/bin/sed -nf
 $=
```

<a name="blog4.12"></a>
### 4.12 Printing the First Lines(打印顺数前几行)

该脚本是sed脚本中最有用又最简单的。它显示了输入流的前10行。使用"q"的作用是立即退出sed，不再读取额外的行浪费时间和资源。

```bash
 #!/usr/bin/sed -f
 10q
```


<a name="blog4.13"></a>
### 4.13 Printing the Last Lines(打印倒数几行)

输出倒数N行比输出顺数前N行要复杂的多，但确实很有用。N值是下面脚本中`1,10 !s/[^\n]*\n//`的数值10对应值，此处表示输出倒数10行。

该脚本类似于`tac`脚本，都将每次的结果保留在hold space中最后输出。

```bash
 #!/usr/bin/sed -nf

 1! {; H; g; }
 1,10 !s/[^\n]*\n//
 $p
 h
```

关键点，该脚本维持了一个10行的窗口，在向其中添加一行时滑动该窗口并删除最旧的一行(第二个s命令有点类似于"D"命令，但却不用重新进入SCRIPT循环)

在写高级或复杂sed脚本时，**"窗口滑动"**的技术作用非常大，因为像"P"这样的命令要实现相同的目的需要手动做很多额外的工作。

为了介绍这种技术，在剩下的几个示例中，均使用了"N"、"P"和"D"命令充分演示了该技术。此处是一个"窗口滑动"技术对"tail"命令的实现。

这个脚本看上去更复杂，但实际上工作方式和上一个脚本是一样的：当踢掉了合理的行后，不再使用hold space来保存内部行的状态，而是使用"N"和"D"命令来滑动pattern space：

```bash
 #!/usr/bin/sed -f

 1h
 2,10 {; H; g; }
 $q
 1,9d
 N
 D
```

注意在读取了输入流的前10行后，其中的第1、2、4行的命令就失效了。之后，所有的工作是：最后一行时推出，追加下一行到pattern space中并移除其内第一行。

<a name="blog4.14"></a>
### 4.14 Make Duplicate Lines Unique(移除重复行)

这个示例充分展现了"N"、"D"和"P"命令的艺术所在，这可能是成为大师路上最难的几个命令。

```bash
 #!/usr/bin/sed -f
 h

 :b
 # On the last line, print and exit
 $b
 N
 /^\(.*\)\n\1$/ {
     # The two lines are identical. Undo the effect of the N command.
     g
     bb
 }

 # If the 'N' command had added the last line, print and exit
 $b

 # The lines are different; print the first and go
 # back working on the second.
 P
 D
```

正如所见，我们使用"P"和"D"维护了一个2行的窗口空间。这种窗口(滑动)技术在高级sed脚本中经常会使用。

(注：**"N"、"P"和"D"是sed中绝佳组合，通常为这3个命令关联不同的定址表达式以及感叹号"!"时，得到的结果千变万化。一方面"N"和"D"可以"滑动窗口"，另一方面，"P"和"D"可以输出窗口的第一行并滑动，再配合"N"或其它进入多行模式的方法(如"G"，或"s"命令添加了换行符`\n`)，使得窗口的大小可以一直维持下去。通常这3个命令同时使用时，它们的相对前后顺序是"NPD"。**另外，"D"比较特殊，它会重新进入SCRIPT循环，使得窗口中的内容只保留符合条件的行，因此滑动窗口变得更加"动态"，要使用s命令实现这种动态窗口滑动，只能借助"t"标签跳转来实现循环)

<a name="blog4.15"></a>
### 4.15 Print Duplicated Lines of Input(只打印重复行)

该脚本只打印重复行，就像"uniq -d"一样。

```bash
 #!/usr/bin/sed -nf

 $b
 N
 /^\(.*\)\n\1$/ {
     # Print the first of the duplicated lines
     s/.*\n//
     p

     # Loop until we get a different line
     :b
     $b
     N
     /^\(.*\)\n\1$/ {
         s/.*\n//
         bb
     }
 }

 # The last line cannot be followed by duplicates
 $b

 # Found a different one.  Leave it alone in the pattern space
 # and go back to the top, hunting its duplicates
 D
```

<a name="blog4.16"></a>
### 4.16 Remove All Duplicated Lines(移除所有重复行)

该脚本移除所有重复行，就像"uniq -u"一样。

```bash
 #!/usr/bin/sed -f

 # Search for a duplicate line --- until that, print what you find.
 $b
 N
 /^\(.*\)\n\1$/ ! {
     P
     D
 }

 :c
 # Got two equal lines in pattern space.  At the
 # end of the file we simply exit
 $d

 # Else, we keep reading lines with 'N' until we
 # find a different one
 s/.*\n//
 N
 /^\(.*\)\n\1$/ {
     bc
 }

 # Remove the last instance of the duplicate line
 # and go back to the top
 D
```

<a name="blog4.17"></a>

### 4.17 Squeezing Blank Lines(压缩连续的空白行)

作为最后一个示例，这里给出了3个脚本，用于增加复杂度和速度，这可以使用"cat -s"来实现同样的功能：压缩空白行。

第一个脚本的实现方式是保留第一个空行，如果发现后续还有连续空行，则直接跳过。

```bash
 #!/usr/bin/sed -f

 # on empty lines, join with next
 # Note there is a star in the regexp
 :x
 /^\n*$/ {
 N
 bx
 }

 # now, squeeze all '\n', this can be also done by:
 # s/^\(\n\)*/\1/
 s/\n*/\
 /
```

下面这个脚本要更复杂一些，它会移除所有的第一个空行，并保留最后一个空行。

     #!/usr/bin/sed -f
    
     # delete all leading empty lines
     1,/^./{
     /./!d
     }
    
     # on an empty line we remove it and all the following
     # empty lines, but one
     :x
     /./!{
     N
     s/^\n$//
     tx
     }

下面这个脚本会移除前导和尾随空行，速度最快。注意，"n"和"b"会彻底完成整个循环，而不会依赖于sed自动读取下一行。

```bash
 #!/usr/bin/sed -nf

 # delete all (leading) blanks
 /./!d

 # get here: so there is a non empty
 :x
 # print it
 p
 # get next
 n
 # got chars? print it again, etc...
 /./bx

 # no, don't have chars: got an empty line
 :z
 # get next, if last line we finish here so no trailing
 # empty lines are written
 n
 # also empty? then ignore it, and get next... this will
 # remove ALL empty lines
 /./!bz

 # all empty lines were deleted/ignored, but we have a non empty.  As
 # what we want to do is to squeeze, insert a blank line artificially
 i\

 bx
```

<a name="blog5"></a>

## 5. GNU sed's Limitations and Non-limitations(GNU sed的限制和优点)

对于那些想写具有可移植性的sed脚本，需要注意有些sed程序有众所周知的buffer大小限制，要求不能超过4000字节，在POSIX标准中明确说明了要不小于8192字节。在GNU sed则没有这些限制。

然而，在匹配的时候是使用递归处理子模式和无限重复。这意味着可用的栈空间可能会限制那些可以通过某些pattern处理的缓冲区的大小。


<a name="blog6"></a>
## 6. Other Resources for Learning About 'sed'(学习sed的其他资源)

除了某些关于sed的书籍(专门研究或作为某个章节讨论的shell编程)之外，可以从"sed-users"邮件列表的FAQ中(包含一些sed书籍的推荐和建议)获取更多关于sed的信息：<http://sed.sourceforge.net/sedfaq.html>

此外，还有sed的新手教程和其他一些sed相关资源：

<http://www.student.northpark.edu/pemente/sed/index.htm>

<http://sed.sf.net/grabbag>

"sed-user"邮件列表由Sven Guckes负责维护，可以访问<http://groups.yahoo.com>来订阅。

<a name="blog7"></a>
#7. Reporting Bugs(Bugs说明)

如要报告bug，邮送至`[bug-sed@gnu.org](bug-sed@gnu.org)`，请同时包含在邮件的body中包含sed的版本号，即`sed --version`的输出结果。

请不要像下面一样报告bug：
```
while building frobme-1.3.4
$ configure
error--> sed: file sedscr line 1: Unknown option to 's'
```
以下是一些容易被报告的bug，但实际上却不是bug：(注：也就是容易让人疑惑的点，对于深入理解sed帮助很大)

- 'N'命令在最后一行上的处理方式

大多数版本的sed在"N"命令处理输入流的最后一行时会直接退出而不打印任何东西。GNU sed会输出pattern space的内容，除非使用了"-n"选项。这是GNU sed特意的。

例如，`sed N foo bar`将依赖于"foo"是偶数行还是奇数行。或者，当想在某个匹配行后读取后续的几行时，传统的sed可能只能写成这样：

```bash
/foo/{ $!N; $!N; $!N; $!N; $!N; $!N; $!N; $!N; $!N }
```

来替代：

```bash
/foo/{ N;N;N;N;N;N;N;N;N; }
```

无论何时，最简单的工作方式是使用`$d;N`，或者设置"POSIXLY_CORRECT"变量为一个非空值，来解决上述传统的依赖行为。

- 正则表达式崩溃(问题出在反斜线上)

sed使用的是POSIX的基础正则表达式语法，根据POSIX标准，有些转义序列没有预定义，在sed中需要注意的包括：

```bash
\|、\+、\?、\<、\>、\b、\B、\w、\W、\'和\`
```

由于所有的GNU程序都是用POSIX的基础正则表达式，sed会将这些转义符解析为特殊的元字符，因此`x\+`会匹配一个或多个"x"，`abc\|def`会匹配"abc"或"def"。

这些语法在用其他版本的sed运行时可能会出现问题，特别是某些版本的sed程序会将`|`和`+`解析为字面符号的`|`和`+`。因此在某些版本的sed上运行需要参考其所支持的正则表达式语法做相应修改。

再者说，某些脚本中`s|abc\|def||g`来移除"abc"或"def"字符串，但这只在sed 4.0.x版之前能正常工作，在新版的sed中会将其解析为移除"abc|def"字符串。这在POSIX中同样没有定义对应的标准。

- '-i'选项会破坏只读文件

简单地说，"sed -i"将删除只读文件中的内容。通俗地说，"-i"选项会破坏受保护的只读文件。这不是bug，而是Unix文件系统工作方式引起的结果。

普通文件的权限表示的是可以对该文件中的数据做什么样的操作，但是目录的权限表示的是可以对目录中的文件做什么样的操作。"sed -i"绝不会为了写入而再次打开该文件。取而代之的是，它会先写入一个临时文件，最后将其重命名为原文件名：删除或重命名文件的权限是目录权限控制的，因此该操作依赖的是目录的权限，而非文件本身的权限。同样的原因，"sed -i"不允许修改一个可读但其目录只读的普通文件。


- '0a'无法工作，报错

根本就没有0行。0行的概念只有一种情况下使用`0,/REGEXP/`，它和`1,/REGEXP/`只有一点不同：如果REGEXP能匹配上第一行，则前者的结果是只有第一行，而后者会继续向下匹配直到能匹配REGEXP，因为范围地址必须要跨越至少两行，除非直到最后一行都没有匹配上。

- '\[a-z\]'会忽略大小写

这是字符集环境设置的问题。POSIX强制`[a-z]`这样的字符列表采用当前系统当前字符集的排序规则进行排序，C字符集环境下，它代表的是小写字母序列，其他字符集环境下，可能代表的是小写和大写字母序列，这依赖于字符集。

为了解决这个问题，可以设置`LC_COLLATE`和`LC_CTYPE`环境变量为C。

- `s/.*//`不会清空pattern space

会发生这种情况，可能是因为你的输入流中包含了多字节序列(如UTF-8).POSIX强制这样的序列字符无法被"."匹配，因此`s/.*//`将不会如你所愿那样清空pattern space。事实上，在绝大多数多字节序列环境下，没有任何办法脚本的中途清空pattern space。出于这个原因，GNU sed提供了一个"z"命令，它将`\0`作为输入流的行分隔符。

为了解决这些问题，可以设置`LC_COLLATE`和`LC_CTYPE`环境变量为C。
