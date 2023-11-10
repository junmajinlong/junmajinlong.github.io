---
title: Perl语法的基本规则
p: perl/perl_syntax.md
date: 2019-07-06 17:37:30
tags: Perl
categories: Perl
---

因为是比较凌乱的用法规则收集，所以能看懂则看，不能看懂也无所谓。以后也会遇到。

- **Perl脚本第一行使用`#!`。Perl的后缀名一般为".plx"或".pl"，运行时使用`perl NAME.plx`即可**

  例如，1.plx内容如下：
  ```
  #!/usr/bin/perl
  print "hello world\n"
  ```
  执行该脚本：
  ```
  shell> perl 1.plx
  ```

- **注释。Perl只支持"#"注释，所以只支持单行注释、行内到结尾注释**
  ```
  # comment
  print "hello world\n" # comment
  ```
- **Perl脚本中，除了最后一行，每行都需要以";"结尾，除非是注释行**

- **Perl中不需要对变量进行声明，可以直接赋值、引用**
  ```
  $var=12;
  print $var;
  ```

- **use指定使用某个版本的perl，如5.10版本。注意，use中是5.010而不是5.10，`use 5.10`会被perl认为是5.100版**
  ```
  use 5.010;
  ```

  如果指定更细致的小版本号，如5.10.1版，则：`use 5.010001;`。

- **最好都加上`use utf8`语句**
  ```
  use utf8;
  ```
- **最好在每个Perl程序中加上`use strict`语句，这在后面写稍大一点的Perl程序基本上是一种规范**

  该功能让Perl编译器以严格的态度对待Perl程序，如果定义了变量却未使用过，或者引用了未定义过的变量，都会编译错误。
  ```
  use strict;
  ```

- **可以加上warning信息进行调试**
  ```
  use warnings;
  ```

  或者`perl -w`，或者在Perl脚本中：
  ```
  #!/usr/bin/perl -w
  ```

- **Perl中可以通过反引号来执行操作系统中的命令**
  ```
  $var=`date +"%F %T"`
  print $var
  ```

- **Perl中调用自带的内置函数时，可以使用括号传递参数，也可以省略括号**

  例如，调用print函数：
  ```
  print("hello world\n");
  print "hello world\n";
  ```

- **Perl中的ENV：Perl可以通过ENV这个hash直接访问操作系统的环境变量**
  ```
  print $ENV{PATH};   # 输出操作系统的PATH环境变量
  ```
  如果Perl想访问操作系统中某个变量，可以直接在操作系统中设置，然后通过Perl访问：
  ```
  $ myvar=2;export myvar;

  print $ENV{myvar};
  ```

- **Perl中token之间如果是不同的命名类型，则中间的空格分隔符号可以省略**

  主要体现在函数和参数之间的空格。
  ```
  print"abc","def\n";   -> print "abc","def\n"
  print$var;      -> print $var
  my$var="abc";   -> my $var
  print~~length$var -> print length $var
  ```
  显然，参数部分的首字符如果是数值、下划线或字母，则会被当作函数名的一部分进行解析，这是错误的省略方式：
  ```
  print1+3;
  ```
