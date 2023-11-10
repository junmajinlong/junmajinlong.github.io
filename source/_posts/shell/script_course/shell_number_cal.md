---
title: Shell脚本深入教程：Bash数值运算
p: shell/script_course/shell_number_cal.md
date: 2020-05-19 14:17:06
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash数值运算

## Bash内置数值运算

使用`$(())`、`$[]`或内置命令`let`或`declare -i`(以及等价的typeset命令)可以进行数值运算，它们都**只能进行整数运算**。

- let命令只能单独作为一个命令行运算  
- `declare -i`声明整数类型的变量，声明后就能直接参与数值运算  
- `$(())`和`$[]`可以在命令内部进行运算，运算完成后会进行算术替换：即将运算结果替换到命令行中  

例如：

```shell
# 使用let
n=1
let n=n+1
echo $n      # 2
let n=$n+1
echo $n      # 3

# 使用declare -i
declare -i a b c
a=10
b=100
c=a+b
echo $c   # 110

# 使用$(())和$[]
x=10
y=20
echo $(( (x+y)*2 ))
echo $[ (x+y)*2 ]
```

## 小数运算

Bash自身语法不支持小数运算。但如果真的需要进行精确到小数的运算，是可以借助其它命令工具，比如bc、awk、perl等等。

```shell
# 使用awk
$ echo 1.2 2.3 | awk '{print $1+$2}'
3.5
$ awk 'BEGIN{print 1.2+2.3}'
3.5

# 使用bc
$ echo 1.2+2.3 | bc
3.5

# 使用perl
$ perl -E 'say 1.2+2.3'
3.5
$ echo 1.2 2.3 | perl -anE 'say $F[0]+$F[1]'
3.5
```