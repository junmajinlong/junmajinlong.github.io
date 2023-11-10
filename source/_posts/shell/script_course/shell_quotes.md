---
title: Shell脚本深入教程：理解Shell的单双引号
p: shell/script_course/shell_quotes.md
date: 2023-07-10 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------


# 理解Shell的单双引号

Shell的引号是一个重要知识点，也是Shell最难的知识点。虽然Bash的引号解析规则很简单，但是想要一次性写出正确的复杂的命令行是很难的，如果不理解引用规则，调试引号问题会花费很长时间。

对于Shell中引号的使用，有如下结论：

1. **引号要配对**，否则在词法分析的划分token阶段会一直等待用户提供更多引号来完成命令行  
2. 单引号和双引号以及反斜线的效果：
- (1).反斜线：使反斜线后一个字符变为普通的字面字符 
- (2).双引号：双引号内所有字符变为字面符号，但`` \、$、`(反引号) ``除外，如果开启了`!`引用历史命令时，则感叹号也除外。所以：  
  - 双引号内可用反斜线转义双引号本身，使得双引号变成普通字符，例如`echo "hel\"lo"`
  - 双引号内可以执行命令替换、变量替换、反斜线转义、算术运算等，但不能执行大括号扩展、波浪号扩展、通配符路径扩展等  
  - 成对的双引号内出现命令替换时，命令替换是先执行的，因此命令替换部分单独解析引号，也因此命令替换内部的引号不和外部引号进行从左向右的配对  
- (3).单引号：单引号内的所有字符全部变为字面符号符号。但注意：单引号内不能再使用单引号，即使使用了反斜线转义也不允许  
  - 所以，单引号内所有Shell扩展和替换都不执行  
3. **每一个命令替换都开启一个新的独立的命令行解析上下文，包括对引号的解析**。也就是说，命令替换中的引号和命令替换外部的引号互不影响。并且，命令替换中嵌套的命令替换，命令行解析上下文也是独立的。

例如：

```shell
# 输出单引号
echo "hello'world"
```

在Shell开始词法分析时，首先读取到双引号，于是会继续向后读取，直到遇到另一个配对的双引号作为引用结束，其中中间会读到单引号，它是在双引号范围内的，所以它不需要配对。

其它示例：

```shell
# 输出双引号
echo 'hello"world'
echo "hello\"world"

# 单双引号混用：awk单引号隔开第一、第二字段
echo hello world | awk '{print $1"'"'"'"$2}'
echo hello world | awk "{print \$1\"'\"\$2}"
echo hello world | awk '{print $1"\047"$2}'

# sed或awk使用Shell中的变量
line=`cat /etc/fstab | wc -l`
sed -n $$((line-2))'p' /etc/fstab 
sed -n "$((line-2))p" /etc/fstab

# sed中使用命令替换
sed -n 's/'$(hostname -I)'/0.0.0.0/' a.txt
sed -n "s/$(hostname -I)/0.0.0.0/" a.txt
```

<a name="tag1"></a>

## `$ ''`和`$ ""`

有些时候命令行中是加单引号还是双引号是非常难处理的，如果在考虑了单双引号之后还是难以为继，不妨考虑以下使用`$''`，当然还有一个`$""`。

(1).如果没有特殊定制bash环境或有特殊需求，`$"string"`和`"string"`是完全等价的，使用`$""`只是为了保证本地化。

以下是`man bash`关于`$""`的解释：
```
A double-quoted string preceded by a dollar
sign ($"string") will cause the string to be
translated according to the current locale. 
If the current locale is C or POSIX, the
dollar sign is ignored. If the string is 
translated and replaced, the replacement is 
double-quoted.
```
(2).还有`$`后接单引号的`$'string'`，这在bash中被特殊对待：会将某些反斜线序列(如`\n，\t，\"，\'`等)继续转义，而不认为它是字面符号(如果没有`$`符号，单引号会强制将string翻译为字面符号，包括反斜线)。简单的例子：
```bash
[root@xuexi ~]# echo 'a\nb'
a\nb
[root@xuexi ~]# echo $'a\nb'
a
b
```
以下是`man bash`里关于`$'`的说明：
```
Words of the form $'string' are treated specially. 
The word expands to string, with backslash-escaped 
characters replaced as specified by the ANSI C standard. 
Backslash escape sequences, if present, are decoded as follows:
     \a     alert (bell)
     \b     backspace
     \e
     \E     an escape character
     \f     form feed
     \n     new line
     \r     carriage return
     \t     horizontal tab
     \v     vertical tab
     \\     backslash
     \'     single quote
     \"     double quote
     \nnn   the eight-bit character whose value is the 
            ctal value nnn (one to three digits)
     \xHH   the eight-bit character whose value is the 
            hexadecimal value HH (one or two hex digits)
     \uHHHH the Unicode (ISO/IEC 10646) character whose 
            value is the hexadecimal value HHHH (one to 
            four hex digits)
     \UHHHHHHHH  the Unicode (ISO/IEC 10646) character 
                 whose value is the hexadecimal value 
                 HHHHHHHH (one to eight hex digits)
     \cx    a control-x character
```
