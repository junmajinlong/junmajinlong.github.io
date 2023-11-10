---
title: Perl使用模块和@INC
p: perl/perl_module_inc.md
date: 2019-07-07 17:38:37
tags: Perl
categories: Perl
---

# Perl使用模块和@INC

## use加载模块

安装模块后，都会有对应的文档，可以通过`perldoc MODULE_NAME`来获取模块的使用帮助。

例如：获取`File::Utils`的使用帮助。
```
perldoc File::Utils
```

要在perl程序中使用模块，需要使用use来装载(load)模块。例如，`File::Basename`模块：
```
#!/usr/bin/perl

use File::Basename;
```
一般来说，所有要装载的模块都会写在perl程序的开头，因为use语句是程序编译期间执行的，而且以后要查看、修改程序中使用的模块也方便。当然，这并非必须，可以在任意地方书写use语句。

装载模块后，模块内的属性就会导入到当前程序的名称空间供当前程序使用。例如，`File::Basename`模块提供的两个子程序是basename()和dirname()，他们分别获取给定文件路径的basename和dirname。
```
#!/usr/bin/perl
use File::Basename;

my $basename = basename( "/usr/bin/perl" );
my $dirname = dirname( "/usr/bin/" );
print "$basename\n$dirname";
```

或者，只导入给定的函数：
```
use File::Basename ('basename','dirname');
```
推荐直接用qw()列表的方式，即使只导入一个函数：
```
use File::Basename qw(basename dirname);
```

函数导入到当前名称空间后，模块中的函数可能会和当前程序文件中定义的同名子程序冲突。这时需要指定函数的全名：
```
my $dirname = File::Basename::dirname( "/usr/bin/" );
```

如果，导入模块时给出一个空列表，它和不给列表是不一样的。也就是说，下面两种方式是不等价的：
```
use File::Basename;
use File::Basename ();
```

前者表示导入模块的默认属性，包括一些默认函数。而后者表示什么都不导入，这时如果要使用这个模块中的函数，只能使用全名。

另外，有些模块比较复杂，函数、属性、方法比较多，一般这时候会提供标签分组功能，其实它们是一堆函数集合。这样可以在导入的时候按照分组标签来导入。常见的一个标签是`:all`，它表示导入所有属性。
```
use CGI qw(:all);
```

如果模块还提供了一个名为tag的标签，那么可以导入这个标签：
```
use MODULE_NAME qw(:tag);
```

想要知道模块是否提供了标签，以及提供了哪些标签，可以`man MODULE`获取。例如，CGI模块的`man CGI`文档中有一段内容如下：
```
Here is a list of the function sets you can import:

    :cgi
       Import all CGI-handling methods, such as param(), path_info() and the like.

    :form
       Import all fill-out form generating methods, such as textfield().

    :html2
       Import all methods that generate HTML 2.0 standard elements.

    :html3
       Import all methods that generate HTML 3.0 elements (such as <table>, <super> and <sub>).

    :html4
       Import all methods that generate HTML 4 elements (such as <abbrev>, <acronym> and <thead>).

    :netscape
       Import the <blink>, <fontsize> and <center> tags.

    :html
       Import all HTML-generating shortcuts (i.e. 'html2', 'html3', 'html4' and 'netscape')

    :standard
       Import "standard" features, 'html2', 'html3', 'html4', 'form' and 'cgi'.

    :all
        Import all the available methods.  
```


## 模块查找路径和@INC

默认情况下，perl将从`@INC`指定的路径中查找模块，它就像shell的PATH环境变量一样。

以下是`@INC`的路径：
```
[root@xuexi ~]# perl -e 'foreach (@INC){print "$_\n"};'
/etc/perl
/usr/local/lib/x86_64-linux-gnu/perl/5.26.1
/usr/local/share/perl/5.26.1
/usr/lib/x86_64-linux-gnu/perl5/5.26
/usr/share/perl5
/usr/lib/x86_64-linux-gnu/perl/5.26
/usr/share/perl/5.26
/usr/local/lib/site_perl
/usr/lib/x86_64-linux-gnu/perl-base
```

如果我们手动安装的包，或者安装到了一个非默认的查找路径下(例如不同用户安装到了不同家目录下)，这时可以通过在shell中设置`PERL5LIB`环境变量，perl会临时从这个环境变量中去查找模块。
```
[root@xuexi ~]# export PERL5LIB="/home/fairy/myperl"
```

或者，在perl命令行中使用`-I`选项，显式指定待运行程序的模块查找路径。
```
[root@xuexi ~]# perl -I/home/fairy/myperl PERL_PROGRAM
```

还有几种更复杂的方法：  
- (1).在perl程序中，在use引用模块之前，使用`BEGIN {unshift @INC,"/home/fairy/myperl"};`语句，使得@INC在编译期间就加上指定的查找目录  
- (2).在perl程序中，在use引用模块之前，使用`use lib "/home/fairy/myperl";`指定lib查找路径  

由于这些方法都不方便，所以，直接设置PERL5LIB环境变量，或者设置`local::lib`即可。
