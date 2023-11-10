---
title: expr命令全解
p: shell/expr.md
date: 2023-07-08 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# expr命令全解

expr命令可以实现数值运算、数值或字符串比较、字符串匹配、字符串提取、字符串长度计算等功能。它还具有几个特殊功能，判断变量或参数是否为整数、是否为空、是否为0等。

## expr中文手册(info expr)

先看expr命令的info文档 `info expr` 的翻译。

```
16.4.1 字符串表达式
-------------------------
'expr'支持模式匹配和字符串操作。字符串表达式的优先级高于数值表达式和逻辑关系表达式。
 
'STRING : REGEX'
     执行模式匹配。两端参数会转换为字符格式，且第二个参数被视为正则表达式(GNU基本正则)，它默认会隐含前缀"^"。随后将第一个参数和正则模式做匹配。
 
     如果匹配成功，且REGEX使用了'\('和'\)'，则此表达式返回匹配到的，如果未使用'\('和'\)'，则返回匹配的字符数。
 
     如果匹配失败，如果REGEX中使用了'\('和'\)'，则此表达式返回空字符串，否则返回为0。
 
     只有第一个'\(...\)'会引用返回的值；其余的'\(...\)'只在正则表达式分组时有意义。
 
     在正则表达式中，'\+'，'\?'和'\|'分表代表匹配一个或多个，0个或1个以及两端任选其一的意思。
 
'match STRING REGEX'
     等价于'STRING : REGEX'。
 
'substr STRING POSITION LENGTH'
     返回STRING字符串中从POSITION开始，长度最大为LENGTH的子串。如果POSITION或LENGTH为负数，0或非数值，则返回空字符串。
 
'index STRING CHARSET'
     CHARSET中任意单个字符在STRING中最前面的字符位置。如果在STRING中完全不存在CHARSET中的字符，则返回0。见后文示例。
    
'length STRING'
     返回STRING的字符长度。
 
'+ TOKEN'
     将TOKEN解析为普通字符串，即使TOKEN是像MATCH或操作符"/"一样的关键字。这使得'expr length + "$x"'或'expr + "$x" : '.*/\(.\)''可以正常被测试，即使"$x"的值可能是'/'或'index'关键字。这个操作符是一个GUN扩展。
     通用可移植版的应该使用'" $token" : ' \(.*\)''来代替'+ "$token"'。
 
   要让expr将关键字解析为普通的字符，必须使用引号包围。
 
16.4.2 算术表达式
--------------------------
 
'expr'支持普通的算术操作，算术表达式优先级低于字符串表达式，高于逻辑关系表达式。
 
'+ -'
     加减运算。两端参数会转换为整数，如果转换失败则报错。
 
'* / %'
     乘，除，取模运算。两端参数会转换为整数，如果转换失败则报错。
 
16.4.3 逻辑关系表达式
---------------------------
 
'expr'支持普通的逻辑连接和逻辑关系。它的优先级最低。
 
'|'
     如果第一个参数非空且非0，则返回第一个参数的值，否则返回第二个参数的值，但要求第二个参数的值也是非空或非0，否则返回0。如果第一个参数是非空或非0时，不会计算第二个参数。
    
     经过测试，以上手册内容是错误的。正确的应该是：如果第一个参数非0，则返回第一个参数的值，否则返回第二个参数。但如果任意一个参数为空，则报错。除非空字符串使用引号包围，此时将和0的处理方式一样。
 
'&'
     如果两个参数都非空且非0，则返回第一个参数，否则返回0。如果第一个参为0或为空，则不会计算第二个参数。
    
     经过测试，以上手册内容是错误的。正确的应该是：如果两个参数都非0，则返回第一个参数，否则返回0。但任意一个参数为空，则报错。除非空字符串使用引号包围，此时将和0的处理方式一样。
 
'< <= = == != >= >'
     比较两端的参数，如果为true，则返回1，否则返回0。"=="是"="的同义词。"expr"首先尝试将两端参数转换为整数，并做算术比较，如果转换失败，则按字符集排序规则做字符比较。
 
括号'()'可以改变优先级，但使用时需要使用反斜线对括号进行转义。
 
16.4.4 'expr'使用示例
-------------------------------
 
以下为expr的一些示例，其中有将shell的元字符使用引号包围的示例。
 
   将shell中变量'foo'的值增加1：
 
     foo=$(expr $foo + 1)
 
   输出变量路径变量'$fname'中不包含'/'的文件名部分：
 
     expr $fname : '.*/\(.*\)' '|' $fname
    
     解释：其中的'|'是expr中的连接符，只不过是被引号包围防止被shell解析。例如$fname=/etc/hosts，则此表达式返回hosts，如果$fname=/usr/share/，则此表达式'|'的左边为空，所以返回'|'右边的值，即$fname，即返回/usr/share/。
 
   An example showing that '\+' is an operator:
 
     expr aaa : 'a\+'    # 解释：因为REGEX部分没有使用\(\)，所以返回匹配的字符数
     => 3
 
     expr abc : 'a\(.\)c'  # 解释：因为REGEX部分使用了\(\)，所以返回匹配的字符
     => b
     expr index abcdef cz
     => 3
     expr index index a    # 解释：因为第二个index是关键字
     error-> expr: syntax error
     expr index + index a  # 解释：使用+将index关键字解析为普通字符串
     => 0
```

