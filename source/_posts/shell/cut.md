---
title: cut命令
p: shell/cut.md
date: 2023-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# cut命令

<a name="blog1.1"></a>

## 选项说明

cut命令将行按指定的分隔符分割成多列，它的弱点在于不好处理多个分隔符重复的情况，因此经常结合tr的压缩功能。

```
-b：按字节筛选；  
-n：与"-b"选项连用，表示禁止将字节分割开来操作；  
-c：按字符筛选；  
-f：按字段筛选；  
-d：指定字段分隔符，不写-d时的默认字段分隔符为"TAB"；因此只能和"-f"选项一起使用。  
-s：避免打印不包含分隔符的行；  
--complement：补足被选择的字节、字符或字段（反向选择的意思或者说是补集）；   
--output-delimiter：指定输出分割符；默认为输入分隔符。  
```

假设/tmp/abc.sh中下面所示的内容。注意：第2行到第5行每列不是都以单个空格分隔的，有的地方重复了几个空格，有的地方只有一个空格，也就是说，文本内容不是很规则。并且最后一行完全没有空格。

```bash
[root@xuexi tmp]# cat abc.sh 
NO Name SubjectID Mark 备注
1  longshuai 001  56 不及格
2  gaoxiaofang  001 60 及格
3  zhangsan 001 50 不及格
4  lisi    001   80 及格
5  wangwu   001   90 及格
djakldj;lajd;sla
```

下面是cut的示例。

<a name="blog1.2"></a>

## 按字段筛选

在abc.sh中有5个字段。筛选出第二字段name列和第4字段mark列。使用空格作为分隔符。

```bash
[root@xuexi tmp]# cut -d" " -f2,4 abc.sh
Name

 001
 50


djakldj;lajd;sla
```

可以看到，输出的是乱七八糟的非预期结果。原因就是分隔符空格在分隔的地方重复了多次。所以想要正确显示结果，需要把重复空格处理掉。

可以使用tr工具来压缩连续字符。
```bash
[root@xuexi tmp]# cat abc.sh | tr -s " " | cut -d " " -f2,4
Name Mark
longshuai 56
gaoxiaofang 60
zhangsan 50
lisi 80
wangwu 90
djakldj;lajd;sla
```
但是输出中的最后一行中完全没有定界符的行也输出了，这需要使用`-s`来取消这样的输出。
```bash
[root@xuexi tmp]# cut -d" " -f2,4 abc.sh -s
Name Mark
longshuai 56
gaoxiaofang 60
zhangsan 50
lisi 80
wangwu 90
```

<a name="blog1.3"></a>
## 使用--complement

输出除了第2字段和第4字段其余的所有字段。
```bash
[root@xuexi tmp]# cut -d" " -f2,4 abc.sh -s --complement
NO SubjectID 备注
1 001 不及格
2 001 及格
3 001 不及格
4 001 及格
5 001 及格
```

<a name="blog1.4"></a>
## 按字节或字符分割

英文和阿拉伯数字是单字节字符，中文是双字节字符，甚至是3字节字符。

使用`-b`来按字节筛选，使用`-c`按字符分割。

注意，按字节或字符分割时将不能指定`-d`，因为`-d`是划分字段的。
```bash
[root@xuexi tmp]# cut -b1-3 abc.sh   # 筛选第1-3个字节的内容
NO 
1 l
2 g
3 z
4 l
5 w
dja
```
由于筛选中文，结果中出现乱码。
```bash
[root@xuexi tmp]# cut -b20 abc.sh  
```
所以`-b`选项需要结合`-n`选项，以禁止`-b`选项将多字节的字符强行分割导致乱码。
```bash
[root@xuexi tmp]# cut -n -b20 abc.sh
a
不
0
及
```
也可以按字符分隔。
```bash
[root@xuexi tmp]# cut -c20 abc.sh    
a
不
0
及
```
<a name="blog1.5"></a>

## 使用--output-delimiter

使用`--output-delimiter`指定输出分隔符。

使用`-b`或者`-c`分隔了多段字符时，可以使用`--output-delimiter`，否则这些多段将拼接在一起。
```bash
# 拼接在一起
[root@xuexi tmp]# cut -b3-5,6-8 abc.sh  
 Name 
longsh
gaoxia
zhangs
lisi 0
wangwu
akldj;

# 逗号分隔多段
[root@xuexi tmp]# cut -b3-5,6-8 abc.sh --output-delimiter ","  
 Na,me 
lon,gsh
gao,xia
zha,ngs
lis,i 0
wan,gwu
akl,dj;
```

<a name="blog1.6"></a>
## cut中的范围指定

可以使用`N-`、`N-M`和`-M`分别表示每行N字符（或字节或字段）后的所有内容、`N-M`段内容和M段之前的内容。注意包括N和M的边界。
```bash
# 输出第三字段和后面所有的内容
[root@xuexi tmp]# cut -d" " -f3- abc.sh -s
SubjectID Mark 备注
001 56 不及格
001 60 及格
001 50 不及格
001 80 及格
001 90 及格
```
范围交叉时，不会重复输出。比如`-f3-5,4-6`，则输出`-f3-6`。
```bash
# 范围交叉
[root@xuexi tmp]# cut -d" " -f3-5,4-6 abc.sh -s
SubjectID Mark 备注
001 56 不及格
001 60 及格
001 50 不及格
001 80 及格
001 90 及格
```
如果范围顺序无序，则Linux会先对范围排序（升序）再输出。例如`-f4-6,2`等价于`-f2,4-6`。
```bash
[root@xuexi tmp]# cut -d" " -f4-6,2 abc.sh -s           
Name Mark 备注
longshuai 56 不及格
gaoxiaofang 60 及格
zhangsan 50 不及格
lisi 80 及格
wangwu 90 及格
```