---
title: Perl的do语句块结构
p: perl/perl_do_block.md
date: 2019-07-06 17:37:51
tags: Perl
categories: Perl
---

# Perl的do语句块结构

do语句块结构如下：
```
do {...}
```

do语句块像是匿名子程序一样，没有名称，给定一个语句块，直接执行。且和子程序一样，do语句块的返回值都是最后一个执行的语句的返回值。

例如，将使用if-elsif-else结构进行赋值的行为改写成do。以下是if-elsif-else结构：
```
my $name;
if($gender eq "male"){
    $name="Malongshuai";
} elsif ($gender eq "female"){
    $name="Gaoxiaofang";
} else {
    $name="RenYao";
}
```

改写成do结构：
```
my $name=do{
    if($gender eq "male"){"Malongshuai"}
    elsif($gender eq "female") {"Gaoxiaofang"}
    else {"RenYao"}
};     # 注意结尾的分号
```

在perl中，使用表达式修饰符改写流程控制结构的时候，控制符左边只能写一个语句。例如下面的if，左边有了print后，就不能再有其它语句。
```
print "..." if(...);
```

使用do结构，可以将多个语句包围，然后执行：
```
#!/usr/bin/perl
use 5.010;

$a=3;
do {
    say "statement1";
    say "statement2";
} if $a > 2;
```

因为do有自己的代码块，有时候可以在这个代码块中使用自己的私有变量。

例如，读取一个文件，将文件中的内容赋值给一个变量。(涉及到后面的内容，看不懂请跳过)
```
my $file_content = do {
    local $/;
    local @ARGV = ("/root/a.txt");
    <>;
};
```

或者：
```
my $file_content = do {
    local $/;
    open my $fh,'<',"/root/a.txt" or die;
    <$fh>;
};
```