## expr使用示例

下面将使用示例来介绍expr的用法，在介绍之前，需要注意三点：

- (1).**数值表达式(`+ - \* / %`)和比较表达式(`< <= = == != >= >`)会先将两端的参数转换为数值，转换失败将报错。所以可借此来判断参数或变量是否为整数。**
- (2).expr中的很多符号需要转义或使用引号包围。
- (3).所有操作符的两边，都需要有空格。

以下是expr示例。

**(1).`string : REGEX`字符串匹配示例。要输出匹配到的字符串结果，需要使用`\(`和`\)`，否则返回的将是匹配到的字符串数量。**

```bash
[root@xuexi ~]# expr abcde : 'ab\(.*\)'
cde

[root@xuexi ~]# expr abcde : 'ab\(.\)'
c

[root@xuexi ~]# expr abcde : 'ab.*'  
5

[root@xuexi ~]# expr abcde : 'ab.'   
3

[root@xuexi ~]# expr abcde : '.*cd*'
4
```

注意，由于REGEX中隐含了`^`，所以使得匹配时都是从string首字符开始的。

```bash
[root@xuexi ~]# expr abcde : 'cd.*'  
0
```

之所以为0，是因为真正的正则表达式是`^cd.*`，而abcde不是c开头而是a开头的，所以无法匹配到任何结果。因此，任何字符串匹配时，都应该从首字符开始。

字符串匹配时，会先将两端参数转换为字符格式。

**(2).`index string chars`用法示例。**

该表达式是从string中搜索chars中某个字符的位置，这个字符是string中最靠前的字符。例如：

```bash
[root@xuexi ~]# expr index abcde dec
3
```

该命令将对字符串"dec"逐字符分解，首先分解得到第一个字符d，从abcde中搜索到d的位置为4，再分解得到第二个字符e，该字符在abcde中的位置为5，最后得到的字符是c，该字符在abcde中的位置为3。其中3是最靠前的字符，所以命令返回的结果为3。

```bash
[root@xuexi ~]# expr index abcde xdc
3
```

如果chars中的所有字符都不存在于string中，则返回0。

```bash
[root@xuexi ~]# expr index abcde 1
0

[root@xuexi ~]# expr index abcde 1x
0
```

**(3).`substr string pos len`用法示例。**

该表达式是从string中取出从pos位置开始长度为len的子字符串。如果pos或len为非正整数时，将返回空字符串。

```bash
[root@xuexi ~]# expr substr abcde 2 3
bcd

[root@xuexi ~]# expr substr abcde 2 4
bcde

[root@xuexi ~]# expr substr abcde 2 5
bcde

[root@xuexi ~]# expr substr abcde 2 0

[root@xuexi ~]# expr substr abcde 2 -1
```

**(4).`length string`用法示例。该表达式是返回string的长度，其中string不允许为空，否则将报错，所以可以用来判断变量是否为空。**

```bash
[root@xuexi ~]# expr length abcde
5

[root@xuexi ~]# expr length 111
3

[root@xuexi ~]# expr length $xxx
expr: syntax error

[root@xuexi ~]# if [ $? -ne 0 ];then echo '$xxx is null';fi
$xxx is null
```

**(5).`+ token`用法示例。**

