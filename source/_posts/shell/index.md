---
title: Shell：Shell系列文章
p: shell/index.md
date: 2019-07-06 17:37:29
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**

--------

<a name="blogshell"></a>

<a name="blogshell1"></a>

# Shell脚本、bash特性系列

**本人已经录制了一门Shell进阶的精品课程，专门讲解shell的“神”，可以免去看man bash的痛苦。课程链接：<https://edu.51cto.com/sd/96966>**。

**还录制了一个关于【Bash那些少为人知少为人用的一些内幕和小技巧】，在B站：<https://www.bilibili.com/video/BV1NB4y1u74B>**。

## Shell脚本从入门到深入教程

- [Shell脚本深入教程：快速入门](/shell/script_course/shell_tutorial)  
- [Shell脚本深入教程：升级Bash](/shell/script_course/bash_update)  
- [Shell脚本深入教程：Bash变量](/shell/script_course/shell_var)  
- [Shell脚本深入教程：Shell脚本带颜色输出](/shell/script_course/shell_color)  
- [Shell脚本深入教程：Bash数值运算](/shell/script_course/shell_number_cal)  
- [Shell脚本深入教程：Bash Here Document和Here String用法](/shell/script_course/shell_heredoc_herestr)  
- [Shell脚本深入教程：Bash支持的运算操作](/shell/script_course/shell_op)  
- [Shell脚本深入教程：Bash数组基础](/shell/script_course/shell_array)  
- [Shell脚本深入教程：Bash操作变量和数组元素](/shell/script_course/shell_op_var_arr)  
- [Shell脚本深入教程：Bash路径通配规则](/shell/script_course/shell_glob)  
- [Shell脚本深入教程：Bash命令替换](/shell/script_course/shell_cmd_substitution)  
- [Shell脚本深入教程：Bash进程替换](/shell/script_course/shell_process_substitution)  
- [Shell脚本深入教程：Bash alias深入理解](/shell/script_course/shell_alias)  
- [Shell脚本深入教程：Bash测试命令](/shell/script_course/shell_test)  
- [Shell脚本深入教程：Bash流程控制语句](/shell/script_course/shell_flow_control)  
- [Shell脚本深入教程：Bash read命令读取数据](/shell/script_course/shell_read)  
- [Shell脚本深入教程：Bash函数](/shell/script_course/shell_function)  
- [Shell脚本深入教程：Bash高级重定向](/shell/script_course/shell_redirection)  
- [Shell脚本深入教程：Bash IFS用法详解](/shell/script_course/shell_ifs)  
- [Shell脚本深入教程：Bash trap信号捕捉用法详解](/shell/script_course/shell_trap)  
- [Shell脚本深入教程：Bash解析命令行和eval命令(★★★)](/shell/script_course/shell_cmdline_parse_eval)  
- [Shell脚本深入教程：理解Shell的单双引号(★★★)](/shell/script_course/shell_quotes)  
- [Shell脚本深入教程：Shell环境和子Shell的概念(★★★)](/shell/script_course/shell_env)  
- [Shell脚本深入教程：Bash启动时加载配置文件的顺序和过程](/shell/script_course/bash_startup)  

## 子shell、bash内置命令特殊性、后台任务的本质

- [前后台进程、孤儿进程和daemon类进程的父子关系](/linux/process_relationship)  
- [什么时候进入子shell](/shell/script_course/shell_env)  
- [bash内置命令的特殊性，后台任务的"本质"](/shell/jobs_special)  
- [shell脚本技巧：如何让shell脚本自杀+bash内置命令的特殊性](/shell/kill_script_self)  

## 理解和深入bash

