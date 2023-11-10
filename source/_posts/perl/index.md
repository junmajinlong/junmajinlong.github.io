---
title: Perl系列文章
p: perl/index.md
date: 2019-07-06 17:37:29
tags: Perl
categories: Perl
---

# 0.Perl书籍推荐

Perl是第一门我认真学习的通用语言，学习Perl，收获了很多Perl和非Perl的东西，感谢Perl，感谢那些好书，在此特地分享我收集的一些书籍和书籍推荐：  
[**Perl书籍下载**](https://pan.baidu.com/s/1-NXIzZK3C_IZNZ8zmFtQzg)  密码：uivf

下面是一些我学习Perl过程中曾读过完整的或部分章节且觉得好的书籍：  
- **入门级别1**：《Perl语言入门》即小骆驼，完全Perl 0基础最佳最快入门书籍  
- **入门级别2**：《Intermediate Perl》即羊驼，中文版《Perl进阶》  
- **入门后复习**：《Beginning Perl》，中文版《Perl入门经典》，似乎网上没什么评价，但个人觉得是系统性学习Perl的非常好的资料  
- **系统性学习和进阶**：《Pro Perl》，中文版《Perl高级编程》（是我整理、完善Perl的最佳书籍）  
- **Perl编码技巧**：《Effective Perl Programming》，中文版《Perl高效编程》  

关于《精通Perl》和《Perl语言编程》（即羊驼一家和大骆驼），虽网上评价很高，但我个人终觉不适合：
- 《精通Perl》是作者(brian d foy)以第一人称来描述他怎么理解Perl的  
- 《Perl语言编程》是Larry Wall自己编写的书籍，也许他智商太高，书中很多地方的跳跃性都非常大，不过这本书确实好，介绍了很多Perl的机制和【内幕】  

然后是某个方向的书籍，比如：  
- http客户端《perl lwp》(循序渐进，作者的写作方式非常友好)  
- 数据库操作《Programming the Perl DBI》（有中文版《Perl DBI编程》，如有数据库基础和文本处理经验，将有很多意外收获，可能是对数据库的一些顿悟）  
- 网络编程《Perl网络编程》(虽然非常老的书，但仍然强烈推荐)  

最后，是我的这些博客，它们是我阅读这些书籍的读书笔记，更多的是我测试和补充的内容，可以免去看英文版，也免去书中的一大堆废话，老外的书，你懂的。

# 1.Perl语言入门

本部分是《Perl语言入门 第六版》(英文书名：Learning Perl)的学习笔记，这本书是Perl家族的"小羊驼"书籍。如果有shell基础，perl入门挺容易的。

## Perl入门秘籍

我重写了之前的Perl教程部分，内容更详实，解释更透彻，知识点组织分布更友好。

秘籍链接：**Perl入门秘籍**<https://perl-book.junmajinlong.com/>。如阅读过程中有问题，请在本文回复。

看本秘籍，就不用再看下面列出的《入门基础》了。

## 入门基础

<table><tbody><tr>
<td><ul>
  <li><a href="/perl/perl_syntax">1.Perl语法的基本规则</a><br></li>
  <li><a href="/perl/perl_num_str">2.Perl的数值和字符串</a><br></li>
  <li><a href="/perl/perl_var">3.Perl的变量</a><br></li>
  <li><a href="/perl/perl_addadd">4.Perl中的自增、自减</a><br></li>
  <li><a href="/perl/perl_cmp">5.Perl的比较操作符</a><br></li>
  <li><a href="/perl/perl_flow_control">6.Perl的流程控制语句</a><br></li>
  <li><a href="/perl/perl_undef_defined">7.Perl的undef类型和defined()函数</a></li>
  <li><a href="/perl/perl_stdin_chomp">8.Perl读取输入&lt;STDIN&gt;、&lt;&gt;和chomp函数</a><br></li>
  <li><a href="/perl/perl_list_array">9.Perl的列表和数组</a><br></li>
  <li><a href="/perl/perl_hash">10.Perl的hash类型</a><br></li>
  <li><a href="/perl/perl_context">11.Perl的执行上下文</a><br></li>
  <li><a href="/perl/perl_slice">12.Perl分片技术</a><br></li>
  <li><a href="/perl/perl_output">13.Perl的输出：print、say和printf</a><br></li>
  <li><a href="/perl/perl_subroutine">14.Perl的子程序</a><br></li>
  <li><a href="/perl/perl_do_block">15.Perl的do语句块结构</a><br></li>
  <li><a href="/perl/perl_die_warn">16.Perl的die和warn函数</a><br></li>
</ul></td>
<td><ul>
  <li><a href="/perl/perl_cmd_args_argv">17.Perl的命令行参数和ARGV</a><br></li>
  <li><a href="/perl/perl_io_handler">18.Perl的IO操作(1)：文件句柄</a><br></li>
  <li><a href="/perl/perl_io_handler_2">19.Perl的IO操作(2)：更多文件句柄模式</a><br></li>
  <li><a href="/perl/perl_io_handler_vars">20.Perl文件句柄相关的常见变量</a><br></li>
  <li><a href="/perl/perl_file_test_stat">21.Perl文件测试操作和stat函数</a><br></li>
  <li><a href="/perl/perl_glob_find">22.Perl文件名通配和文件查找</a><br></li>
  <li><a href="/perl/perl_file_dir_op">23.Perl文件、目录常用操作</a><br></li>
  <li><a href="/perl/perl_file_copy_move_rename">24.Perl复制、移动、重命名文件/目录</a><br></li>
  <li><a href="/perl/perl_time_localtime_gmtime">25.Perl的time、localtime和gmtime函数</a><br></li>
  <li><a href="/perl/perl_re">26.Perl正则表达式超详细教程</a><br></li>
  <li><a href="/perl/perl_substitute_split_join">27.Perl处理数据(一)：s替换、split和join</a><br></li>
  <li><a href="/perl/perl_tr_y">28.Perl处理数据(二)：tr和y///</a><br></li>
  <li><a href="/perl/perl_module">29.Perl模块管理</a><br></li>
  <li><a href="/perl/perl_module_inc">30.Perl使用模块和@INC</a><br></li>
  <li><a href="/perl/perl_system_exec">31.Perl和OS交互(一)：system、exec和反引号</a><br></li>
  <li><a href="/perl/perl_fork">32.Perl和OS交互(二)：fork</a><br></li>
</ul></td>
</tr></tbody></table>


## 其它基础

- [1.Perl函数：字符串相关函数](/perl/perl_str_funcs)  
```
chomp, chop, chr, crypt, fc, hex, index, lc, 
lcfirst, length, oct, ord, pack, q//, qq//, 
reverse, rindex, sprintf, substr, tr///, 
uc, ucfirst, y///
```
- [2.Perl函数：列表相关函数](/perl/perl_list_funcs)  
```
grep, join, map, qw//, reverse, sort, unpack
```
- [3.Perl函数：数组和hash相关函数](/perl/perl_arr_hash_funcs)  
```
数组：each, keys, pop, push, shift, splice, unshift, values
hash：delete, each, exists, keys, values
```
- [4.List::Util模块用法详解](/perl/perl_list_utils)   

<p><a name="blogperloneline"></a></p>

# Perl一行式程序

这部分分3部分，内容比较多，算得上是一本薄书了，所以专门加上了一个《序言》，让它看上去更像是书。

第一部分是针对没有Perl基础，但想用perl一行式命令的人，用于快速掌握学习perl一行式时所必须知道的Perl基础知识。

第二部分是perl的选项、特殊变量的收集，没有多少示例，只是它们详细的解释，专门用来做perl一行式的参考手册或者熟练后的速查手册。第一次学perl一行式的人不建议直接看这一篇文章，而是直接从后面的示例部分开始看，需要完整、详细说明的时候再回来看这篇文章中对应的内容。

第三部分是一大堆perl一行式的使用示例(分成了好几篇文章)，也是学习perl一行式的入口，前提是你已经具备了Perl基础知识。这些例子不一定都是实用的例子，只是为了抛砖引玉。这部分会针对用法来对选项、perl语句做不完整解释，如果想要知道完整的解释，看第二部分的文章。

示例部分主要来自于《Perl One-Liners》这本书，但我自己对内容进行了大量扩充，也进行了更多的解释。

- [1.序言：我为什么学Perl](/perl/perl_why_perl)  
- [2.0基础学习Perl一行式必知的Perl基础](/perl/perl_oneliner_basic)  
- [3.perl选项、特殊变量参考手册](/perl/perl_options_vars)  
- [4.Perl一行式：处理空白符号](/perl/perl_oneliner_1)  
- [5.Perl一行式：处理行号和单词数](/perl/perl_oneliner_2)  
- [6.Perl一行式：字段处理和计算](/perl/perl_oneliner_3)  
- [7.Perl一行式：文本编解码、替换](/perl/perl_oneliner_4)  
- [8.Perl一行式：选择输出、删除的行](/perl/perl_oneliner_5)  


# Perl语言进阶

本部分是《Intermediate Perl 2nd》的学习笔记，这本书是骆驼家族的"羊驼"书，用于Perl的基础进阶学习。部分内容来自《Beginning Perl》，这也是一本好书。

## 引用

- [1.Perl引用入门](https://www.cnblogs.com/f-ck-need-u/p/9708263.html)  
- [2.Perl解除引用：从引用还原到数据对象](https://www.cnblogs.com/f-ck-need-u/p/9710562.html)  
- [3.Perl检查引用类型](https://www.cnblogs.com/f-ck-need-u/p/9713851.html)  
- [4.Perl匿名数组、hash和autovivification特性](https://www.cnblogs.com/f-ck-need-u/p/9718238.html)  
- [5.Perl的浅拷贝和深度拷贝](https://www.cnblogs.com/f-ck-need-u/p/9721265.html)  
- [6.Perl输出复杂数据结构：Data::Dumper,Data::Dump,Data::Printer](https://www.cnblogs.com/f-ck-need-u/p/9719282.html)  
- [7.Perl数据序列化和持久化(入门)：Storable模块](https://www.cnblogs.com/f-ck-need-u/p/9723368.html)    
- [8.Perl子程序引用和匿名子程序](https://www.cnblogs.com/f-ck-need-u/p/9733283.html)    
- [9.一文搞懂：词法作用域、动态作用域、回调函数、闭包](https://www.cnblogs.com/f-ck-need-u/p/9735955.html)    
- [10.Perl回调函数和闭包](https://www.cnblogs.com/f-ck-need-u/p/9738156.html)  
- [11.Perl文件句柄引用](https://www.cnblogs.com/f-ck-need-u/p/9740059.html)  
- [12.Perl正则表达式引用](https://www.cnblogs.com/f-ck-need-u/p/9743199.html)  
- [13.排序变换思路：施瓦茨变换](https://www.cnblogs.com/f-ck-need-u/p/9744275.html)  

## 包和模块  

- [1.Perl导入代码文件(eval、do、require)](https://www.cnblogs.com/f-ck-need-u/p/9770651.html)  
- [2.Perl包和模块(内容来自beginning perl)](https://www.cnblogs.com/f-ck-need-u/p/9779949.html)  
- [3.Perl包相关](https://www.cnblogs.com/f-ck-need-u/p/9771806.html)  
- [4.Perl特殊代码块：BEGIN、CHECK、INIT、END和UNITCHECK](https://www.cnblogs.com/f-ck-need-u/p/9780625.html)  
- [5.Perl：写POD文档](https://www.cnblogs.com/f-ck-need-u/p/9781455.html)  
- [6.Perl构建和打包自己的模块](https://www.cnblogs.com/f-ck-need-u/p/9782737.html)  

## Perl面向对象

- [1.Perl面向对象(1)：从代码复用开始](https://www.cnblogs.com/f-ck-need-u/p/9798757.html)  
- [2.Perl面向对象(2)：对象](https://www.cnblogs.com/f-ck-need-u/p/9811119.html)  
- [3.Perl面向对象(3)：解构——对象销毁](https://www.cnblogs.com/f-ck-need-u/p/9818016.html)  

**待续。。。**

# Perl进程、线程、IO

- [1.Perl信号处理](https://www.cnblogs.com/f-ck-need-u/p/10386248.html)  
- [2.Perl多进程](https://www.cnblogs.com/f-ck-need-u/p/10386933.html)  
- [3.Perl处理和收走子进程](https://www.cnblogs.com/f-ck-need-u/p/10389627.html)  
- [4.Perl进程：僵尸进程和孤儿进程](https://www.cnblogs.com/f-ck-need-u/p/10508297.html)  
- [5.Perl进程间通信](https://www.cnblogs.com/f-ck-need-u/p/10400540.html)  
- [6.Perl SysV IPC](https://www.cnblogs.com/f-ck-need-u/p/10404275.html)  
- [7.Perl线程(1)：解释器线程特性和线程管理](https://www.cnblogs.com/f-ck-need-u/p/10420910.html)  
- [8.Perl线程(2)：数据共享和线程安全](https://www.cnblogs.com/f-ck-need-u/p/10422445.html)  
- [9.Perl线程队列：Thread\:\:Queue](https://www.cnblogs.com/f-ck-need-u/p/10422293.html)  
- [10.Perl线程池](https://www.cnblogs.com/f-ck-need-u/p/10422449.html)  
- [11.Perl IO：简介和常用IO模块](https://www.cnblogs.com/f-ck-need-u/p/10442177.html)  
- [12.Perl IO：read()函数](https://www.cnblogs.com/f-ck-need-u/p/10442188.html)  
- [13.Perl IO：随机读写文件](https://www.cnblogs.com/f-ck-need-u/p/10446013.html)  
- [14.Perl IO：文件锁](https://www.cnblogs.com/f-ck-need-u/p/10447881.html)  
- [15.Perl IO：IO重定向](https://www.cnblogs.com/f-ck-need-u/p/10450299.html)  
- [16.Perl IO：操作系统层次的IO](https://www.cnblogs.com/f-ck-need-u/p/10459768.html)  

# 网络编程

- [Perl获取主机名、用户、组、网络信息](https://www.cnblogs.com/f-ck-need-u/p/10486209.html)  

# balabala

[Perl输出带颜色行号或普通输出行](https://www.cnblogs.com/f-ck-need-u/p/10791792.html)





