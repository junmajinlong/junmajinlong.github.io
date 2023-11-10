---
title: Perl的输出：print、say和printf
p: perl/perl_output.md
date: 2019-07-06 17:37:49
tags: Perl
categories: Perl
---

# Perl的输出：print、say和printf

print、printf和say都可以输出信息。print和say类似，print不自带换行符，say自带换行符，但要使用say，必须写use语句`use 5.010;`，printf像C语言的printf一样，可以定制输出格式，不过我这perl似乎不支持printf，一用就报错，不知道为什么。它们有返回值：如果输出成功，就返回1。

注意perl中有上下文的概念，这几个输出操作也同样有上下文环境：列表上下文。
```
@arr=qw(hello world);
print "hello world","\n"; 
print "hello world\n";  
print @arr;            # 输出helloworld(没空格)
print "@arr";          # 输出hello world(有空格)
```


```
use 5.010;
say "hello world!";  # 自带换行符
```

这些本没有什么可解释的，但是print/say可以以函数格式(`print(args)`/`say(args)`)进行输出，这时候有个陷阱需要注意。
```
print(3+4)*4;
```
这个返回7，而不是28。这是怎么计算的？

Perl中很多时候是可以省略括号的，这往往让我们忘记括号的归属。而Perl中又有上下文的概念，在不同上下文执行同一个操作的结果是不一样的。在这里：
- print不加括号的时候，它需求的参数是一个列表上下文，它后面的所有内容都会被print输出
- print加括号的时候，它只会输出括号中的内容

所以，上面的语句等价于：
```
(print(3+4))*4
```
它先执行`print(7)`，然后拿到print的返回值1，将其乘以4，由于没有赋值给其它变量，所以这个乘法的结果被丢弃。

如果将上面赋值给一个变量：
```
$num = print(3+4)*4;
```
则`$num`的值将为4。


如果想要输出`(3+4)*4=28`的结果，可以将它们放在一个括号里，或者在`(3+4)`的括号前加一个`+`号，它表示让它后面的表达式作为函数的参数，相当于加个括号。所以，下面两个是等价的语句：
```
print ((3+4)*4);
print +(3+4)*4;
```


另外，由于print/say不使用括号的时候，它们会输出其后面的列表。所以有以下技巧：
- 像cat命令一样，直接输出文件内容：`print <>;`
- 像sort命令一样，排序文件内容：`print sort <>;`

