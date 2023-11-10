---
title: Shell脚本深入教程：Bash IFS用法详解
p: shell/script_course/shell_ifs.md
date: 2023-07-12 14:17:28
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash IFS用法详解

IFS的作用概括起来非常简单：

- 在变量替换(扩展)、命令替换(扩展)、算术替换(扩展)时，如果它们的结果没有使用引号包围，则尝试使用IFS将结果进行单词划分  
- 在`read`命令中，根据IFS将所读取的内容划分为单词分别赋值给指定的变量

`man bash`对IFS的解释原文：
```
The Internal Field Separator that is used for word splitting
after expansion and to split lines into words with the read
builtin command. The default value is <space><tab><newline>
```

也就是说，IFS的作用是在各种扩展之后将扩展的结果划分为命令行中的单词，以及将read命令读取的行划分为多个单词。

当然，在`man bash`中的其他地方还能看到IFS的其它用法：`$*`或`${VAR[*]}`时，也将使用IFS，当然这也属于变量扩展的范畴，只是有点不一样，见下文。

## IFS常用示例

IFS最常用于read读取行时将所读行的内容划分为多个字段赋值给各个变量，类似于`awk -F`或`cut -d`等指定字段分隔符的用法。

```bash
# 输出 v1: f1, v2: f2, v3: f3
$ echo 'f1:f2:f3' | (IFS=':';read v1 v2 v3;echo "v1: $v1, v2: $v2, v3: $v3")
```

用于变量扩展、命令替换、算术扩展等：
```bash
# 将以冒号分隔的变量值各个字段保存到数组
data="abc:def:gh"
# 先备份IFS
old_ifs="$IFS"
# 再设置IFS
IFS=":"
# $data进行了变量替换，会按照":"划分为三个字段，分别保存到arr数组中
read -a arr <<<$data
# 最后恢复IFS
IFS="$old_ifs"

# 查看arr中保存的值，输出：
# declare -a arr=([0]="abc" [1]="def" [2]="gh")
declare -p arr
```

这样先备份再恢复的写法很啰嗦，不恢复IFS的话会影响后续shell环境。为了只让修改IFS影响它需要的命令或上下文，除了备份原有IFS值并在后面恢复之外，还可以使用子shell的方式设置IFS，它不会影响父shell环境，并且IFS是bash内置变量，所以也可以直接在命令行前设置其值使其仅影响当前命令：

```bash
data="abc:def:gh"

# 方式1：小括号开启子shell，小括号种修改IFS，不会影响小括号外面的IFS
(
  IFS=":" 
  read -a arr <<<$data
  declare -p arr
)

# 方式2：直接当作变量修改，注意IFS在read命令前
IFS=":" read -a arr <<<$data
declare -p arr
```

IFS也用于 `while read` 结构

```bash
$ cat <<EOF >a.txt
long|18|www.junmajinlong.com
fei|28|www.snax.com
EOF

# 设置IFS为竖线，read读取行时会将竖线作为字段分隔符，并赋值给指定的变量
# 注意，IFS在read命令前
cat a.txt | while IFS="|" read -r user age site;do 
  echo "user: $user, age: $age, site: $site"
done
```
输出：
```
user: long, age: 18, site: www.junmajinlong.com
user: fei, age: 28, site: www.snax.com
```

或用于for遍历某个带分隔符的变量：

```bash
# 下面分三行分别输出 aa bb cc
(
  data="aa:bb:cc"
  IFS=":"
  for x in $data;do
    echo $x
  done
)
```

IFS也会影响带双引号的`$*`或对应数组的用法`${arr[*]}`等包含`*`号的数组类**输出结果**。例如：

```bash
set -- aa bb cc
IFS=":" echo ${*}   # 输出：aa bb cc
IFS=":" echo "${*}" # 输出：aa:bb:cc

arr=(xx yy zz)
IFS=":" echo ${arr[*]}    # 输出：xx yy zz
IFS=":" echo "${arr[*]}"  # 输出：xx:yy:zz
IFS=":" echo "${!arr[*]}" # 输出：0:1:2
```

## 深入理解IFS

