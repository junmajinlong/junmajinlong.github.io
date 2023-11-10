---
title: Perl一行式：文本编解码、替换
p: perl/perl_oneliner_4.md
date: 2019-07-08 17:39:53
tags: Perl
categories: Perl
---

# Perl一行式：文本编解码、替换

**perl一行式程序系列文章**：[Perl一行式](/perl/index#blogperloneline)

-----------------------------

## 文本大小写转换

全部字符转换成大写或小写，有几种方式：
```
# 转大写
$ perl -nle 'print uc' file.log
$ perl -ple '$_ = uc' file.log
$ perl -nle 'print "\U$_"' file.log

# 转小写
$ perl -nle 'print lc' file.log
$ perl -ple '$_ = lc' file.log
$ perl -nle 'print "\L$_"' file.log
```

每行首字母大小写转换：
```
$ perl -nle 'print lcfirst' file.log
$ perl -lpe '$_ = ucfirst' file.log
$ perl -lne 'print \u\L$_' file.log
```

单词首字母大写，其它小写：
```
$ perl -ple 's/(\w+)/\u$1/g' file.log
```

## 修剪前缀、后缀空白

去掉前缀空白的方式：
```
$ perl -ple 's/^\s+//' file.log
```

去掉后缀空白的方式：
```
$ perl -lpe 's/\s+$//' file.log
```

同时去掉前缀和后缀空白：
```
$ perl -lpe 's/^\s+|\s+$//' file.log
```


## 反序输出所有段落

```
$ perl -00 -e 'print reverse <>' file.log
```

前面的文章[压缩连续的空行](/perl/perl_oneliner_1#blog1546583770)解释过，`-00`是按段落读取且压缩连续的空行。

`reverse <>`中reverse的操作对象期待的是一个列表，所以`<>`会一次性读取整个文件且按照段落读取，每个段落是列表中的一个元素。最后reverse函数反序这个列表，然后被print输出。


## 反序输出所有行

```
$ perl -e 'print reverse <ARGV>' file.log
sync   x 4 65534 sync   /bin      /bin/sync
sys    x 3     3 sys    /dev      /usr/sbin/nologin
bin    x 2     2 bin    /bin      /usr/sbin/nologin
daemon x 1     1 daemon /usr/sbin /usr/sbin/nologin
root   x 0     0 root   /root     /bin/bash
```

这里`reverse <ARGV>`表示一次性读取file.log的所有行并进行反转。

也可以使用下面这种方式，但如果文件结尾不正确(缺少eof)，可能会卡住：
```
$ perl -e 'print reverse <>' file.log
```

## ROT13字符映射

Perl中可使用`tr///`或`y///`进行字符一一映射的替换。它们和unix下的tr命令作用类似。

```
$ perl -le '$string="hello";$string =~ y/a-zA-Z/N-Za-mA-Mn-z/;print $string'
URYYb
```


## BASE64编码、解码

`MIME::Base64`模块提供了base64编码、解码的方法。

编码：
```
$ perl -MMIME::Base64 -e 'print encode_base64("coding")'
Y29kaW5n
```

解码：
```
$ perl -MMIME::Base64 -le 'print decode_base64("Y29kaW5n")'
coding
```

编码文件：
```
$ perl -MMIME::Base64 -0777 -ne '
    print encode_base64($_)' file.log
```

解码文件：
```
$ perl -MMIME::Base64 -0777 -ne 'print decode_base64($_)' file
```


## URL转义


使用`URI::Escape`模块即可进行URL转义。该模块需要额外安装`cpan URI::Escape`。
```
$ perl -MURI::Escape -le 'print uri_escape("http://example.com")'
http%3A%2F%2Fexample.com 
```

反转义：
```
$ perl -MURI::Escape -le '
    print uri_unescape("http%3A%2F%2Fexample.com")'
http://example.com
```


## HTML编码、解码

先安装额外HTML格式的编解码模块`cpan HTML::Entities`。
```
$ perl -MHTML::Entities -le 'print encode_entities("<html>")'
$ perl -MHTML::Entities -le 'print decode_entities("&lt;html&gt;")'
```