- [彻底搞懂shell的高级I/O重定向](/shell/fd_duplicate)  
- [一个命令行解析和重定向的问题分析](/shell/cmdline_parse_and_redirect)  
- [变量赋值时的命令行解析](/shell/cmdline_parse_for_assignment)  
- [理解$0和BASH_SOURCE](/shell/bash_source)  
- [获取shell脚本所在目录](/shell/get_script_dir)  
- [Bash Here Document的各种语法姿势](/shell/script_course/shell_heredoc_herestr)  
- [Shell内置命令、函数、别名等调用顺序](/shell/call_order)  
- [\$后加引号(\$"string"和\$'string')](/shell/script_course/shell_quotes#tag1)  
- [获取bash中的时间](/shell/bash_time)  

<a name="cmd_opts"></a>
# shell脚本命令行选项设计

- 1.[使用bashly构建bash脚本命令行](/shell/use_bashly)  
- 2.[getopt设计shell脚本选项(精)](/shell/getopt)  
- 3.[man getopt中文手册(翻译)](/shell/getopt_translate)  

# 正则表达式、grep、sed、awk

## grep命令中文手册

- [grep命令中文手册(精)](/shell/grep_translate)  

## 正则表达式用法详解

我录制了两个正则表达式相关的视频教程：  
- **[正则表达式入门教程](https://edu.51cto.com/sd/73e2f)**  
- **[揭开正则匹配的面纱：精通高级正则表达式](https://edu.51cto.com/sd/f77d7)**

正则表达式博客教程：  
- [1.基础正则表达式详解](/shell/regex_basic)  
- [2.Perl正则表达式超详细教程(精)](/perl/perl_re)  

<a name="sed+awk"></a>

<a name="sed"></a>

## sed命令用法详解

**sed pdf版下载**:[玩透sed：探究sed原理.pdf](/files/sed_inner.pdf)

下面是网页版：  

- [sed修炼系列(一)：花拳绣腿之入门篇(精)](/shell/sed1)  
- [sed修炼系列(二)：武功心法(infosed翻译+注解)(精)](/shell/sed2)  
- [sed修炼系列(三)：sed高级应用之窗口滑动技术(精)](/shell/sed3)  
- [sed修炼系列(四)：sed中的疑难杂症(精)](/shell/sed4)  
- [sed示例：sed删除拼音的音调](/shell/sed_use1)  
- [sed示例：sed从a文件判断是否删除b文件中的行](/shell/sed_use2)  

<a name="awk"></a>

## awk命令用法详解

我录制了两个awk相关的视频教程：  
- **[精通awk精品课程：awk从入门到精通](https://edu.51cto.com/sd/1d5c7)**
- **[Awk经典实战案例精讲](https://edu.51cto.com/sd/0dd13)**

几十篇awk系列博客文章参见：[精通awk系列文章](/shell/awk/index)  

# find命令和xargs命令用法详解

- [1.Linux find常用用法示例(精)](/shell/find_usage)  
- [2.Linux find运行机制详解(精)](/shell/find_intermediate)  
- [3.xargs原理剖析和用法详解(精)](/shell/xargs)  

<a name="blogparallel"></a>
# shell并发执行任务

- 1.[shell高效处理文本(1)：xargs并行处理](/shell/xargs_parallel)(精)  
- 2.GNU Parallel  

# 其它常用命令

- [dd、split和csplit命令](/shell/dd_split_csplit)  
- [数学运算和bc命令](/shell/cal_bc)  
- [expr命令全解](/shell/expr)  
- [date、sleep、usleep命令](/shell/data_sleep_usleep)  
- [tr命令用法和特性全解](/shell/tr)  
- [cut命令](/shell/cut)  
- [玩透sort命令](/shell/sort)  
- [sort命令中文手册(翻译：info sort)](/shell/sort_trans)  
- [uniq命令](/shell/uniq)  
- [seq命令](/shell/seq)  
- [tee的花式用法和pee](/shell/tee_pee)  
- [Bash mapfile按行读取到bash数组](/shell/mapfile)  
- [comm命令求出文件的交集、差集](/shell/comm)
- [md5sum批量比较多个文件的内容是否相同](/shell/check_diff_files)  

<a name="shellothers"></a>

# Shell杂七杂八和脚本示例、技巧

- [Shell脚本杀掉除自己外的旧进程](/shell/kill_old_process)  
- [Linux快速生成大量随机大小的文件](/shell/many_small_files)  
- [Linux快速删除大量小文件(多种方式速度对比)](/shell/rm_many_smallfiles)  
- [修改shell命令提示符和命令的输入颜色](/shell/cmd_color)  
- [shell生成指定长度的随机数](/linux/gen_random)  
- [shell计算毫秒级、微秒级时间差](/shell/time_diff)  
- [shell脚本示例：shell脚本动画小工具(shell版和perl版)](/shell/shell_perl_gif)  
- [shell脚本示例：安全的rm命令，保护重要数据](/shell/rm_is_safe)  
- [functions文件详细分析和说明](/shell/functions)  
- [如何写SysV服务管理脚本](/shell/sysv_script)  

