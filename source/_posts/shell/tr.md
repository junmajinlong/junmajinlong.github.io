---
title: tr命令
p: shell/tr.md
date: 2023-07-11 08:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

<a name="blog1.1"></a>

# tr命令

tr主要用于将从标准输入读取的数据进行结果集映射、字符压缩和字符删除。它首先会将读取的标准输入进行排序然后按照某种方式换行，然后再根据给出的命令行参数做相关处理。

```bash
tr [options] [SET1] [SET2]
-c：使用SET1的补集  
-d：删除字符  
-s：压缩字符  
-t：截断SET1，使得SET1的长度和SET2的长度相同  
```

<a name="blog1.2"></a>
## tr映射

如果同时指定了SET1和SET2，则是将SET1的符号按位置一一对应映射为SET2中的符号。换句话说，就是对应替换。

tr接收到stdin后首先会把将结果按照某种标记符号进行换行。例如：

```bash
# 其中"one space.log"是带有空格的文件名
[root@xuexi tmp]# ls 
a  b  c  d  logdir  one  one space.log  shdir  sh.txt  space.log  test  vmware-root
```

将空格替换为制表符。因为tr一接收到数据就进行了排序换行，所以结果仅只替换了`one space.log`中的空格。

```bash
# 结果是排序后换行的
[root@xuexi tmp]# ls | tr " " "\t"
a
b
c
d
logdir
one
one    space.log
shdir
sh.txt
space.log
test
vmware-root
```

之所以说tr是映射而不是替换，是因为两个结果集替换的时候符号位置是一一对应的。如果SET1比SET2短，则SET2多余的部分会被忽略，如果SET1比SET2长，POSIX认为这是不合理的，但也能执行，只不过结果有些意料之外，见下文。例如下面的例子，因为SET1中只有一个符号`\n`，于是替换时SET2中的Y被忽略。

```bash
[root@xuexi tmp]# ls | tr "\n" "XY" 
aXbXcXdXlogdirXoneXone space.logXshdirXsh.txtXspace.logXtestXvmware-rootX
```

这样就可以实现简单的加密和解密。

```bash
# 加密
[root@xuexi tmp]# echo "12345" | tr "0-9" "9876543210"
87654
# 解密
[root@xuexi tmp]# echo "87654" | tr "0-9" "9876543210"
12345
```

上面的过程是将管道左边的12345对应到0-9的展开式0123456789，并将对应位映射到SET2的数字上。解密也是同理。

