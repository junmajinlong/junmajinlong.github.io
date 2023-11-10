---
title: Perl的die和warn函数
p: perl/perl_die_warn.md
date: 2019-07-06 17:37:52
tags: Perl
categories: Perl
---

# die和warn

- die可以在出现错误的时候停止程序，并给出消息。默认会输出出错的程序名称和出错行号
- warn函数和die函数类似，但和die的区别是不会终止程序
- die和warn的参数末尾如果给了`\n`，将不会输出出错的程序名称和出错的程序行号
- `use autodie;`可以自动探测操作系统层面上的错误并停止程序

例如，下面打开文件的操作：
```
if ( ! open LOG "<" "/tmp/a.log" ){
    die "open file wrong: $!";
}
```
上面`$!`是收集操作系统报告的错误，并由perl报告出来。例如没有/tmp/a.log文件，上面的程序段落就会报错：
```
wrong open file: No such file or directory at 6.plx line 8.
```

其中的`$!`对应的消息是`No such file or directory`。


并不是每个错误都会有操作系统对应的错误。有些错误是perl自身的问题，这时候就不需要`$!`。
```
if ( @ARGV < 2 ){
    die "wrong! get help!"
}
```

默认情况下，die都会自动加上程序名和发生错误的行号。如果不想要这些消息，可以手动在die的末尾加上`\n`符号。
```
if ( @ARGV < 2 ){
    die "wrong! get help!\n"
}
```

使用autodie特性，可以
```
#!/usr/bin/perl

use autodie;
open LOG,"<","/tmp/a.log";
close LOG;
```