完整的规则，man bash中的`Word Splitting`段落中解释清楚了：
```
The shell treats each character of IFS as a delimiter,
and splits the results of the other expansions into 
words using these characters as field terminators. If
IFS is unset, or its value is exactly <space><tab><newline>,
the default, then sequences of <space>, <tab>, and <newline>
at the beginning and end of the results of the previous
expansions are ignored, and any sequence of IFS characters
not at the beginning or end serves to delimit words. If IFS
has a value other than the default, then sequences of the
whitespace characters space, tab, and newline are ignored at
the beginning and end of the word, as long as the whitespace
character is in the value of IFS (an IFS whitespace character).
Any character in IFS that is not IFS whitespace, along with
any adjacent IFS whitespace characters, delimits a field. 
A sequence of IFS whitespace characters is also treated as a 
delimiter. If the value of IFS is null, no word splitting occurs
```

解释总结一下规则：  
- IFS未设置时的默认值为`<space><tab><newline>`，即空格、制表符、换行符，则扩展结果中的连续前缀空白和连续后缀空白都被忽略，且非前缀和后缀的连续空白将被认为压缩为单个单词分隔符。
- 如果IFS设置为其它值，只要其中含有空格，扩展结果中的前缀和后缀空白也被忽略。
- 如果IFS设置为其它值，只要其中含有空格，则其它非空格字符如果与空格相邻，则划分为一个字段。也就是说这个非空格字符和相邻的一个或多个连续的空格将被认为压缩为单个单词分隔符。
- 如果IFS设置为其它值，只要其中含有空格，则连续的空格也会被认为是单个分隔符。
- 如果IFS为空，即设置为`IFS= IFS="" IFS=''`，则不划分单词。

上面所说的前缀后缀空白被忽略的忽略二字有点不准确，应该是是压缩为单个单词分隔符。也就是说，如果有连续前缀空白或连续后缀空白，则被压缩为单个前缀分隔符或单个后缀分隔符。

并且由上面的规则可见，只要IFS中含有空格，它就会变得特殊。

此外注意，只有没有使用引号包围时的变量替换、命令替换、算术替换，才会进行单词划分，使用了引号后，是不划分单词的。

下面用示例来解释上面的规则。

## 示例解释IFS的规则

下面示例中，IFS为默认值，前缀和后缀空白实际上都不是被忽略，而是被压缩为单个空格，并且非前缀和非后缀连续空白的也一样被压缩为单个空格。
```bash
$ (cmd="   echo a   b    c   ";echo x${cmd}x )
x echo a b c x

# 与上面的结果做对比：
$ (cmd="echo a   b    c";echo x${cmd}x )
xecho a b cx
```

下面设置IFS为冒号，进行变量替换后根据IFS划分单词，所以变量替换的结果被替换为三个单词：
```bash
# echo $path 进行变量替换后，命令等价于`echo a b c`
$ (IFS=":" ; path="A:B:C" ; echo $path)
A B C
```

下面将IFS设置为逗号，则`$cmd`在进行变量替换后，等价于`echo a b    c`，最后搜索echo命令并执行。
```bash
$ (IFS="," ; cmd="echo,a,b   c";$cmd)
a b   c
```

下面`IFS`设置为逗号和空格，因为IFS中含有空格，将忽略变量扩展结果中的连续空白前缀和后缀(实际上是被压缩为单个分隔符)，且因为b后面的逗号和连续的空格相邻，所以逗号和连续的空格被当作单个单词分隔符。
```bash
(IFS=", " ; cmd="   echo,a,b,   c     ";echo x${cmd}x )
x echo a b c x
```

下面`IFS`设置为逗号和空格，连续空白的前缀和后缀都被压缩为单个分隔符，且a后面的逗号和连续的空格相邻，它们被压缩为单个分隔符，b后面的逗号跟了相邻的一大堆空格，空格后又跟了一个逗号(c前面)，所以b后面的逗号和相邻的空格被压缩为一个分隔符，它后面的逗号(c前面)也是一个分隔符，所以输出结果中b和c之间是两个空格。
```bash
$ (IFS=", " ; cmd="   echo,a,    b,      ,c     ";echo x${cmd}x )
x echo a b  c x
```

下面`IFS`设置为逗号和空格，a和b之间的逗号两边都相邻了连续的空白，它们全被压缩当作单个分隔符，c前面的逗号前相邻了连续空白，也被压缩为单个分隔符，所以输出结果中，a和b和c之间都是单个空格。
```bash
$ (IFS=", " ; cmd="echo,a    ,    b      ,c";echo x${cmd}x )
xecho a b cx
```