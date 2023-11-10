---
title: 精通awk系列(16)：gawk支持的正则表达式
p: shell/awk/awk_regexp.md
date: 2019-12-07 10:37:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# gawk支持的正则

```
.       # 匹配任意字符，包括换行符
^
$
[...]
[^...]
|
+
*
?
()
{m}
{m,}
{m,n}
{,n}

[:lower:]
[:upper:]
[:alpha:]
[:digit:]
[:alnum:]
[:xdigit:]
[:blank:]
[:space:]
[:punct:]
[:graph:]
[:print:]
[:cntrl:]

以下是gawk支持的：
\y    匹配单词左右边界部分的空字符位置 "hello world"
\B    和\y相反，匹配单词内部的空字符位置，例如"crate" ~ `/c\Brat\Be/`成功
\<    匹配单词左边界
\>    匹配单词右边界
\s    匹配空白字符
\S    匹配非空白字符
\w    匹配单词组成字符(大小写字母、数字、下划线)
\W    匹配非单词组成字符
\`    匹配字符串的绝对行首  "abc\ndef"
\'    匹配字符串的绝对行尾
```

gawk不支持正则修饰符，所以无法直接指定忽略大小写的匹配。

如果想要实现**忽略大小写匹配**，则可以将字符串先转换为大写、小写再进行匹配。或者设置预定义变量IGNORECASE为非0值。

```
# 转换为小写
awk 'tolower($0) ~ /bob/{print $0}' a.txt

# 设置IGNORECASE
awk '/BOB/{print $0}' IGNORECASE=1 a.txt
```

