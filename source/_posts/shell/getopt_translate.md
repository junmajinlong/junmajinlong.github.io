---
title: man getopt中文手册(翻译)
p: shell/getopt_translate.md
date: 2020-02-12 17:36:29
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# getopt命令中文手册(man getopt翻译)

```
## NAME

       getopt - 解析命令行选项(加强版)

## SYNOPSIS

       getopt optstring parameters
       getopt [options] [--] optstring parameters
       getopt [options] -o|--options optstring [options] [--] parameters

(译注：  
1. 后面的译文中将分别称呼这3种语法格式为语法1、语法2、语法3  
2. 请区分option、parameter、argument、option argument、non-option parameter。如不清楚，请参考：https://www.junmajinlong.com/shell/getopt  
)

## DESCRIPTION

getopt用于拆分(解析)命令行中的选项，以便能被shell程`如shell脚本)轻松解析，也用来检查选项是否合理。该命令使用的是GNU getopt(3)程序实现的。

getopt的参数分为两部分：用于修改getopt解析模式的选项(即语法中的options和-o|--options optstring)和待解析的参数(即语法中的parameters部分)。第二部分将从第一个非选项参数开始，或者从"--"之后的第一项内容开始。如果在第一部分中没有给定"-o|--options"，则第二部分的第一个参数将用作短选项字符串。

如果设置了环境变量GETOPT_COMPATIBLE，或者它的第一个参数不是一个选项(即未使用"-"开头的字符串，即语法1)，getopt将生成与其它getopt(1)兼容的输出。它仍然会对参数进行调整，也会识别可选参数(见下面的COMPATIBILITY部分)。

传统的getopt(1)无法拷贝空白字符以及其它特殊字符(特指shell的特殊字符)。要解决该问题，getopt可以生成引号保护的输出，然后再用shell去解释(通常使用eval命令)，这样可以保护这些特殊字符，但前提是不能使用兼容版本的getopt(即语法2和语法3可以解决该问题)。要判断所安装的getopt是否是加强版的getopt，只需使用特殊选项"-T"进行测试即可。


## OPTIONS

`-a, --alternative`  
允许长选项(long options)用单个短横线"-"开头。(即单个短横线开头的选项也会被认为是长选项)

`-h, --help`  
输出getopt的用法帮助信息。

`-l, --longoptions longopts`  
识别长选项(多个字符的选项)。longopts可以一次性指定多个选项名称，只需使用逗号分隔即可。该选项可以多次使用，longopts会进行累积。在longopts中，每个长选项名称后面都可以跟一个冒号，表示该选项需要一个参数，也可以跟两个冒号，表示该选项的参数是可选的。

`-n, --name progname`  
当getopt(3)报告错误时，getopt(3)将使用该名称。注意，getopt(1)的错误仍然被报告为来自getopt。

`-o, --options shortopts`  
识别短选项(单个字符的选项)。如果没有给定该选项，getopt的第一个未使用"-"开头的参数(且不是option argument)将被用于短选项字符串。shortopts中的每个短选项字符后面都可以跟一个冒号，表示该选项需要一个参数，也可以跟两个冒号，表示该选项的参数是可以选的。shortopts的第一个字符可以设置"+"或"-"来影响选项的解析方式和输出方式(详细内容见下面的SCANNING MODES)。

`-q, --quiet`  
禁止getopt(3)输出的报错信息。

`-Q, --quiet-output`  
禁止普通的输出。但仍然会输出getopt(3)产生的错误信息，除非同时指定了"-q"选项。

`-s, --shell shell`  
指定使用哪种shell的引号解析方式。如果未使用-s选项，则使用BASH。其它支持的shell类型包括：sh、bash、csh和tcsh。

`-u, --unquoted`
不要使用引号包围输出。注意，禁止引号保护后，空白字符和shell特殊字符将被破坏(就像其它版本的getopt(1)一样)。

`-T, --test`  
测试使用的getopt(1)是加强版的还是传统旧版的。加强版的getopt没有任何输出，且退出状态码为4。其它版本的getopt(1)或者加强版的getopt设置了GETOPT_COMPATIBLE环境变量，则输出"--"，且退出状态码为0。

`-V, --version`  
输出getopt的版本号并退出。

## PARSING

本节指定getopt参数的第二部分的格式。下一节(OUTPUT)描述生成的输出。这些参数通常是shell函数调用的参数。必须注意的是，调用shell函数时的每个参数都恰好对应getopt参数列表中的一个参数(请参阅EXAMPLES)。所有的解析行为都由GNU getopt(3)完成。

参数解析的方式是从左向右解析。每个参数都被分类为短选项、长选项、选项参数(option argument)和非选项类型的参数(non-option parameter)其中的一种。

一个简单的短选项是使用"-"跟一个短选项字符。如果选项需要一个参数，这个选项参数可以直接写在选项字符的后面(译注：例如`-n3`)，或者作为下一个parameter(例如，在命令中使用空白分隔(译注：例如`-n 3`))。如果选项的参数是可选的，当需要解析给定的选项参数时，参数必须直接写在选项字符后面(译注：即不能用空白分隔选项和它的参数)。

