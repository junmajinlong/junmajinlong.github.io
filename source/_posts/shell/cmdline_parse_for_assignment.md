---
title: Shell：变量赋值时的命令行解析
p: shell/cmdline_parse_for_assignment.md
date: 2020-10-30 00:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# 变量赋值时的命令行解析

## 问题描述

今天群里遇到了这么个bash的变量赋值问题：

```bash
$ c=1
$ tmp$c=$(echo abc)
tmp1=abc: command not found
```

初看之下，就是一个变量赋值的错误，但这个问题却一点都不简单，甚至难度系数很高。如果没有深入理解bash解析命令行的规则，基本不可能分析出报错结果为什么是`tmp1=abc: command not found`。

## 涉及到的理论

在bash中，虽然我们经常敲的是下面几种命令行格式：

```bash
var=value     # 变量赋值语句
CMD args      # 执行某个命令
>redirection  # 各种重定向方式
```

但实际上，上面三种命令行格式可以随意组合：

```bash
var=value CMD args
var=value >redirection  # 这种写法虽合理，但无意义
CMD args >redirection
```

甚至，对于变量赋值和重定向操作可以同时有多个：

```bash
var1=value1 var2=value2
var1=value1 var2=value2 CMD args

>redirection1 <redirection2
CMD args >redirection1 <redirection2
```

事实上，bash中一个简单命令(即不是命令组合或结构体)的完整格式是这样的：

```bash
var=value ...  CMD args  >redirection ...
```

也就是说，一个**完整的简单命令由三部分组成：变量赋值部分、命令和参数部分、重定向部分**。

其中：  
- 变量赋值和重定向部分可以同时有多个  
- 变量赋值必须在CMD的前面  
- 变量赋值部分的变量名必须符合bash要求的命名规范：  
  - 普通变量的变量名可以是字母、下划线和数字，但以字母或下划线开头  
  - 还有bash支持的一部分特殊变量名，比如`$$ $1 $0 $?`(实际上变量名不带`$`前缀，这里加上是为了更易懂)  
- 重定向部分可以在任意位置，比如`>redirection CMD args`也是合理的  
- 变量赋值和重定向操作从左向右执行  
- 这三部分可以随意省略，随意组合  

对于变量赋值部分：  
- 如果有变量赋值而没有命令部分，则变量赋值在当前shell下生效  
- 如果有变量赋值也有命令部分，则变量赋值只在该命令的环境变量中生效，不影响当前shell  
- 如果**变量名不符合bash的变量命名规范，则不认为是变量赋值语句，而是当作CMD命令部分**  

```bash
# 只有变量赋值部分，而没有命令部分，因此a变量在当前shell下生效
a=1
b=2 >/dev/null
echo $a $b   # 1 2

# 有变量赋值和命令部分，变量将作为该命令的环境变量，不影响当前shell
a=11 bash -c 'echo $a'  # 11
echo $a    # 空

# 下面的都不符合变量命名规范，都会当作命令部分执行
# 因此都会报错：command not found
tmp$=abc
tmp$c=abc
tmp+3=3
tmp+3=3 echo hhh  # echo hhh被当作命令tmp+3=3的参数
```

## 理论再进一步：shell如何解析一个完整的简单命令

前面解释过，一个简单命令的完整格式包括三部分：

```bash
var=value   CMD args   >redirection
```

无论它们如何省略或如何组合，shell都将它们作为一个包含三部分的完整简单命令来解析。

对于简单命令，解析规则和顺序如下：(已经明确说了是简单命令，因此在解析简单命令之前已经对命令行划分好了token和操作符)  

![](/img/shell/1603988594364.png)

## 问题最终的解答

有了前面的理论基础之后，再回到最初的问题：

```bash
$ c=1
$ tmp$c=$(echo abc)
tmp1=abc: command not found
```

现在分析这个问题已经很简单了。

对于`tmp$c=$(echo abc)`，它是一个简单命令，它看上去是一个变量赋值语句。

但实际上，在上文描述的第一步：标记变量赋值和重定向并将它们保留下来的步骤当中，就被发现了这不是一个变量赋值语句，因为变量名不符合命名规范，于是将它当作一个普通的命令而非变量赋值语句。

既然这个看似是变量赋值的部分被当作命令，那么在上文描述的第二步，开始对该命令部分执行各种Shell扩展，对于`tmp$c=$(echo abc)`这个命令来说，它涉及两部分扩展。首先变量扩展后得到`tmp1=$(echo abc)`，然后命令替换得到`tmp1=abc`。然后第二步完成。

因为这里没有变量赋值和重定向，所以没有第三步和第四步的过程。

于是到第五步，开始执行第二步完成之后得到的命令`tmp1=abc`，显然这个命令不存在，于是报错`tmp1=abc: command not found`。

## 参考阅读

全部在man bash中：  

- 变量命名规范：Definitions小节的name说明  
- 赋值语句：Shell Parameters小节  
- 简单命令的解析流程：Simple Command Expansion小节  