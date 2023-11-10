---
title: Perl一行式：处理空白符号
p: perl/perl_oneliner_1.md
date: 2019-07-08 17:39:49
tags: Perl
categories: Perl
---

# Perl一行式：处理空白符号

**perl一行式程序系列文章**：[Perl一行式](/perl/index#blogperloneline)

-----------------------------

假如文件file.log内容如下：
```
root   x 0     0 root   /root     /bin/bash
daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
bin    x 2     2 bin    /bin      /usr/sbin/nologin
sys    x 3     3 sys    /dev      /usr/sbin/nologin
sync   x 4 65534 sync   /bin      /bin/sync
```

<a name="blog1546583770"></a>
## 每行后加一空行

```
$ perl -pe '$\ = "\n"' file.log
```
结果：
```
root   x 0     0 root   /root     /bin/bash

daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin

bin    x 2     2 bin    /bin      /usr/sbin/nologin

sys    x 3     3 sys    /dev      /usr/sbin/nologin

sync   x 4 65534 sync   /bin      /bin/sync

```

这里出现了选项 -p 和 -e，出现了特殊变量`$\`，附带的，稍后还会解释另一个选项 -n。

perl的**`-e`选项**表示后面接perl的一行式表达式，就像sed的-e选项一样。这是一行式perl程序最常见的一个选项。perl还有一个`-E`选项，和`-e`一样都用来指定一行式表达式，但`-E`可以使用一些高版本的功能。

perl的**`-p`选项**表示print操作，即对每一读入的行在经过表达式操作后都默认输出，和sed的p命令是一样的。

实际上，perl中的`-p`选项等价于下面的逻辑，理解了这个逻辑，对理解sed的逻辑会很有帮助。
```
while(<>){
    ... -e 指定的表达式代码在这里 ...
} continue {
    print or die "-p failed: $!\n";
}
```

Perl中的continue和其它语言的continue有点不一样，Perl中的continue表示每轮循环的主体执行完之后，执行另一段代码。也就是说，每一行内容经过`-e`指定的表达式处理后，都会被continue代码块中的print输出。

解释下-p选项的过程：`while(<>)`每次读取一行数据赋值给默认变量`$_`，然后经过-e的表达式进行处理，处理完后执行continue的print，这里print没有参数，所以表示输出默认变量`$_`的内容，也就是被处理后的行数据。

另一个选项 -n ，表示处理文件但默认不输出处理后的行。如果想要输出，只能自己在-e表达式中指定输出操作(print/say/printf)。逻辑为：
```
while (<>) {
    ...  # -e expression here
}
```

也就是说，-n和-p两个选项会自动读取文件（如果都存在，则-p会覆盖-n），不需要在-e的表达式中自己写读取文件的逻辑。如果没有这两个选项，那么在-e中要自己写才能读参数文件：
```
$ perl -e 'while(<>){...}'
```

一般来说，选择使用-p还是-n的规则如下：  
- **某些行不需要输出或者需要被删除的时候，不应该使用-p，因为它默认会输出所有行**  
    - 换句话说，**如果不需要输出所有行时，不使用-p，需要输出所有行，可以考虑使用-p**  
- **不使用-p的时候，几乎都可以使用-n**  


最后是关于特殊变量`$\`表示print的输出行分隔符(awk的ORS变量)，它默认为undef，所以print输出的每段数据之间都是紧连在一起的。此处示例将`$\`指定为换行符。

由于`<>`读取数据时已经将文本中每一行的`\n`也读取了，所以加上`$\`已经有两个连续的`\n`，于是每行后面都会多一空行。

实际上没有必要为每一读入的行都设置`$\`，可以将它设置在BEGIN块中：
```
$ perl -pe 'BEGIN{$\ = "\n"}' file.log
```

对于本示例【每行之后加上空行】有多种解决方式。例如：
```
$ perl -pe '$_ .= "\n"' file.log
$ perl -nE 'say "$_\n"' file.log
$ perl -pe 's/$/\n/' file.log
......
```
<a name="blog1546583770"></a>
## 每行后加空行，但空行除外

```
$ perl -pe '$_ .= "\n" unless /^$/'
```

这里使用unless逻辑进行空行匹配，如果匹配空行，就不对当前行追加尾随换行符。unless测试等价于if的否定测试。

有些空行可能是包含了空白符号(空格、制表符等)的行，这些空白肉眼不可识别，但却占了字符空间，使得无法匹配`^$`，所以更好的匹配模式是：
```
$ perl -pe '$_ .= "\n" if /\S/'
```

`\S`表示任意单个非空白字符，`\s`表示任意单个空白字符。所以这里的逻辑是：只要行能匹配非空白字符，就追加尾随换行符。

<a name="blog1546583770"></a>
## 每行后加N空行

想在每行后面加两空行、三空行、四空行、N空行该如何解决？

```
$ perl -pe '$_ .= "\n" x 3' file.log
```

Perl的字符串可以使用小写字母`x`表示重复N次，例如`"a" x 3`得到"aaa"，`"ab" x 2`得到`abab`。上面的示例表示为每行都追加3个换行符。

通过字符串重复操作，可以很轻松地输出等长的分割线：
```
$ perl -e '
        print "-" x 30,"\n";
        print "hahaha\n";
        print "-" x 30,"\n";
        print "heihei\n";'
------------------------------
hahaha
------------------------------
heihei
```

<a name="blog1546583770"></a>
## 每行前加空行

最简单的方式是使用s替换操作。
```
$ perl -pe 's/^/\n/' file.log
```

<a name="blog1546583770"></a>
## 移除所有空白行

```
$ perl -ne 'print unless /^$/' file.log
```

此处使用了`-n`选项，表示禁止默认的输出。print和匹配操作的对象都使用默认变量`$_`。等价于：

整个逻辑是：只要匹配了空行`/^$/`，就不输出。

这也有好几种方式可以实现，例如：
```
$ perl -ne 'print if /\S/' file.log
```

比较独特的一种实现方式是使用length函数：
```
$ perl -lne 'print if length' file.log
```

这里涉及选项`-l`和函数length()，且print和length都没有指定操作对象，所以使用默认变量`$_`，等价于：
```
$ perl -lne 'print $_ if length $_' file.log
```

length函数可以获取字符串的字符个数，注意是字符数不是字节数。

-l选项在结合-n或-p使用的时，会自动对读入的行移除尾随换行符，然后在输出的时候自动追加尾随输出分隔符(如换行符，如何追加分隔符请参看[Perl一行式参考手册](/perl/perl_options_vars) )。

这里的逻辑是：如果是空行，那么在被-l移除换行符后length返回0，也就是布尔假，所以只有不是空行的行才会被输出。

<a name="blog1546583770"></a>
## 压缩连续空行：按段落读取

先准备一段测试数据paragraph.log：
```
first paragraph:
        first line in 1st paragraph
        second line in 1st paragraph
        third line in 1st paragraph


second paragraph:
        first line in 2nd paragraph
        second line in 2nd paragraph
        third line in 2nd paragraph


third paragraph:
        first line in 3rd paragraph
        second line in 3rd paragraph
        third line in 3rd paragraph
```

sed/awk中想要压缩连续空行，总要多读入几行进行连续空行的判断。例如:
```
$ sed -nr '$!N;/^\n$/!P;D' paragraph.log
```

但在perl一行式中，这会变得无比的简单：
```
$ perl -00pe '' paragraph.log
```

这里两个关注点：`-00`和`-e ''`。

`-e ''`的表达式部分为空，表示什么也不做。什么也不做的时候，也可以写成`-e0`。
```
$ perl -00pe0 paragraph.log
```

`-0OCTNUM`表示设置输入行分隔符`$/`。

如果省略8进制值OCTNUM，则-0表示设置`$/`为undef，即`$/ = undef`，也就是一次性从文件头读到文件尾当作一行赋值给`$_`。

这里指定了8进制的值为0，对应于ASCII的空字符串，即等价于`$/ = ""`，它表示**按段落读取**(slurp读取模式)，并压缩连续的空行为单个空行。

什么是段落？中间隔了至少一空行的是上下两个段落，段落意味着可能包含了连续的多行。但是如果隔了连续的空行呢？设置`$/ = ""`会按段落读取，并压缩连续的空行为单空行，然后作为上面的段落的一部分。设置`$/ = "\n\n"`也表示按段落读取，但它不会压缩连续的空行。

如何知道是否是按段落读取？可用下面的示例进行测试：
```
$ perl -ne '
    BEGIN{$/ = "";}
    print $_."xxxxx" if /2nd/' paragraph.log
```

会发现追加的几个字符`xxxxx`是单独附加在第二段落的尾部的，而不是能匹配`2nd`的每一行上。

<a name="blog1546583770"></a>
## 压缩/扩展所有连续空行为N空行

在上面一节压缩连续空行的基础上，实现这个目的已经非常容易了：
```
$ perl -00pe '$_ .= "\n" x 3' paragraph.log
```

这表示将每个段落之间规范为4个连续的空行进行分隔。之所以是4空行而不是3，是因为压缩成单空行后，又追加了3空行。

<a name="blog1546583770"></a>
## 压缩/扩展单词间的空格数量

要实现这样的功能，这个对于sed来说也非常的容易。这里给几个简单示例。

1.每行单词间的空给双倍化：每个空白都扩成2空格
```
$ perl -lpe 's/ /  /g' file.log
```

2.移除每行单词间的所有空白
```
$ perl -lpe 's/ //g' file.log
```

3.每行单词间连续空白压缩为单空格
```
$ perl -lpe 's/\s+/ /g' file.log
```

4.所有字符间插入一个空格
```
$ perl -lpe 's// /g' file.log
```

注意，上面插入空格时，也会在行首和行尾插入空格符号。

<a name="blog1546583771"></a>
## 直接修改文件

perl的`-i`选项可以用来原地修改、拷贝副本。用法和sed的`-i`一致。

例如：
```
$ perl -i".bak" -lpe 's/$/\n/g' file.log

$ cat file.log
root   x 0     0 root   /root     /bin/bash

daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin

bin    x 2     2 bin    /bin      /usr/sbin/nologin

sys    x 3     3 sys    /dev      /usr/sbin/nologin

sync   x 4 65534 sync   /bin      /bin/sync

$ cat file.log.bak
root   x 0     0 root   /root     /bin/bash
daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
bin    x 2     2 bin    /bin      /usr/sbin/nologin
sys    x 3     3 sys    /dev      /usr/sbin/nologin
sync   x 4 65534 sync   /bin      /bin/sync
```




