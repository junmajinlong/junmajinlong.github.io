---
title: Shell脚本深入教程：Bash支持的运算操作
p: shell/script_course/shell_op.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash支持的运算操作

```
id++ id-- ++id --id
自增、自减

- +
一元运算符，即正负号

! ~
逻辑取反、位取反

**
幂运算

* / %
乘法、除法、取模

+ -
加法、减法

<< >>
位左移、位右移

&
按位与运算

^
按位异或运算

|
按位或运算

<= >= < >
大小比较

== !=
等值比较

&&
逻辑与

||
逻辑或

expr ? expr : expr
三目条件运算

= *= /= %= += -= <<= >>= &= ^= |=
各种赋值语句

expr1 , expr2
多个表达式，例如$((x+=1,y+=3,z+=4))
```

几个注意事项：

1. 空变量或未定义变量或字符串变量参与数值运算时，都当作数值0  
2. 变量替换先于算术运算，所以既可以使用变量名`var`，也可使用变量引用`$var`，例如`$[a+1]`和`$[$a+1]`在结果上是等价的  
3. 算术表达式可以进行嵌套，先计算最内层。如`$(( (6+4)/$[2+3] ))`  
4. 0开头的表示八进制，0x或0X开头的表示十六进制，可使用`base#N`的方式指定N是多少进制的数。例如`echo $[ 010 + 10 ]`得到18、`echo $[ 0x10 + 10 ]`得到26，`echo $[ 2#100 + 10 ]`得到14。参见下面一个实际案例  
5. 由上面的运算符可看出，`$[]`、`$(())`以及let命令，既**支持算术运算，还支持赋值、大小比较、等值比较**。如`a=3;echo $[a==3]`  

有时候从字符串中截取得到的数值是以0开头的，如果要让它参与运算，需要指定为十进制，否则会当作八进制。例如：

```shell
datetime1=`date +"%s.%N"`
nano_seconds1=${datetime1#*.}
sleep 1
datetime2=`date +"%s.%N"`
nano_seconds2=${datetime2#*.}

echo $[ 10#$nano_seconds2 - 10#$nano_seconds1 ]
```