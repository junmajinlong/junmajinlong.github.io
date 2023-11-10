---
title: Shell脚本深入教程：Bash操作变量和数组元素的方式
p: shell/script_course/shell_op_var_arr.md
date: 2020-05-19 14:18:02
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash操作变量、数组元素

所有对变量、数组进行的操作，都是在Shell解析命令行的变量替换阶段完成。

## 变量替换

下面都有省略冒号版本的，表示如果parameter未定义，则怎样怎样。
```
${parameter:-word}
如果parameter未定义或者为空，则用word做变量替换，否则用parameter的值做变量替换(不会赋值)。

${parameter:+word}
如果parameter未定义或者为空，则不做变量替换，否则用word做变量替换。

${parameter:?word}
如果parameter为空或未定义，则报错，如果是脚本则退出。如果已定义且非空，则直接变量替换。

${parameter:=word}
如果parameter未定义或者为空，则先将word赋值给parameter，然后再用parameter的值做变量替换。

${parameter-word}
${parameter+word}
${parameter?word}
${parameter=word}
如果parameter未定义，则按照上面对应的方式进行操作  
```

常用的是`=`版本的，用来判断变量是否已定义或是否为非空变量。

## 变量长度或数组长度

![](/img/shell/1589872183776.png)


## 子串、子元素内容截取

```
${parameter:offset}
${parameter:offset:length}
```
可操作普通变量、位置变量、`$@`、`$*`、数组。offset指定起使位置，可为负数值表示从尾部开始计算。length为正数时表示截取长度，为负数时表示从尾部开始计算的结尾偏移位置。不指定length时表示从offset开始截取到结尾。

当操作普通变量时：
```shell
$ string=01234567890abcdefgh
$ echo ${string:7}
7890abcdefgh
$ echo ${string:7:0}

$ echo ${string:7:2}
78
$ echo ${string:7:-2}   # index=7到index=-2
7890abcdef
$ echo ${string: -7}    # index=-7
bcdefgh
$ echo ${string: -7:0}

$ echo ${string: -7:2}  # index=-7，且长度为2
bc
$ echo ${string: -7:-2} # index=-7到index=-2
bcdef
```
当操作位置变量或数组某元素时：
```shell
$ set -- 01234567890abcdefgh
$ echo ${1:7}
7890abcdefgh
$ echo ${1:7:0}

$ echo ${1:7:2}
78
$ echo ${1:7:-2}
7890abcdef
$ echo ${1: -7}
bcdefgh
$ echo ${1: -7:0}

$ echo ${1: -7:2}
bc
$ echo ${1: -7:-2}
bcdef

$ array[0]=01234567890abcdefgh
$ echo ${array[0]:7}
7890abcdefgh
$ echo ${array[0]:7:0}

$ echo ${array[0]:7:2}
78
$ echo ${array[0]:7:-2}
7890abcdef
$ echo ${array[0]: -7}
bcdefgh
$ echo ${array[0]: -7:0}

$ echo ${array[0]: -7:2}
bc
$ echo ${array[0]: -7:-2}
bcdef
```
当操作`$@`或`$*`时，index=0从`$0`开始，且length必须不能小于0，否则报错：
```shell
$ set -- 1 2 3 4 5 6 7 8 9 0 a b c d e f g h
$ echo ${@:7}
7 8 9 0 a b c d e f g h
$ echo ${@:7:0}

$ echo ${@:7:2}
7 8
$ echo ${@:7:-2}
bash: -2: substring expression < 0
$ echo ${@: -7:2}
b c
$ echo ${@:0}
./bash 1 2 3 4 5 6 7 8 9 0 a b c d e f g h
$ echo ${@:0:2}
./bash 1
$ echo ${@: -7:0}
```
当直接操作数组本身时，index=0从第一个元素开始，且length必须不能小于0，否则报错。
```shell
$ array=(0 1 2 3 4 5 6 7 8 9 0 a b c d e f g h)
$ echo ${array[@]:7}
7 8 9 0 a b c d e f g h
$ echo ${array[@]:7:2}
7 8
$ echo ${array[@]: -7:2}
b c
$ echo ${array[@]: -7:-2}
bash: -2: substring expression < 0
$ echo ${array[@]:0}
0 1 2 3 4 5 6 7 8 9 0 a b c d e f g h
$ echo ${array[@]:0:2}
0 1
$ echo ${array[@]: -7:0}
```

## 变量、子元素内容删除

```shell
${parameter#word}
${parameter##word}
${parameter%word}
${parameter%%word}
```

返回被删除部分数据后的结果。

如果是普通变量，则：

