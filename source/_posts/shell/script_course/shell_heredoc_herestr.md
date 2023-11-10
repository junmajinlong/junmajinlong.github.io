---
title: Shell脚本深入教程：Here Document和Here String用法
p: shell/script_course/shell_heredoc_herestr.md
date: 2020-05-19 14:17:01
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Here Document和Here String的用法

这两种功能都是重定向部分的内容，但很基础，所以在此介绍。简单概括它们：
- here document：将指定的一行或多行文档内容作为标准输入  
- here string：将指定的字符串作为标准输入  

here doc使用`<<`作为语法符号，here string使用`<<<`作为语法符号，标准重定向使用`<`作为语法符号，它们的作用都是将指定的内容作为标准输入。

## Here String

语法：
```bash
<<<"String"
# 或者
<<<'String'
```

`<<<`表示here string，也就是说该符号后面是一个字符串，它将作为标准输入。

例如：
```bash
cat <<<"www.junmajinlong.com"
```

例如，有些版本的passwd命令可以接受`--stdin`选项表示从标准输入中读取密码。下面两个命令等价：
```bash
$ passwd --stdin user <<< password_value
$ echo password_value | passwd --stdin user
```

使用单引号包围here string时，不会进行变量替换、命令替换等，使用双引号包围时会进行替换。
```bash
$ a=3333
$ cat <<<$a            
3333
$ cat <<<"hello world$a"
hello world3333
$ cat <<<'hello world$a' 
hello world$a
```

here string常可以替代管道前的echo命令`echo xxx |`。例如：
```bash
# 下面是等价的
echo hello world | grep "llo"
grep "llo" <<<"hello world"
```

## Here Document

写脚本的时候，Bash Script中经常需要将多行数据写入文件或管道(即将多行数据作为标准输入)，当然可以每行一个echo或printf，但是麻烦，且破坏了多行数据的整体性和关联性。这时候需要用到here doc的功能。

here document表示这里是一个文档。既然是文档，就会有文档起始标记和文档终止标记，起始和终止标记中间的内容，属于文档内容。

标准语法和基本常用语法如下：

```bash
# 标准语法
[n]<<[-]word
    here-document
delimiter

# 常用语法
<<eof
DATA LINE 1
DATA LINE 2
eof
```

其中`<<word`是固定的，`<<`表示这里是here document的语法，word是文档起始标记，是什么字符做起始标记可随意，常用的是不区分大小写的`EOF EOL EOS EOB END`等，可随意字符，但文档的结束标记也需要和去除引号后的word相同，且结束标记必须在行首(不能有前缀空白)。

> EOF EOL EOS END都是什么意思，shell中常用的是eof，但剩余几个比较常出现在其它语言中，比如Perl和Ruby都支持功能更为丰富的here document语法。
> - EOF：end of file，表示文件/文档到此结束
> - EOL：end of line，表示行到此结束
> - EOS：end of string，表示字符串到此结束
> - EOB：end of block，表示块到此结束
> - END：end，表示到此结束

例如，通过heredoc方式将两行数据追加写入/tmp/a.log，其中EOF是文档起始标记，结束标记也是EOF且出现在行首，中间的两行内容是文档内容，文档数据将作为标准输入被cat读取，并通过追加重定向写入/tmp/a.log文件。
```bash
$ cat >>/tmp/a.log <<EOF
LINE 1
LINE 2
EOF

$ cat /tmp/a.log
LINE 1
LINE 2
```

如果使用`<<-word`语法，则文档内容和结束标记前缀制表符会被忽略，注意是制表符前缀才被忽略，空格前缀是不会被处理的。

下面假设`<TAB>`是制表符，`<SP>`是空格，则：
```bash
$ cat >/tmp/a.log <<-eof
<SP><SP><SP><SP>line 1
<TAB><TAB>line 2
<TAB>eof

$ cat /tmp/a.log
    line 1
line 2
```

如果文档起始标记使用引号包围，即`<<'EOF'`或者`<<"EOF"`，不管是单引号还是双引号，则文档内容部分将不做任何变量替换、算术替换、命令替换等操作，但不加引号时，默认会进行这些替换：

```bash
$ a=333
# 输出 hello 333
$ cat <<eof
hello $a
eof

# 输出 hello $a
$ cat <<"eof"
hello $a
eof
```

有些人会将here document作为脚本的多行注释语法。虽然语法可行，但是不建议，因为它是正常的语句，当作注释时，容易搞混淆。

```bash
<<COMMENT
  comment line 1
  comment line 2
COMMENT
```

## Here Document的各种位置摆放姿势

虽说here document作为文档可以当作一个文件名来看待，但因为它是以重定向的方式存在的，使得它有各种使用姿势。

**1.将here document内容写进文件**。

```shell
cat >>a.log <<EOF
abc
def
EOF

# 或者
cat <<EOF>>a.log
abc
def
EOF
```

这就是here document作为一种重定向语法的好处，使用here document时，它的位置可以随意摆放，而不像使用文件名，位置固定死了。

**2.将here document内容放进管道**。

```shell
# 方案一
cat <<EOF | awk
abc
def
EOF

# 方案二
( cat | awk ) <<EOF
abc
def
EOF

# 方案三
{ cat | awk } <<EOF
abc
def
EOF

# 方案四
{
cat <<EOF
abc
def
EOF
} | awk

# 方案五
(
cat <<EOF
abc
def
EOF
) | awk
```

其实只要理解了here document是一种重定向，位置可随意，就很容易理解上面几种方案。

**3.多个here document嵌套**。

在Perl和Ruby语言中，都支持多个Here Document嵌套的语法，有时候这非常方便。但是在bash中，也可以实现嵌套。

```shell
(cat ; echo '======='; cat <&3) <<EOF 3<<eof
abc
def
EOF
ghi
jkl
eof
```