可以在单个短横线后面跟多个短选项字符(译注：例如`tar -zcf`的zcf都在单个"-"后面)，只要它们不需要参数(允许最后一个给参数)即可(译注，参考`tar -zcf a.tar.gz`的格式)。

长选项一般使用"--"开头，后面跟长选项的名称。如果长选项需要参数，可以使用"="连接长选项和它的参数，或者将参数作为长选项的下一个parameter(译注，例如两种形式：`--file=FILE`或`--file FILE`)。如果长选项的参数是可选的，必须使用"="连接长选项和它的参数值。长选项可以进行缩写，只要不会产生歧义即可。

如果一个parameter不是使用"-"开头，也不是它前面的选项所需的参数，那么它就是一个非选项类型的参数(non-option parameter)。每个跟在"--"长选项后的parameter总是被解析为非选项类型的参数(non-option parameter)。如果设置了环境变量`POSIXLY_CORRECT`，或者如果短选项字符使用"+"开头，所有剩余的参数被解析为非选项类型的参数(non-option parameter)。


## OUTPUT

OUTPUT是前一小节所描述的每个元素生成的。output输出的顺序和input给定的顺序是一致的，但非选项类型的参数(non-option parameter)除外。可以使用兼容模式(不使用引号保护)处理output，也可以使用引号包围的模式处理output，这样空白字符、argument中的特殊字符以及非选项类型的参数都可以被原样保留(见下面的QUOTING)。在shell脚本中处理output时，一般来说，各个元素可以一个一个地被处理(大多数shell语言中可以使用shift命令实现)，但这种模式是有缺陷的，非引号保护模式下，如果有特殊字符，各个元素可能会被切割在预料之外的位置。

如果解析parameter时出现了问题，例如需要的argument没有找到，或者无法识别某个选项，将向stderr中报告错误，且产生错误的那个选项将不会有任何output，并返回一个非0状态码。

对于短选项，将生成一个由"-"和一个选项字符组成的parameter。如果选项有选项参数(argument)，下一个parameter将被作为该选项的argument。如果选项有一个可选参数，但没找到这个参数，此时，如果在引号保护模式下，将生成一个使用引号包围的空参数作为下一个参数，如果在兼容模式(非引号保护)下，则不会生成下一个参数。注意，许多其它版本的getopt(1)不支持可选参数。

(译注：例如"-a"选项，将生成`-a`作为一个parameter，如果它有选项参数，则生成`-a ARG`共两个parameter，如果选项参数可选，在引号保护模式下，生成`"-a" ""`共两个parameter，在非引号保护模式下，将只有`-a`一个parameter)

如果单个横线"-"后面一次性跟了多个短选项字符，将会分隔它们为独立的parameter。(译注，如"-avz"将生成`-a -v -z`共3个parameter)。

对于长选项，将生成一个由"--"和完整的长选项名组成的parameter。只要它是长选项，无论它是缩写形式的，还是使用单个短横线"-"指定的，生成parameter时都会补齐。此外，长选项的参数处理方式和短选项处理方式一样。

一般来说，在处理完所有option和它们的argument之前，不会生成非选项类型的参数(non-option parameter)的output。在处理非选项类型参数时，首先会生成一个"--"作为一个独立的parameter，然后各个非选项类型的参数(non-option parameter)将按照顺序生成各自的output，每个都是单独的parameter。只有当短选项字符的第一个字符是"-"时，才会在它们被发现的位置处生成该非选项类型的参数(non-option parameter)的Output(如果使用语法1，将不支持该功能。此时，所有前缀的"-"和"+"都会被忽略)。


## QUOTING

在兼容模式下，空白字符、argument或非选项类型的参数(non-option parameter)中的特殊字符将不会被正确处理。因为output是提供给shell脚本的，脚本并不知道如何将这个output拆分成不同的parameter。要避免该问题，可以使用加强版getopt提供的QUOTING功能，它会使用引号保护这些parameter，当这些output再次提供给shell时(一般会用eval命令来解析)，将能正确地被拆分成独立的parameter。

如果设置了GETOPT_COMPATIBLE环境变量，或者使用了语法1，或者使用了"-u"选项，将禁用QUOTING功能。

不同的shell使用不同的引号处理规则。可以使用"-s"选项指定你想使用的shell。目前支持的shell类型有：sh、bash、csh和tcsh。事实上，只有两种风格的区别：类sh(sh-like)的引号处理风格和类csh(csh-like)的引号处理风格。即使你使用其它类型的shell，也仍然可以指定这几种shell。


## SCANNING MODES

短选项的第一个字符可以使用"+"或"-"开头来指定一种扫描模式。如果使用语法1，这两个特殊符号将被忽略，但如果设置了环境变量POSIXLY_CORRECT，则仍然会执行扫描。

如果第一个字符为"+"，或者设置了环境变量POSIXLY_CORRECT，只要发现了第一个非选项类型的参数(non-option parameter)就会停止解析(例如，parameter未使用"-"开头)，而且它后面的所有参数都会被解释为非选项类型的参数(non-option parameter)。

如果第一个字符为"-"，非选项类型的参数(non-option parameter)的output将根据它所在的位置进行生成。而默认情况下，它们是被收集到一个独立的"--"参数之后的。在这种扫描模式下，其实也会生成"--"参数，只不过生成在所有parameter的最尾部。
```