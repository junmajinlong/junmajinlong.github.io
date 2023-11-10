---
title: grep中文手册(info grep翻译)
p: shell/grep_translate.md
date: 2019-12-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# grep命令中文手册(info grep翻译)

```
1 Introduction
**************

'grep'用于搜索给定文件中能匹配给定pattern列表的行。当某行能匹配上，(默认)将拷贝该行到标准输出，或者根据你所指定的选项生成其它序列的输出。

尽管'grep'所期望的是在文本行中做匹配，但即使某输入行的大小长度超出了可用内存空间也不会受到限制，它仍可以匹配一行中任意字符串。如果输入文件的最后一个字节不是换行符，'grep'会自动补上一个。由于换行符也是pattern列表的分隔符，因此没有任何办法匹配文本中的换行符。

2 Invoking 'grep'
*****************

'grep'命令行的一般语法格式为：

     grep OPTIONS PATTERN INPUT_FILE_NAMES

OPTIONS部分可以指定0或多个。只有当没有使用"-e PATTERN"或"-f FILE"时，指定的PATTERN才被grep可视。可以指定0或多个INPUT_FILE_NAMES。

2.1 Command-line Options(命令行选项)
========================

'grep'有大量选项可用：一些是POSIX.2中的，一些是GNU扩展的。长选项都是GNU扩展选项，即使它们来自于POSIX。由POSIX指定的短选项，被明确标注为便于POSIX可移植性编程。有少数几个选项是为了兼容古老版本的grep。
有几个额外的选项用于控制使用哪种变体'grep'匹配引擎(注：fgrep/pgrep/egrep)。

2.1.1 Generic Program Information
---------------------------------

'--help'
     输出简短的grep命令行使用帮助并退出。

'-V'
'--version'
     输出'grep'的版本号。

2.1.2 Matching Control(控制匹配模式)
----------------------

'-e PATTERN'
'--regexp=PATTERN'
    明确指定使用此处的PATTERN作为待匹配的pattern。该选项可以指定多次，它可以保护以"-"开头的pattern。('-e'是POSIX指定的选项。)

'-f FILE'
'--file=FILE'
    从FILE中获取pattern列表，每行一个pattern。空的FILE表示不给定任何pattern，所以不会匹配到任何内容。('-f'是POSIX指定的选项。)

'-i'
'-y'
’--ignore-case'
    忽略PATTERN中的大小写，也忽略输入文件中的大小写区别。'-y'是废弃的用于和老版本保持兼容性的选项。('-i'是POSIX指定的选项。)

'-v'
'--invert-match'
    反转匹配的结果，即选择那些未匹配到的行。('-v'是POSIX指定的选项。)

'-w'
'--word-regexp'
    仅选择能精确匹配整个单词的行。单词的组成字符包括：字母、数字和下划线。除了这些字符，其余都是该选项筛选单词时的单词边界分隔符。
    (注：例如字符串"fstab fstab(5)"，grep -w 'fstab'或grep -w 'fsta.'能匹配这两个单词，但grep -w 'fsta'无法匹配任意一个)

'-x'
'--line-regexp'
    仅选择能精确匹配整行内容的行。('-x'是POSIX指定的选项。)
    (注：例如某行"abcde"，grep -x 'abc'将无法匹配该行，而grep -x 'abcd.'能匹配该行)
    
2.1.3 General Output Control(控制输出内容)
----------------------------

'-c'
'--count'
     不再输出匹配的内容，而是输出匹配到的行数量。如果给定了"-v"选项，则输出未匹配到的行数量。('-c'是POSIX指定的选项。)

'--color[=WHEN]'
'--colour[=WHEN]'
      对匹配到的内容赋予颜色并输出。WHEN的有效值包括：'never'、'always'或'auto'。

'-L'
'--files-without-match'
     不再输出匹配的内容，而是输出未能被匹配到的文件名，当某文件中的某行被匹配到，将不再继续向下搜索该文件。
     (注：和"-l"输出的文件名相反)

'-l'
'--files-with-matches'
     不再输出匹配的内容，而是输出能被匹配到的文件名，当某文件中的某行被匹配到，将不再继续向下搜索该文件。('-l'是POSIX指定的选项。)

'-m NUM'
'--max-count=NUM'
     当匹配成功的行有NUM行时，停止读取文件。如果是普通文件作为标准输入，则输出这匹配到的NUM行。grep会在最后一次匹配行后做位置标记，使得调用的另一个进程可以从此处恢复并继续向下搜索。例如，下面的shell脚本：

          while grep -m 1 PATTERN
          do
            echo xxxx
          done < FILE
    
     而下面的shell脚本则以不同于上面脚本方式运行，因为此处使用的是管道，这不是一个实体文件：
    
          cat FILE |
          while grep -m 1 PATTERN
          do
            echo xxxx
          done

'-o'
'--only-matching'
     输出被匹配到的字符串，而不是输出整行。每个被匹配到的字符串都使用单独的行输出。

'-q'
'--quiet'
'--silent'
     静默模式，立即退出，即使遇到了错误。不写任何内容到标准输出。如果匹配到了内容则退出状态码为0。('-q'是POSIX指定的选项。)

'-s'
'--no-messages'
     禁止输出因文件不存在或文件没有读权限而产生的错误信息。('-s'是POSIX指定的选项。)
     
     (注：由于POSIX和GNU grep的差异性，在可移植性的脚本中，应尽量避免使用"-q"和"-s"，而是使用重定向的方式重定向到/dev/null)

2.1.4 Output Line Prefix Control(控制输出行的前缀)
--------------------------------

当输出行有前缀要输出时，它们的顺序总是：文件名、行号、字节的偏移量，这个顺序不会因为前缀控制选项的顺序而改变。

'-b'
'--byte-offset'
     Print the 0-based byte offset within the input file before each line of output.  If '-o' ('--only-matching') is specified, print the offset of the matching part itself.  When 'grep' runs on MS-DOS or MS-Windows, the printed byte offsets depend on whether the '-u' ('--unix-byte-offsets') option is used; see below.

'-H'
'--with-filename'
     输出匹配到内容所在文件的文件名。当指定了多个输入文件时，这是默认的。

'-h'
'--no-filename'
     禁止输出文件名。当只有一个输入文件时，这是默认的。

'--label=LABEL'
     Display input actually coming from standard input as input coming from file LABEL.  This is especially useful when implementing tools like 'zgrep'; e.g.:

          gzip -cd foo.gz | grep --label=foo -H something

'-n'
'--line-number'
     输出匹配内容在文件中的行号，每个文件都单独从1开始计数。('-n'是POSIX指定的选项。)

'-T'
'--initial-tab'
     Make sure that the first character of actual line content lies on a tab stop, so that the alignment of tabs looks normal.  This is useful with options that prefix their output to the actual content:'-H', '-n', and '-b'.  In order to improve the probability that lines from a single file will all start at the same column, this also causes the line number and byte offset (if present) to be printed in a minimum-size field width.

'-u'
'--unix-byte-offsets'
     Report Unix-style byte offsets.  This option causes 'grep' to report byte offsets as if the file were a Unix-style text file, i.e., the byte offsets ignore the 'CR' characters that were stripped.  This will produce results identical to running 'grep' on a Unix machine.  This option has no effect unless the '-b' option is also used; it has no effect on platforms other than MS-DOS and MS-Windows.

'-Z'
'--null'
     在输出文件名时，使用"\0"放在文件名后，这会替换原本使用的字符，如换行符或冒号。例如"grep -lZ"输出的每个文件都在同一行而不是分行，"grep -HZ"使得文件名后没有冒号。

2.1.5 Context Line Control(控制输出行的上下文)
--------------------------

无论下面的选项如何设置，grep都不会多次输出同一行。如果指定了"-o"选项，这些选项将失效，并给出一个警告。

'-A NUM'
'--after-context=NUM'
     除了输出匹配到的行，还输出匹配到内容的后NUM行。

'-B NUM'
'--before-context=NUM'
     除了输出匹配到的行，还输出匹配到内容的前NUM行。

'-C NUM'
'-NUM'
'--context=NUM'
     除了输出匹配到的行，还输出匹配到内容的前NUM行和后NUM行。

'--group-separator=STRING'
     当使用'-A', '-B' or '-C'时，使用STRING替代默认的组分隔符。
     
     (注：组分隔符表示匹配到的内容的上下文。例如"-A 2"，在某行匹配到时，还将输出后两行，这是一个组。下一次匹配成功时，如果是在该组之后行匹配上的，则这两组中间默认使用"--"分隔)

'--no-group-separator'
     当使用'-A', '-B' or '-C'时，不输出任何组分隔符，而是将不同组相邻输出。

2.1.6 File and Directory Selection(文件和目录的选择)
----------------------------------

'-a'
'--text'
     Process a binary file as if it were text; this is equivalent to the '--binary-files=text' option.

'--binary-files=TYPE'
     If the first few bytes of a file indicate that the file contains binary data, assume that the file is of type TYPE.  By default, TYPE is 'binary', and 'grep' normally outputs either a one-line message saying that a binary file matches, or no message if there is no match.  If TYPE is 'without-match', 'grep' assumes that a binary file does not match; this is equivalent to the '-I' option. If TYPE is 'text', 'grep' processes a binary file as if it were text; this is equivalent to the '-a' option.  _Warning:_ '--binary-files=text' might output binary garbage, which can have nasty side effects if the output is a terminal and if the terminal driver interprets some of it as commands.

'-D ACTION'
'--devices=ACTION'
     If an input file is a device, FIFO, or socket, use ACTION to process it.  By default, ACTION is 'read', which means that devices are read just as if they were ordinary files.  If ACTION is 'skip', devices, FIFOs, and sockets are silently skipped.

'-d ACTION'
'--directories=ACTION'
     If an input file is a directory, use ACTION to process it.  By default, ACTION is 'read', which means that directories are read just as if they were ordinary files (some operating systems and file systems disallow this, and will cause 'grep' to print error messages for every directory or silently skip them).  If ACTION is 'skip', directories are silently skipped.  If ACTION is 'recurse', 'grep' reads all files under each directory, recursively; this is equivalent to the '-r' option.

'--exclude=GLOB'
     忽略basename能被GLOB匹配到的文件。GLOB通配符包括："*"、"?"和"[...]"。

'--exclude-from=FILE'
     从FILE中读取exclude的排除规则。

'--exclude-dir=DIR'
     筛选出不进行递归搜索的目录，使用DIR进行匹配。

'-I'
     Process a binary file as if it did not contain matching data; this is equivalent to the '--binary-files=without-match' option.

'--include=GLOB'
     只搜索basename能被GLOB匹配的文件。

'-r'
'-R'
'--recursive'
     从命令行中给定的目录中递归进去，搜索其中的每个文件和目录。

2.1.7 Other Options(其他选项)
-------------------

'--line-buffered'
     Use line buffering on output.  This can cause a performance penalty.

'--mmap'
     This option is ignored for backwards compatibility.  It used to read input with the 'mmap' system call, instead of the default 'read' system call.  On modern systems, '--mmap' rarely if ever yields better performance.

'-U'
'--binary'
     Treat the file(s) as binary.  By default, under MS-DOS and MS-Windows, 'grep' guesses the file type by looking at the contents of the first 32kB read from the file.  If 'grep' decides the file is a text file, it strips the 'CR' characters from the original file contents (to make regular expressions with '^' and '$' work correctly).  Specifying '-U' overrules this guesswork, causing all files to be read and passed to the matching mechanism verbatim; if the file is a text file with 'CR/LF' pairs at the end of each line, this will cause some regular expressions to fail. This option has no effect on platforms other than MS-DOS and
 MS-Windows.

'-z'
'--null-data'
     以"\0"作为输入行的分隔符，而不再以换行符分隔两行。
     (注：这为grep提供了简单的跨行匹配的能力。见后文示例14。)

2.2 Exit Status(退出状态码)
===============

通常情况下，如果能匹配到内容，则退出状态码为0，否则为1。但是如果发生了错误，则退出状态码为2，除非使用了"-s"或"-q"选项。

2.3 'grep' Programs(各种grep程序)
===================

有4种grep程序分别支持不同的搜索引擎，使用下面4个选项可以选择使用哪种grep程序。

'-G'
'--basic-regexp'
     使用基础正则表达式引擎解析PATTERN，因此只支持基础正则表达式(BRE)。这是默认grep程序。

'-E'
'--extended-regexp'
     使用扩展正则表达式引擎解析PATTERN，因此支持扩展正则表达式(ERE)。('-E'是POSIX指定的选项。)

'-F'
'--fixed-strings'
     不识别正则表达式，而是使用字符的字面意义解析PATTERN，因此只支持固定字符串的精确匹配。('-F'是POSIX指定的选项。)

'-P'
'--perl-regexp'
     使用perl正则表达式引擎解析PATTERN，因此支持Perl正则表达式。但该程序正处于研究测试阶段，因此会给出一个警告。

     此外，"grep -E"和"grep -F"可分别简写为egrep和fgrep。但这两个简写程序是传统写法，已被废弃，虽仍支持，但只是为了兼容老版本程序。
     (注：还有zgrep和pgrep，但它们不是grep家族的程序，zgrep是gzip提供，pgrep用于查看进程名和pid的映射关系)

3 Regular Expressions(正则表达式)
*********************

正则表达式是一种用于描述字符串集合的表达式。正则表达式类似于算术表达式，也使用各种操作符组合各短小表达式。grep可以理解三种不同版本的正则表达式：基础正则表达式BRE、扩展正则表达式ERE和Perl正则表达式。下面所描述的是扩展正则表达式的内容，在后文会比较BRE和ERE的不同之处。而Perl正则功能更完整、性能更好，可以从pcresyntax(3)和pcrepattern(3)中获取详细信息，但有些操作系统中可能无法获取。

3.1 Fundamental Structure(基本结构)
=========================

基本结构块是匹配单个字符的正则表达式。大多数字符，包括字母和数字，都能自己匹配自己，例如给定正则表达式"a"，它能匹配字母a。所有的元字符都具有特殊意义，需要使用反斜线进行转义。

正则表达式可以使用下面几种方式来表示重复次数。

'.'
     点"."可以匹配任意单个字符。
     The period '.' matches any single character.

'?'
     可以匹配前一个条目0或一次。例如，"ca?b"可以匹配"cb"也可以匹配到"cab"，但不能匹配到"caab"。如果使用了分组，如"c(ca)?b"能匹配"cb"或"ccab"。

'*'
     匹配前一个条目0或任意多次。

'+'
     匹配前面的条目一次或多次。

'{N}'
     匹配前面的条目正好N次。

'{N,}'
     匹配前面的条目N次或更多次。即至少匹配N次。

'{,M}'
     匹配前面的条目最多M次。即匹配0到M次。

'{N,M}'
     匹配前面的条目N到M次。

两个正则表达式可以进行串联，串联后的匹配结果是这两个正则表达式的匹配结果进行的串联。例如正则表达式"ab"就是"a"和"b"串联后的正则。

两个正则表达式还可以使用竖线符号"|"进行连接，这表示二者选一，只要能匹配竖线两边任意一个正在表达式均可，若能同时匹配上也可。例如字符串"acx"、"bx"、"accb"均能被正则表达式"ac|b"匹配上，其中"accb"被同时匹配上。

重复次数的符号优先级高于串联高于二者选一符号"|"，使用括号可以改变优先级规则。

3.2 Character Classes and Bracket Expressions(字符类和中括号表达式)
=============================================

中括号正则表达式是使用"["和"]"包围的字符列表。它能匹配该列表中的任意单个字符。如果列表中的第一个字符是"^"，则表示不匹配该列表中的任意单个字符。例如，'[0123456789]'能匹配任意数字。

中括号中可以使用连字符"-"连接两个字符表示"范围"。例如，C字符集下的"[a-d]"等价于"[abcd]"。大多数字符集规则和字典排序规则一样，这意味着"[a-d]"不等价于"[abcd]"，而是等价于"[aBbCcDd]"。可以设置环境变量"LC_ALL"的值为C使得采取C字符集的排序规则。

最后，预定义了几个特定名称的字符类，它们都使用中括号包围。如下：

'[:alnum:]'
     匹配大小写字母和数字。等价于字符类'[:alpha:]'与字符类'[:digit:]'的和。

'[:alpha:]'
     字母字符类。匹配大小写字母。等价于字符类'[:lower:]'和字符类'[:upper:]'的和。

'[:blank:]'
     空白字符类。包括：空格和制表符。

'[:cntrl:]'
     控制字符类。在ASCII中，这些字符的八进制代码从000到037，还包括177(DEL)。

'[:digit:]'
     数字字符类。包括：'0 1 2 3 4 5 6 7 8 9'。

'[:graph:]'
     绘图类。包括：大小写字母、数字和标点符号。

'[:lower:]'
     小写字母类。包括：'a b c d e f g h i j k l m n o p q r s t u v w x y z'。

'[:print:]'
     打印字符类。包括：大小写字母、数字、标点符号和空格。等价于字符类'[:alnum:]'与字符类'[:punct:]'和空格的和。

'[:punct:]'
     标点符号类。包括：'! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ ' { | } ~'。

'[:space:]'
     空格字符类。包括：空格、制表符、垂直制表符、换行符、回车符和分页符。

'[:upper:]'
     大写字母类。包括：'A B C D E F G H I J K L M N O P Q R S T U V W X Y Z'。

'[:xdigit:]'
     十六进制类。包括：'0 1 2 3 4 5 6 7 8 9 A B C D E F a b c d e f'。

例如，"[[:alnum:]]"表示"[0-9A-Za-z]"，"[^[:digit:]]"表示[^0123456789]，"[ABC[:digit:]]"表示"[ABC0-9]"。注意，字符类必须包含在额外的中括号内。

中括号中的大多数元字符都丢失了它们特殊意义，而成为普通的字面符号。

']'
     该符号表示中括号的结束。如果要匹配该字面字符，则必须将其放在字符列表的最前面。即"[]...]"。

'[.'
     该符号表示排序符号的开始。
     (注：排序类需要在字符集中预先定义好才能使用。例如[.ab.]表示将“ab”作为整体匹配，不匹配a或b。但默认情况下，字符集里肯定是没有定义好"ab"这个排序整体的，所以无法使用)
     
'.]'
     表示排序符号的结束。

'[='
     表示等价类的开始。
     (注：例如，[=e=]表示将字母e的第一声和第三声等不同音节的同字母看成相同字符。)

'=]'
     表示等价类的结束。

'[:'
     表示字符类的开始。

':]'
     表示字符类的结束。

'-'
     该字符是范围连接符，因此要匹配该符号的字面意义，需要将其放在列表的最前面或最后面或作为范围的结束字符。
'^'
     该字符表示不在列表中的字符。如果想匹配该字符的字面意义，则必须不能放在列表的第一个字符。

3.3 The Backslash Character and Special Expressions(反斜线字符和特殊的表达式)
===================================================

反斜线"\"后使用特定的字符表示特殊意义，如下：

'\b'
     匹配单词边界处的空字符。
     (注：grep中的单词有数字、字母和下划线组成，其他所有字符都是单词的分隔符。)

'\B'
     和"\b"相反，表示匹配非单词边界的空字符。

'\<'
     匹配单词起始位置处的空字符。

'\>'
     匹配单词结束位置处的空字符。

     (注：所以\bWORD\b等价于\<WORD\>。另外，grep选项"-w"也表示匹配单词边界)

'\w'
     匹配单词成分的字符。是[_[:alnum:]]的同义词。

'\W'
     匹配非单词成分的字符，是[^_[:alnum:]]的同义词。

`\s'
     匹配空白字符，是[[:space:]]的同义词。