expr中有些符号和关键字有特殊意义，如`match、index、length`，如果要让其成为字符，使用该表达式将任意token强制解析为普通字符串。

```bash
[root@xuexi ~]# expr index index d
expr: syntax error

[root@xuexi ~]# expr index length g
expr: syntax error

[root@xuexi ~]# expr index + length g
4
```

对值为关键字的变量来说，则无所谓。

```bash
[root@xuexi ~]# len=lenght

[root@xuexi ~]# expr index $len g
4
```

**(6).算术运算用法示例。**

```bash
[root@xuexi ~]# expr 1 + 2
3

[root@xuexi ~]# a=3
[root@xuexi ~]# b=4

[root@xuexi ~]# expr $a + $b
7

[root@xuexi ~]# expr 4 + $a
7

[root@xuexi ~]# expr $a - $b
-1
```

算术乘法符号`*`因为是shell的元字符，所以要转义，可以使用引号包围，或者使用反斜线。

```bash
[root@xuexi ~]# expr $a * $b
expr: syntax error

[root@xuexi ~]# expr $a '*' $b
12

[root@xuexi ~]# expr $a \* $b
12

# 除法只能取整数
[root@xuexi ~]# expr $b / $a
1

[root@xuexi ~]# expr $b % $a
1
```

任意操作符两端都需要有空格，否则：

```bash
[root@xuexi ~]# expr 4+$a 
4+3

[root@xuexi ~]# expr 4 +$a
expr: syntax error
```

由于expr在进行算术运算时，首先会将操作符两边的参数转换为整数，任意一端转换失败都将会报错，所以可以用来判断参数或变量是否为整数。

```bash
[root@xuexi ~]# expr $a + $c
expr: non-integer argument

[root@xuexi ~]# if [ $? != 0 ];then echo '$a or $c is non-integer';fi          
$a or $c is non-integer
```

**(7).比较操作符`< <= = == != >= >`用法示例。其中`<`和`>`是正则表达式正的锚定元字符，且`<`会被shell解析为重定向符号，所以需要转义或用引号包围。**

这些操作符会首先会将两端的参数转换为数值，如果转换成功，则采用数值比较，如果转换失败，则按照字符集的排序规则进行字符大小比较。比较的结果若为true，则expr返回1，否则返回0。

```bash
[root@xuexi ~]# a=3

[root@xuexi ~]# expr $a = 1
0

[root@xuexi ~]# expr $a = 3
1

[root@xuexi ~]# expr $a \* 3 = 9
1

[root@xuexi ~]# expr abc \> ab
1

[root@xuexi ~]# expr akc \> ackd
1
```

**(8).逻辑连接符号`&`和`|`用法示例。这两个符号都需要转义，或使用引号包围。**

以下是官方文档中给出的解释，但实际使用过程中是不完全正确的。

- `&`表示如果两个参数同时满足非空且非0，则返回第一个参数的值，否则返回0。且如果发现第一个参数为空或0，则直接跳过第二个参数不做任何计算。
- `|`表示如果第一个参数非空且非0，则返回第一个参数值，否则返回第二个参数值，但如果第二个参数为空或为0，则返回0。且如果发现第一个参数非空或非0，也将直接跳过第二个参数不做任何计算。

正确的应该是：

- `&`表示如果两个参数都非0，则返回第一个参数，否则返回0。但任意一个参数为空，则expr报错。除非空字符串使用引号包围，则处理方法和0一样。
- `|`表示如果第一个参数非0，则返回第一个参数的值，否则返回第二个参数。但如果任意一个参数为空，则expr报错。除非空字符串使用引号包围，则处理方法和0一样。

```bash
[root@xuexi ~]# expr $abc '|' 1
expr: syntax error

[root@xuexi ~]# expr "$abc" '|' 1
1

[root@xuexi ~]# expr "$abc" '&' 1 
0

[root@xuexi ~]# expr $abc '&' 1 
expr: syntax error

[root@xuexi ~]# expr 0 '&' abc
0

[root@xuexi ~]# expr abc '&' 0
0

[root@xuexi ~]# expr abc '|' 0
abc

[root@xuexi ~]# expr 0 '|' abc  
abc

[root@xuexi ~]# expr abc '&' cde
abc

[root@xuexi ~]# expr abc '|' cde
abc
```