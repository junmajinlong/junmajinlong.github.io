---
title: Perl的命令行参数和ARGV
p: perl/perl_cmd_args_argv.md
date: 2019-07-06 17:37:53
tags: Perl
categories: Perl
---

# Perl的命令行参数和ARGV

## Perl程序名：$0

`$0`表示当前**正在运行的Perl脚本名**。有3种情况：
1. 如果执行方式为`perl x.pl`，则`$0`的值为`x.pl`而非perl命令本身  
2. 如果执行方式为`./x.pl`，则`$0`的值为`./x.pl`  
3. 如果执行的是`perl -e`或`perl -E`一行式perl程序，则`$0`的值为`-e`或`-E`  

## Perl命令行参数ARGV

- perl将perl命令行的参数列表放进数组ARGV(@ARGV)中。既然是数组，就可以访问($ARGV[n])、遍历，甚至修改数组元素  
- ARGV数组分三种情况收集：  
    - `perl x.pl a b c`方式运行时，脚本名x.pl之后的`a b c`才会被收集到ARGV数组  
    - `./x.pl a b c`方式运行时，`a b c`才会被收集到ARGV数组  
    - `perl -e 'xxxxx' a b c`方式运行时，`a b c`才会被收集到ARGV数组  
- ARGV数组索引从0开始计算，索引0位从脚本名(perl程序名)之后的参数开始计算  
- 默认，这些命令行参数是perl程序的数据输入源，也就是perl会依次将它们当作文件进行读取  
- 参数是有序的，读取的时候也是有序的  
- 需要区分ARGV变量和ARGV数组：  
    - `$ARGV`表示命令行参数代表的文件列表中，当前被处理的文件名  
    - `@ARGV`表示命令行参数数组  
    - `$ARGV[n]`：表示命令行参数数组的元素  
    - `ARGV`：表示`<>`当前正在处理的文件句柄  

例如，test.plx的内容如下：
```
/usr/bin/perl

print '$ARGV[0] ---> ',$ARGV[0],"\n",
      '$ARGV[1] ---> ',$ARGV[1],"\n",
      '$ARGV[2] ---> ',$ARGV[2],"\n",
      '$ARGV[3] ---> ',$ARGV[3],"\n",
      '$ARGV[4] ---> ',$ARGV[4],"\n";
```
执行这个程序：
```
shell> ./test.plx -w a b c d
$ARGV[0] ---> -w
$ARGV[1] ---> a
$ARGV[2] ---> b
$ARGV[3] ---> c
$ARGV[4] ---> d
```

因为是数组，所以可以修改数组，比如强制指定元素：
```
/usr/bin/perl

@ARGV=qw(first second third);
print '$ARGV[0] ---> ',$ARGV[0],"\n",
      '$ARGV[1] ---> ',$ARGV[1],"\n",
      '$ARGV[2] ---> ',$ARGV[2],"\n";
```
```
shell> ./test.plx a b c d
$ARGV[0] ---> first
$ARGV[1] ---> second
$ARGV[2] ---> third
```

例如，读取2个文件(a.log,b.log)的内容：
```
/usr/bin/perl

while(<>){
    print $_;
}
```
```
shell> ./test.plx a.log b.log
```

如果想读取标准输入，只需使用"-"作为文件参数即可。
```
$ echo -e "abcd\nefg" | ./test.plx a.log - b.log
```

上面将按先后顺序读取a.log，标准输入(管道左边命令的输出内容)，b.log。