有一种ROT13加密算法，它的加密和解密使用一套字符。它的SET1的字母位和SET2的字母位完全反向成对。例如SET1指定符号是`axy`，如果SET2想将其对应为`opq`，则须将SET1扩展为`axyopq`，SET2扩展为`opqaxy`，最终是`(a,x,y,o,p,q)`和`(o,p,q,a,x,y)`,这样`(a,o)`和`(o,a)`就能成功配对。再将其扩展为A-Z和a-z，就是所谓的ROT13加密，甚至还可以将0-9也加上去和9-0对应。下面是SET1和SET2的对应式。

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm
```

现在加密`I love you`

```bash
[root@xuexi tmp]# echo "I love you" | tr "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz" "NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm" 
V ybir lbh
```

将`V ybir lbh`解密。

```bash
[root@xuexi tmp]# echo "V ybir lbh" | tr "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz" "NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm"
I love you
```

<a name="blog1.3"></a>
## 完全对应的替换

默认情况下，当指定的SET1比SET2字符长时，从最后一个对应的位置开始，SET1的剩余字符都和SET2的最后一个字符对应。假如`SET1=[1234]`，`SET2=[abc]`，则3对应c，4也对应c，此时如果tr的操作对象中出现3或者4都会被替换为c。

使用`-t`可以先截断SET1比SET2中长的字符，例如上面截断多余的4，`SET1=[123]`和`SET2=[abc]`就实现了完全对应。

```bash
[root@xuexi tmp]# cat x.txt
NO Name SubjectID Mark 备注
1  longshuai 001  56 不及格
2  gaoxiaofang  001 60 及格
3  zhangsan 001 50 不及格
4  lisi    001   80 及格
5  wangwu   001   90 及格
[root@xuexi tmp]# cat x.txt | tr  "fang" "jin"   # 结果中n和g都被替换为了n
NO Nime SubjectID Mirk 备注
1  lonnshuii 001  56 不及格
2  nioxiiojinn  001 60 及格
3  zhinnsin 001 50 不及格
4  lisi    001   80 及格
5  winnwu   001   90 及格
[root@xuexi tmp]# cat x.txt | tr -t "fang" "jin"  # g被截断，只对应替换fan为jin
NO Nime SubjectID Mirk 备注
1  longshuii 001  56 不及格
2  gioxiiojing  001 60 及格
3  zhingsin 001 50 不及格
4  lisi    001   80 及格
5  wingwu   001   90 及格
```

<a name="blog1.4"></a>
## 压缩符号

这功能太爽了。

```bash
tr -s [SET1] [SET2]
```

如果不指定SET2，则仅只压缩，不做替换。SET1可以指定多个字符，这样会对每个字符都进行压缩，例如`tr -s "0a"`，即会压缩连续的0，也会压缩连续的a。如果指定了SET2，则压缩后还一一对应地进行替换。

假如x.txt文件中的内容如下，空格有的地方多，有的地方少，也就是说这是一个没有格式的文件。

```bash
[root@xuexi tmp]# cat x.txt
NO Name SubjectID Mark 备注
1  longshuai 001  56 不及格
2  gaoxiaofang  001 60 及格
3  zhangsan 001 50 不及格
4  lisi    001   80 及格
5  wangwu   001   90 及格
```

使用tr压缩空格使其变的规则。

```bash
[root@xuexi tmp]# cat x.txt | tr -s " "
NO Name SubjectID Mark 备注
1 longshuai 001 56 不及格
2 gaoxiaofang 001 60 及格
3 zhangsan 001 50 不及格
4 lisi 001 80 及格
5 wangwu 001 90 及格
```

如果指定SET2，假如替换为`-`。

```bash
[root@xuexi tmp]# cat x.txt | tr -s " " "-" 
NO-Name-SubjectID-Mark-备注
1-longshuai-001-56-不及格
2-gaoxiaofang-001-60-及格
3-zhangsan-001-50-不及格
4-lisi-001-80-及格
5-wangwu-001-90-及格
```

<a name="blog1.5"></a>
## 删除符号和补集

`tr -d`是删除指定的符号，只能接一个SET1。

```bash
[root@xuexi tmp]# cat x.txt | tr -d " "
NONameSubjectIDMark备注
1longshuai00156不及格
2gaoxiaofang00160及格
3zhangsan00150不及格
4lisi00180及格
5wangwu00190及格
```

`tr -c SET1 SET2`是将标准输入按照SET1求补集，并将补集部分的字符全部替换为SET2，即将不在标准输入中存在但SET1中不存在的字符替换为SET2的字符。但是SET2如果指定的字符大于1个，则只取最后一个字符作为替换字符。使用`-c`的时候应该把`-c SET1`作为一个整体，不要将其分开。

例如：

```bash
[root@xuexi tmp]# echo "abcdefo"| tr -c "ao" "y"
ayyyyyoy[root@xuexi tmp]# 
```

标准输入`abcdefo`按照`SET1="ao"`求得的补集为`bcdef`，将它们替换为y，结果即为`ayyyyyo`，但是结果的最后面多了一个y并且紧接着命令提示符。这是因为`abcdefo`尾部的`\n`也是ao的补集的一部分，并将其替换为y了。如果不想替换最后的`\n`，可以在SET1中指定`\n`。

```bash
[root@xuexi tmp]# echo "abcdefo"| tr -c "ao\n" "y" 
ayyyyyo
```

如果SET2指定多个字符，将只取最后一个字符作为替换字符。

```bash
[root@xuexi tmp]# echo "abcdefo"| tr -c "ao\n" "ay"
ayyyyyo
[root@xuexi tmp]# echo "abcdefo"| tr -c "ao\n" "yb"
abbbbbo
```

`-c`常和`-d`一起使用，如`tr -d -c SET1`。它先执行`-c SET1`求出SET1的补集，再对这个补集执行删除。也就是说，最终的结果是完全匹配SET1中的字符。注意，`-d`一定是放在`-c`前面的，否则被解析为`tr -c SET1 SET2`，执行的就不是删除补集，而是替换补集为`-d`的最后一个字符d了。

```bash
# 对数字和分行符求补集，并删除这些补集符号
[root@xuexi tmp]# echo "one 1 two 2 three 3"| tr -d -c "[0-9]\n"
123
# 再加一个空格求补集
[root@xuexi tmp]# echo "one 1 two 2 three 3"| tr -d -c "[0-9] \n"
 1  2  3
 # -d选项放在-c选项的后面是替换行为
[root@xuexi tmp]# echo "one 1 two 2 three 3"| tr -c "[0-9]\n" -d
dddd1ddddd2ddddddd3
# 保留字母
[root@xuexi tmp]# echo "one 1 two 2 three 3"| tr -d -c "[a-zA-z]\n"
onetwothree
# 保留字母的同时保留空格
[root@xuexi tmp]# echo "one 1 two 2 three 3"| tr -d -c "[a-zA-z] \n"
one  two  three
```

从上面补集的实验中可以看到，其实指定的`[0-9]`和`[a-z]`是一个字符类，最终的结果显示的是这个类中的对象。

在tr中可以使用以下几种字符类。这些类也可以用在其他某些命令中。

```
[:alnum:]所有的数字和字母。  
[:alpha:]所有的字母。  
[:blank:]所有水平空白=空格+tab。  
[:cntrl:]所有控制字符（非打印字符），在ascii表中的八进制0-37对应的字符和177的del。  
[:digit:]所有数字。  
[:graph:]所有打印字符，不包含空格=数字+字母+标点。  
[:lower:]所有小写字母。  
[:print:]所有打印字符，包含空格=数字+字母+标点+空格。  
[:punct:]所有标点符号。  
[:space:]所有水平或垂直空白=空格+tab+分行符+垂直tab+分页符+回车键。  
[:upper:]所有大写字母。  
[:xdigit:]所有十六进制数字
```

使用方法例如下面的。例如`[:upper:]`等价于`[A-Z]`，`[:digit:]`等价于`[0-9]`。

```bash
[root@xuexi tmp]# echo "one ONE 1 two TWO 2 three THREE 3" | tr -d -c "[:upper:] \n"
 ONE   TWO   THREE 
[root@xuexi tmp]# echo "one ONE 1 two TWO 2 three THREE 3" | tr -d -c "[:alpha:] \n"
one ONE  two TWO  three THREE 
[root@xuexi tmp]# echo "one ONE 1 two TWO 2 three THREE 3" | tr -d -c "[:digit:] \n"
  1   2   3
```


