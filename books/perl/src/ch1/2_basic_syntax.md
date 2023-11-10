## Perl试手

在开始正式介绍Perl的内容之前，先对Perl最基本的一些用法混个眼熟。

- **Unix系统下，Perl脚本第一行使用`#!`。Perl脚本的后缀名一般为【.plx】或【.pl】，运行时使用`perl NAME.plx`即可**  

  例如，1.pl内容如下：

  ```perl
  #!/usr/bin/perl
  print "hello world\n"
  ```

  执行该脚本：

  ```bash
  $ perl 1.pl
  ```

  Windows系统下，不要加`#!`，因为Windows是通过关联打开`.pl`文件类型的应用程序来运行的。

- **Perl脚本中，除了注释行和代码块的最后一行，每行都需要以`;`结尾**  

- **Perl使用#作为注释符号，所以只支持单行注释、行尾注释**  

  ```perl
  # 这是单行注释
  print "hello world\n";  # 行尾注释
  ```

- **Perl中变量有三种数据类型：标量、数组、hash**  

  - 标量是存放单个数据的类型，标量使用`$`符号前缀来表示，如`$name`  

  - 数组是存放一系列数据的类型，数组使用`@`符号前缀，如`@names`  

  - hash是存放键值对(key-value)的数据类型，hash通常也称为映射、字典、关联数组，hash使用`%`符号前缀，如`%person`  

  - 其实上面关于前缀的说法是不准确的，但暂时这样理解，以免还未入门就放弃Perl  

  ```perl
  # 变量name是一个标量类型，只保存了一个字符串数据
  $name = "junmajinlong";
  
  # 变量language是一个数组类型，可保存多个数据
  @languages = ("Perl", "Ruby", "Shell", "Rust");
  
  # 变量person是一个hash类型，可保存key-value键值对数据
  %person = (name => "junmajinlong", age => 23,);
  ```

- **Perl常使用print()、say()、和printf()进行输出**  

  - print()输出时不加尾部换行符  
  - printf()用于格式化输出  
  - say()和print()类似，但输出时自动加尾部换行符，但使用say()时要求至少使用perl v5.10版本或开启say特性  

  ```perl
  print "hello world", "\n";  # 手动加上行尾换行符
  print "hello world\n";      # 效果同上
  
  printf "name: %s\n", "junmajinlong";
  
  use 5.010;    # 指定使用Perl v5.10版本
  say "hello world";  # 自动在行尾加上换行符
  ```

- **use关键字可用于指定使用哪个包、哪个特性、哪个版本的perl，等**  

  ```perl
  # 指定使用Time::HiRes包中的time函数
  use Time::Hires qw(time);
  
  # 指定使用say特性
  use features 'say';
  
  # 指定使用Perl 5.10版本
  use 5.010;
  ```

  注意，上面use指定版本的版本值是5.010而不是5.10，`use 5.10`会被perl认为是5.100版如果指定更细致的小版本号，如5.10.1版，则：`use 5.010001;`。

  也可以以如下方式指定版本号：

  ```perl
  use v5.10;
  ```

- **Perl中调用自带的内置函数时，可以使用括号传递参数，也可以省略括号。但省略括号时，有时候需要注意陷阱**  

  例如，调用print函数：
  ```perl
  print("hello world\n");
  print "hello world\n";
  ```

- **Perl中的双引号字符串内，可以使用变量替换、表达式替换。这种行为称为字符串的变量内插(interpolation)和表达式内插**  

  ```perl
  # 变量内插
  $name = "junmajinlong";
  print "name: $name\n";
  
  # 表达式内插，直接强记表达式内插语法：@{[EXPR]}
  print "10 + 10 = @{[10+10]}";
  ```

- **Perl中不需要对变量进行声明，可以直接赋值、使用**  

  ```perl
  # 全局变量
  $var=12;
  # 或者使用my、our等关键字定义有作用域的变量
  my $var = 23;
  print $var;
  ```

- **可以在每个Perl脚本中加上`use strict`语句，这是写稍大一点的Perl程序时的一种规范**  

  strict模式使得Perl编译器以严格的态度对待Perl程序，比如不允许使用未定义的变量、不允许直接定义全局变量。
  ```perl
  use strict;  # strict模式
  
  $var = "hello"; # 错，不允许直接定义全局变量
  print "$x";     # 错，不允许使用未定义的变量
  ```

  当指定使用的perl版本为`v5.12`或更高，则自动进入strict模式，因此可以省略`use strict;`。

- **可以加上warning信息进行调试，perl将在需要提示的地方发出警告**  

  ```perl
  use warnings;
  ```

  或者`perl -w`，或者在Perl脚本中：  
  ```perl
  #!/usr/bin/perl -w
  ```

- **如果Perl脚本中使用了中文(或其他多字节字符)，建议加上`use utf8;`，否则结果可能会偏离期待**  

- **为变量、函数取名时，应符合标识符规范：由大小写字母、下划线、数字组成，且不是数字开头**。但注意，Perl自身使用了大量特殊字符，有很多内置变量不符合标识符规范，例如`$!、@_`等变量  

- **Perl中可以通过反引号来执行操作系统中的命令**  

  ```perl
  $var=`date +"%F %T"`
  print $var
  ```