- `#`表示从左向右非贪婪删除word匹配成功的部分  
- `##`表示从左向右贪婪删除word匹配成功的部分  
- `%`表示从右向左非贪婪删除word匹配成功的部分  
- `%%`表示从右向左贪婪删除word匹配成功的部分  

```shell
$ file_name="Linux.docx.jpg"
$ echo ${file_name%%.*}
Linux
$ echo ${file_name%.*}
Linux.docx
$ echo ${file_name##*.}
jpg
$ echo ${file_name#*.}
docx.jpg
```

如果是位置变量`@`或`*`(即`$@#、$@##、$@%、$@%%`或`$*#、$*##、$*%、$*%%`)，则表示对每个位置参数都进行匹配删除操作。

```shell
$ set -- xabca xabcb xabcc
$ echo ${@##*abc}     
a b c
```

如果是数组且带索引`@`或`*`，则表示对每个数组元素都进行匹配删除操作。例如：

```shell
$ arr[1]=xabca
$ arr[2]=xabcb
$ arr[3]=xabcc
$ arr[4]=xabcd
$ arr[5]=xabce
$ arr[6]=xabcf
$ arr[7]=xabcg
$ echo ${arr[@]##*abc}
a b c d e f g
```

## 变量、子元素内容替换

```shell
# 注：均为贪婪匹配
${parameter/pattern/replacement}   # pattern匹配到的内容非全局替换成replacement
${parameter//pattern/replacement}  # pattern匹配到的内容全局替换成replacement
${parameter/#pattern/replacement}  # pattern必须匹配开头，否则无效
${parameter/%pattern/replacement}  # pattern必须匹配结尾，否则无效
${parameter/pattern/}              # pattern匹配的直接删除
${parameter/pattern}               # pattern匹配的直接删除
```

如果parameter是普通变量，则用于替换或删除所匹配成功的内容。

如果parameter是位置参数的`@`或`*`，则替换删除操作作用于每个位置变量。

如果parameter是数组且索引为`@`或`*`，则替换删除操作作用于每个数组元素。

例如：

```shell
$ file=linux.shell.jpg
$ echo ${file/l/L}    
Linux.shell.jpg
$ echo ${file//l/L}
Linux.sheLL.jpg
$ echo ${file/#l/L}
Linux.shell.jpg
$ echo ${file/%g/G}
linux.shell.jpG
$ echo ${file//l} 
inux.she.jpg
$ echo ${file/#linux.}
shell.jpg
```

## 变量、子元素内容大小写转换

```shell
${parameter^pattern}
${parameter^^pattern}
${parameter,pattern}
${parameter,,pattern}
```

将pattern匹配到的内容进行大小写转换。pattern只匹配单个字符。

- `^`或`^^`表示将小写转大写  
- `,`或`,,`表示大写转小写  
- `^`或`,`表示只转换第一个匹配的字符  
- `^^`或`,,`表示全局转换匹配的字符  
- 省略pattern时，等价于指定为`?`，表示对第一个字符(非全局)或每个字符(全局)进行转换  

如果是位置变量`@`或`*`，则对所有位置变量进行转换。

如果是数组，且索引为`@`或`*`，则对数组所有元素进行转换。

例如：

```shell
$ file=linux.shell.jpg
$ echo ${file^^}
LINUX.SHELL.JPG
$ echo ${file^} 
Linux.shell.jpg
$ echo ${file^^?}
LINUX.SHELL.JPG
$ echo ${file^^x}
linuX.shell.jpg
```

## 变量名匹配

```shell
${!prefix*}
${!prefix@}
```

匹配当前Shell环境下已定义的以prefix开头的变量名。

`@`符号表示各元素单独使用双引号包围，`*`符号表示使用一对双引号包围所有元素。

```shell
$ abc=hello
$ abcdef=HELLO
$ echo ${!abc*}
abc abcdef
$ echo ${!abc@}
abc abcdef
```

## 获取数组索引

```shell
${!name[@]}
${!name[*]}
```

当name是数组时，扩展为数组索引。当name不是数组时，如果name已定义(即使为空)，则扩展为0，如果未定义，则扩展未空。

所以，这个功能可以**判断是否是数组、是否已定义变量、是否未定义**。

```shell
# 普通数组时：
$ arr[1]=one
$ arr[2]=two
$ echo ${!arr[@]}
1 2

# 关联数组时
$ declare -A aarr
$ aarr['one']=1
$ aarr['two']=2
$ echo ${!aarr[@]}
one two

# 未定义变量时
$ echo ${!xyz[@]}

# 已定义变量且为空时
$ xyz=
$ echo ${!xyz[@]}
0
# 已定义变量且不为空时
$ xyz=hello
$ echo ${!xyz[@]}
0
```