`\S'
     匹配非空白字符，是[^[:space:]]的同义词。

例如，"\brat\b"匹配被分割后的"rat"，"\Brat\B"匹配"crate"但不匹配"furry rat"。

3.4 Anchoring(锚定)
=============

脱字符"^"以及美元符"$"是锚定元字符，分别匹配行首和行尾的空字符。

3.5 Back-references and Subexpressions(后向引用和子表达式)
======================================

反向引用"\N"表示匹配前面第N个括号中的正则子表达式，其中N是单个数字。例如"(a)\1"表示"aa"。当使用二选一的操作符"|"时，如果分组不参与匹配过程，则后向引用将失败。例如"a(.)|b\1"将无法匹配"ba"。
如果使用"-e"或"-f FILE"指定了多个PATTERN，则每个pattern的后向序列值都相互独立。

(注：例如：'([ac])e\1|b([xyz])\2t'能匹配aea或cec，但不能匹配cea或aec，还能匹配bxxt或byyt或bzzt。但如果将"\2"换成"\1"，即'([ac])e\1|b([xyz])\1t'，将无法匹配b[xyz]at或b[xyz]ct，因为第一个括号在左边，无法参与右边的正则搜索。
(注：反向引用也称为后向引用或回溯引用)

3.6 Basic vs Extended Regular Expressions(基础正则和扩展正则)
=========================================

在基础正则表达式中，元字符'?'、'+'、'{'、'|'、'(',和')'都表示字面意思，取而代之的是加上反斜线的版本：'\?'、'\+'、'\{'、'\|'、'\('和'\)'。

4 Usage(使用示例)
*******

以下是一些GNU grep的使用示例：

     grep -i 'hello.*world' menu.h main.c

该命令用于列出menu.h和main.c中包含"hello"字符串且后面带有"world"字符串的所有行，hello和world中间可以有任意多个字符。注意正则表达式的"-i"选项使得grep忽略大小写，所以还能匹配"Hello, world!"。

   下面是一些使用grep时常见的问题和答案。

  1. 如何列出匹配的文件名？

          grep -l 'main' *.c

     将列出当前目录下所有以".c"结尾且文件中包含'main'字符串的文件名。

  2. 如何递归搜索目录？

          grep -r 'hello' /home/gigi

     搜索/home/gigi目录下所有文件，且文件中包含'hello'字符串。如果要灵活控制搜索的文件，可以结合find和xargs命令一起使用。例如下面的例子仅搜索C源文件。

          find /home/gigi -name '*.c' -print0 | xargs -0r grep -H 'hello'

     这不同于下面的命令：

          grep -rH 'hello' *.c

     这仅仅只是搜索当前目录下以".c"结尾的文件。此处的"-r"选项基本上算是多余的，除非当前目录下有以".c"结尾的目录，但这是很少见的情况。上面的find命令更类似于下面的命令：

          grep -rH --include='*.c' 'hello' /home/gigi

  3. 如果pattern以短横线"-"开头会如何？
  
          grep -e '--cut here--' *

     将搜索"--cut here--"。但如果不给定"-e"选项，grep将可能把"--cut here"解析成一系列的选项。

  4. 如何搜索整个单词，而不是单词中的一部分？

          grep -w 'hello' *

     这将搜索当前目录下所有文件，并找出包含"hello"整个单词的文件，它无法匹配"Othello"。更灵活的控制可以使用"\<"和"\>"来匹配单词的开始和结尾。例如：

     grep 'hello\>' *

     仅搜索"hello"结尾的单词，因此可以匹配"Othello"。

  5. 如何输出匹配行的上下几行？

          grep -C 2 'hello' *

     这将输出匹配行以及它的前后两行。

  6. 如何强制grep即输出匹配行又输出文件名？

     只需在文件列表中加上'/dev/null'即可。

          grep 'eli' /etc/passwd /dev/null

     将得到:

          /etc/passwd:eli:x:2098:1000:Eli Smith:/home/eli:/bin/bash

     还可以使用GNU扩展选项"-H"：

          grep -H 'eli' /etc/passwd

  7. 为什么有人在ps的后面使用奇怪的正则表达式？

          ps -ef | grep '[c]ron'

     如果pattern中不加上中括号，将匹配包含cron字符串的进程，包括grep自身，因为grep命令的表达式中包含了cron字符串。但如果加上了中括号，则grep命令行中包含的是"[c]ron"字符串，而grep所匹配的字符串是cron而不是[c]ron。
     在输出结果上，这其实等价于下面这条命令：
     
          ps -ef | grep 'cron' | grep -v 'grep'

  8. 为什么grep的结果中会报告"Binary file matches"？

     如果grep列出二进制文件中的所有匹配行，将很可能生成一大堆乱七八糟的无用信息，因此GNU的grep默认禁止这样的输出。如果想要输出二进制内容，使用"-a"或"--binary-files=text"选项。

  9. 为什么'grep -lv'输出的是包含非匹配行的文件名？

     'grep -lv'列出的是包含一行或多行非匹配行的文件名。如果想要列出无匹配内容的文件名，则使用"-L"选项。
     
     (注：例如a.txt中一部分行匹配到了，一部分行没匹配到，而b.txt中完全没有匹配上，则grep -lv将输出a.txt，而不是b.txt。因此可推测"-v"选项的操作优先级要高于"-l"，即先搜索出反转行，再输出包含这些反转行的文件)

 10. 使用"|"可以实现or逻辑，如何实现AND逻辑？

          grep 'paul' /etc/motd | grep 'franc,ois'

     将搜索出同时包含"paul"和"franc,ois"的所有行。

 11. 如何同时搜索文件和标准输入？

     只需使用"-"代替标准输入的文件名即可：

          cat /etc/passwd | grep 'alain' - /etc/motd

 12. 正则表达式中如何表达出回文结构？(注：回文结构表示正读和反读的结果是一样的，例如12321,abcba)

     可以使用反向引用来实现。例如，一个4字符的结构使用BRE来实现：

          grep -w -e '\(.\)\(.\).\2\1' file

     它可以匹配单词"radar"或"civic"。

     Guglielmo Bondioni提出了一个正则表达式，可以搜索长达19个回文结构的字符串，其中使用了9个子表达式和9个反向引用。因为BRE或ERE最多只支持9个反向引用。 
     
          grep -E -e '^(.?)(.?)(.?)(.?)(.?)(.?)(.?)(.?)(.?).?\9\8\7\6\5\4\3\2\1$' file

 13. 为何反向引用会失效？

          echo 'ba' | grep -E '(a)\1|b\1'

     这不会输出任何内容，因为左边的表达式"(a)\1"无法匹配，因为输入数据中没有"aa"，因此右边的"\1"无法引用任何内容，意味着将不匹配任何东西。(此例中右边表达式仅在左边表达式成功匹配时才能生效。)
     
     注：经测试，即使左边表达式能匹配上，右边表达式中引用左边的分组时也无效。例如"echo 'baaca' | grep -E '(a)\1|c\1'"可以匹配大其中的"aa"，但却匹配不到"ca"。

 14. grep如何跨行匹配？

     标准的grep无法实现该功能，因为它是基于行读取的。因此，仅仅使用字符类"[:space:]"无法如你想象中那样匹配换行符。

     GNU的grep有一个选项"-z"，它可以处理使用"\0"结尾的行。因此，可以匹配输入数据中的换行符，但通常很可能在输出结果时，输出的是所有内容而不仅是被匹配的行，因此经常需要结合输出控制选项如"-q"来使用。例如：

          printf 'foo\nbar\nabc' | grep -z 'foo[[:space:]]\+bar'
          printf 'foo\nbar\nabc' | grep -z -q 'foo[[:space:]]\+bar'

     如果这还不满足需求，可以将输入数据进行格式转换然后交给grep，或者使用其他工具替代grep，如"sed"、"awk"、"perl"或其他很多工具都能跨行操作。

 15. What do 'grep', 'fgrep', and 'egrep' stand for?

     The name 'grep' comes from the way line editing was done on Unix. For example, 'ed' uses the following syntax to print a list of matching lines on the screen:

          global/regular expression/print
          g/re/p

     'fgrep' stands for Fixed 'grep'; 'egrep' stands for Extended 'grep'.

5 Known Bugs(已知的一些bug)
==============

当"{n,m}"指定的重复次数很多时，将导致grep消耗大量内存。此外，越模糊的正则表达式消耗的时间和空间越多，也会让grep消耗大量内存。
反向引用的功能非常慢，因此可能会消耗大量时间。
(注：递归搜索时，也会消耗巨量的内存，很容易提示内存溢出错误而提前退出。)
```