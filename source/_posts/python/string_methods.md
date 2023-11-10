---
title: Python字符串对象方法整理
p: python/string_methods.md
date: 2020-02-03 10:37:29
tags: Python
categories: Python
---


--------

**回到：[Python系列文章](/python/index)**  

--------

Python中字符串对象提供了很多方法来操作字符串，功能相当丰富。
```
>> print(dir(str))

'capitalize', 
'casefold', 
'center', 
'count', 
'encode', 
'endswith', 
'expandtabs', 
'find', 
'format', 
'format_map', 
'index', 
'isalnum', 
'isalpha', 
'isdecimal', 
'isdigit', 
'isidentifier', 
'islower', 
'isnumeric', 
'isprintable', 
'isspace', 
'istitle', 
'isupper', 
'join', 
'ljust', 
'lower', 
'lstrip', 
'maketrans', 
'partition', 
'replace', 
'rfind', 
'rindex', 
'rjust', 
'rpartition', 
'rsplit', 
'rstrip', 
'split', 
'splitlines', 
'startswith', 
'strip', 
'swapcase', 
'title', 
'translate', 
'upper', 
'zfill'
```
这些方法的使用说明见[官方文档:string methods](https://docs.python.org/3/library/stdtypes.html#string-methods)，本文对它们进行详细解释，各位以后可将本文当作手册。

这里没有模式匹配(正则)相关的功能。python中要使用模式匹配相关的方法操作字符串，需要`import re`导入re模块。关于正则模式匹配，参见：[re Module Contents](https://docs.python.org/3/library/re.html#module-contents)。

注意，Python中**字符串是不可变对象**，所以所有修改和生成字符串的操作的实现方法都是另一个内存片段中新生成一个字符串对象。例如，`'abc'.upper()`将会在划分另一个内存片段，并将返回的`ABC`保存在此内存中。

下文出现的"S"表示待操作的字符串。本文没有对`casefold,encode,format,format_map`进行介绍，前两者和unicode有关，后两者内容有点太多。

<a name="blog1"></a>
# 1.大小写转换

<a name="blog1.1"></a>
## 1.1 lower、upper

>S.lower()
>S.upper()

返回S字符串的小写、大写格式。(注意，这是新生成的字符串，在另一片内存片段中，后文将不再解释这种行为)

例如：
```
>>> print('ab XY'.lower())
ab xy
>>> print('ab XY'.upper())
AB XY
```

<a name="blog1.2"></a>
## 1.2 title、capitalize

>S.title()
>S.capitalize()

前者返回S字符串中所有单词首字母大写且其他字母小写的格式，后者返回首字母大写、其他字母全部小写的新字符串。

例如：
```
>>> print('ab XY'.title())
Ab Xy
>>> print('abc DE'.capitalize())
Abc de
```

<a name="blog1.3"></a>
## 1.3 swapcase

>S.swapcase()

`swapcase()`对S中的所有字符串做大小写转换(大写-->小写，小写-->大写)。

```
>>> print('abc XYZ'.swapcase())
ABC xyz
```

<a name="blog2"></a>
# 2.isXXX判断

<a name="blog2.1"></a>
## 2.1 isalpha,isdecimal,isdigit,isnumeric,isalnum

>S.isdecimal()
>S.isdigit()
>S.isnumeric()
>S.isalpha()
>S.isalnum()

测试字符串S是否是数字、字母、字母或数字。对于非Unicode字符串，前3个方法是等价的。

例如：
```
>>> print('34'.isdigit())
True
>>> print('www.junmajinlong.com'.isalpha())
True
>>> print('www.junmajinlong.com'.isalnum())
True
```

<a name="blog2.2"></a>
## 2.2 islower,isupper,istitle

>S.islower()
>S.isupper()
>S.istitle()

判断是否小写、大写、首字母大写。要求S中至少要包含一个字符串字符，否则直接返回False。例如不能是纯数字。

注意，`istitle()`判断时会对每个单词的首字母边界判断。例如，`word1 Word2`、`word1_Word2`、`word1()Word2`中都包含两个单词，它们的首字母都是"w"和"W"。因此，如果用`istitle()`去判断它们，将返回False，因为`w`是小写。

例如：
```
>>> print('a34'.islower())
True
>>> print('AB'.isupper())
True
>>> print('Aa'.isupper())
False
>>> print('Aa Bc'.istitle())
True
>>> print('Aa_Bc'.istitle())
True
>>> print('Aa bc'.istitle())
False
>>> print('Aa_bc'.istitle())
False

# 下面的返回False，因为非首字母C不是小写
>>> print('Aa BC'.istitle())
False
```

<a name="blog2.3"></a>
## 2.3 isspace,isprintable,isidentifier

>S.isspace()
>S.isprintable()
>S.isidentifier()

分别判断字符串是否是空白(空格、制表符、换行符等)字符、是否是可打印字符(例如制表符、换行符就不是可打印字符，但空格是)、是否满足标识符定义规则。

例如：

1. 判断是否为空白。没有任何字符是不算是空白。  
```
>>> print(' '.isspace())
True
>>> print(' \t'.isspace())
True
>>> print('\n'.isspace())
True
>>> print(''.isspace())
False
>>> print('Aa BC'.isspace())
False
```
2. 判断是否是可打印字符。  
```
>>> print('\n'.isprintable())
False
>>> print('\t'.isprintable())
False
>>> print('acd'.isprintable())
True
>>> print(' '.isprintable())
True
>>> print(''.isprintable())
True
```
3. 判断是否满足标识符定义规则。  
标识符定义规则为：只能是字母或下划线开头、不能包含除数字、字母和下划线以外的任意字符。  
```
>>> print('abc'.isidentifier())
True
>>> print('2abc'.isidentifier())
False
>>> print('abc2'.isidentifier())
True
>>> print('_abc2'.isidentifier())
True
>>> print('_abc_2'.isidentifier())
True
>>> print('_Abc_2'.isidentifier())
True
>>> print('Abc_2'.isidentifier())
True
```



<a name="blog3"></a>
# 3.填充

<a name="blog3.1"></a>
## 3.1 center

> S.center(width[, fillchar])

将字符串居中，左右两边使用fillchar进行填充，使得整个字符串的长度为width。fillchar默认为空格。如果width小于字符串的长度，则无法填充直接返回字符串本身(不会创建新字符串对象)。

例如：

1. 使用下划线填充并居中字符串
```
>>> print('ab'.center(4,'_'))
_ab_
>>> print('ab'.center(5,'_'))
__ab_
```
2. 使用默认的空格填充并居中字符串
```
>>> print('ab'.center(4))
 ab 
>>> print(len('ab'.center(4)))
4
```
3. width小于字符串长度
```
>>> print('abcde'.center(3))
abcde
```

<a name="blog3.2"></a>
## 3.2 ljust和rjust

>S.ljust(width[, fillchar])
>S.rjust(width[, fillchar])

`ljust()`使用fillchar填充在字符串S的右边，使得整体长度为width。`rjust()`则是填充在左边。如果不指定fillchar，则默认使用空格填充。

如果width小于或等于字符串S的长度，则无法填充，直接返回字符串S(不会创建新字符串对象)。

例如：
```
>>> print('xyz'.ljust(5,'_'))
xyz__
>>> print('xyz'.rjust(5,'_'))
__xyz
```

<a name="blog3.3"></a>
## 3.3 zfill

>S.zfill(width)

用0填充在字符串S的左边使其长度为width。如果S前有正负号`+/-`，则0填充在这两个符号的后面，且符号也算入长度。

如果width小于或等于S的长度，则无法填充，直接返回S本身(不会创建新字符串对象)。

```
>>> print('abc'.zfill(5))
00abc

>>> print('-abc'.zfill(5))
-0abc

>>> print('+abc'.zfill(5))
+0abc

>>> print('42'.zfill(5))
00042

>>> print('-42'.zfill(5))
-0042

>>> print('+42'.zfill(5))
+0042
```

<a name="blog4"></a>
# 4.子串搜索

<a name="blog4.1"></a>
## 4.1 count

> S.count(sub[, start[, end]])

返回字符串S中子串sub出现的次数，可以指定从哪里开始计算(start)以及计算到哪里结束(end)，索引从0开始计算，不包括end边界。

例如：
```
>>> print('xyabxyxy'.count('xy'))
3

# 次数2，因为从index=1算起，即从'y'开始查找，查找的范围为'yabxyxy'
>>> print('xyabxyxy'.count('xy',1))
2

# 次数1，因为不包括end，所以查找的范围为'yabxyx'
>>> print('xyabxyxy'.count('xy',1,7))
1

# 次数2，因为查找的范围为'yabxyxy'
>>> print('xyabxyxy'.count('xy',1,8))
2
```

<a name="blog4.2"></a>
## 4.2 endswith和startswith

> S.endswith(suffix[, start[, end]])
> S.startswith(prefix[, start[, end]])

`endswith()`检查字符串S是否以suffix结尾，返回布尔值的True和False。suffix可以是一个元组(tuple)。可以指定起始start和结尾end的搜索边界。

同理`startswith()`用来判断字符串S是否是以prefix开头。

例如：

1. suffix是普通的字符串时。
```
>>> print('abcxyz'.endswith('xyz'))
True
　
# False，因为搜索范围为'yz'
>>> print('abcxyz'.endswith('xyz',4))
False
　
# False，因为搜索范围为'abcxy'
>>> print('abcxyz'.endswith('xyz',0,5))
False
>>> print('abcxyz'.endswith('xyz',0,6))
True
```

2. suffix是元组(tuple)时，只要tuple中任意一个元素满足endswith的条件，就返回True。
```
# tuple中的'xyz'满足条件
>>> print('abcxyz'.endswith(('ab','xyz')))
True
　
# tuple中'ab'和'xy'都不满足条件
>>> print('abcxyz'.endswith(('ab','xy')))
False
　
# tuple中的'z'满足条件
>>> print('abcxyz'.endswith(('ab','xy','z')))
True
```

<a name="blog4.3"></a>
## 4.3 find,rfind和index,rindex

> S.find(sub[, start[, end]])
> S.rfind(sub[, start[, end]])¶
> S.index(sub[, start[, end]])
> S.rindex(sub[, start[, end]])

find()搜索字符串S中是否包含子串sub，如果包含，则返回sub的索引位置，否则返回"-1"。可以指定起始start和结束end的搜索位置。

index()和find()一样，唯一不同点在于当找不到子串时，抛出`ValueError`错误。

rfind()则是返回搜索到的最右边子串的位置，如果只搜索到一个或没有搜索到子串，则和find()是等价的。

同理rindex()。

例如：
```
>>> print('abcxyzXY'.find('xy'))
3
>>> print('abcxyzXY'.find('Xy'))
-1
>>> print('abcxyzXY'.find('xy',4))
-1

>>> print('xyzabcabc'.find('bc'))
4
>>> print('xyzabcabc'.rfind('bc'))
7

>>> print('xyzabcabc'.rindex('bcd'))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError: substring not found
```

可以使用`in`操作符来判断字符串S是否包含子串sub，它返回的不是索引位置，而是布尔值。
```
>>> 'xy' in 'abxycd'
True
>>> 'xyz' in 'abxycd'
False
```

<a name="blog5"></a>
# 5.替换

<a name="blog5.1"></a>
## 5.1 replace

>S.replace(old, new[, count])

将字符串中的子串old替换为new字符串，如果给定count，则表示只替换前count个old子串。如果S中搜索不到子串old，则无法替换，直接返回字符串S(不创建新字符串对象)。
```
>>> print('abcxyzoxy'.replace('xy','XY'))
abcXYzoXY
>>> print('abcxyzoxy'.replace('xy','XY',1))
abcXYzoxy
>>> print('abcxyzoxy'.replace('mn','XY',1))
abcxyzoxy
```

<a name="blog5.2"></a>
## 5.2 expandtabs

> S.expandtabs(N)

将字符串S中的`\t`替换为一定数量的空格。默认N=8。

注意，`expandtabs(8)`不是将`\t`直接替换为8个空格。例如`'xyz\tab'.expandtabs()`会将`\t`替换为5个空格，因为"xyz"占用了3个字符位。

所以，在替换"\t"为空格时，会减掉"\t"前面的字符数量。如果"\t"的前面正好没有字符，则直接将"\t"替换为N个空格。

另外，它不会替换换行符(`\n`或`\r`)。

例如：
```
>>> '01\t012\t0123\t01234'.expandtabs(4)
'01  012 0123    01234'          --> 2个空格、1个空格、4个空格

>>> '01\t012\t0123\t01234'.expandtabs(8)
'01      012     0123    01234'  --> 6个空格、5个空格、4个空格

>>> '01\t012\t0123\t01234'.expandtabs(7)
'01     012    0123   01234'     --> 5个空格、4个空格、3个空格

>>> print('012\t0123\n01234'.expandtabs(7))
012    0123        --> 4个空格
01234
```

<a name="blog5.3"></a>
## 5.3 translate和maketrans

>S.translate(table)
>static str.maketrans(x[, y[, z]])

`str.maketrans()`生成一个字符一 一映射的table，然后使用`translate(table)`对字符串S中的每个字符进行映射。

如果你熟悉Linux，就知道tr命令，translate()实现的功能和tr是类似的。

例如，现在想要对"I love Fairy"做一个简单的加密，将里面部分字符都替换为数字，这样别人就不知道转换后的这句话是什么意思。

```
>>> in_str='abcxyz'
>>> out_str='123456'

# maketrans()生成映射表
>>> map_table=str.maketrans(in_str,out_str)

# 使用translate()进行映射
>>> my_love='I love Fairy'
>>> result=my_love.translate(map_table)
>>> print(result)
I love F1ir5
```

注意，`maketrans(x[, y[, z]])`中的x和y都是字符串，且长度必须相等。

如果`maketrans(x[, y[, z]])`给定了第三个参数z，则这个参数字符串中的每个字符都会被映射为None。

例如，不替换"a"和"y"。
```
>>> in_str='abcxyz'
>>> out_str='123456'
>>> map_table=str.maketrans(in_str,out_str,'ay')
>>> my_love='I love Fairy'
>>> result=my_love.translate(map_table)
>>> print(result)
I love Fir
```

<a name="blog6"></a>
# 6.分割

<a name="blog6.1"></a>
## 6.1 partition和rpartition

>S.partition(sep)
>S.rpartition(sep)

搜索字符串S中的子串sep，并从sep处对S进行分割，最后返回一个包含3元素的元组：sep左边的部分是元组的第一个元素，sep自身是元组的二个元素，sep右边是元组的第三个元素。

`partition(sep)`从左边第一个sep进行分割，`rpartition(sep)`从右边第一个sep进行分割。

如果搜索不到sep，则返回的3元素元组中，有两个元素为空。partition()是后两个元素为空，rpartition()是前两个元素为空。

例如：
```
# 只搜索到一个sep时，两者结果相同
>>> print('abcxyzopq'.partition('xy'))
('abc', 'xy', 'zopq')
>>> print('abcxyzopq'.rpartition('xy'))
('abc', 'xy', 'zopq')

# 搜索到多个sep时，分别从左第一个、右第一个sep分割
>>> print('abcxyzxyopq'.partition('xy'))
('abc', 'xy', 'zxyopq')
>>> print('abcxyzxyopq'.rpartition('xy'))
('abcxyz', 'xy', 'opq')

# 搜索不到sep
>>> print('abcxyzxyopq'.partition('xyc'))
('abcxyzxyopq', '', '')
>>> print('abcxyzxyopq'.rpartition('xyc'))
('', '', 'abcxyzxyopq')
```

<a name="blog6.2"></a>
## 6.2 split、rsplit和splitlines

>S.split(sep=None, maxsplit=-1)
>S.rsplit(sep=None, maxsplit=-1)
>S.splitlines([keepends=True])

都是用来分割字符串，并生成一个列表。

`split()`根据sep对S进行分割，maxsplit用于指定分割次数，如果不指定maxsplit或者给定值为"-1"，则会从左向右搜索并且每遇到sep一次就分割直到搜索完字符串。如果不指定sep或者指定为None，则改变分割算法：以空格为分隔符，且将连续的空白压缩为一个空格。

`rsplit()`和`split()`是一样的，只不过是从右边向左边搜索。

splitlines()用来专门用来分割换行符。虽然它有点像`split('\n')`或`split('\r\n')`，但它们有些区别，见下文解释。

首先是split()的示例分析(`rsplit()`示例略)。

```
# sep为单个字符时
>>> '1,2,3'.split(',')
['1', '2', '3']

>>> '1,2,3'.split(',',1)
['1', '2,3']    # 只分割了一次

>>> '1,2,,3'.split(',')
['1', '2', '', '3']  # 不会压缩连续的分隔符

>>> '<hello><><world>'.split('<')
['', 'hello>', '>', 'world>']

# sep为多个字符时
>>> '<hello><><world>'.split('<>')
['<hello>', '<world>']

# 不指定sep时
>>> '1 2 3'.split()
['1', '2', '3']

>>> '1 2 3'.split(maxsplit=1)
['1', '2 3']

>>> '   1    2   3   '.split()
['1', '2', '3']

>>> '   1    2   3  \n'.split()
['1', '2', '3']

# 显式指定sep为空格、制表符、换行符时
>>> ' 1  2  3  \n'.split(' ')
['', '1', '', '2', '', '3', '', '\n']

>>> ' 1  2  3  \n'.split('\t')
[' 1  2  3  \n']

>>> ' 1 2\n3 \n'.split('\n')
[' 1 2', '3 ', '']  # 注意列表的最后一项''

>>> ''.split('\n')
['']
```

再是splitlines()的示例分析。

`splitlines()`中可以指定各种换行符，常见的是`\n`、`\r`、`\r\n`。如果指定keepends为True，则保留所有的换行符。
```
>>> 'ab c\n\nde fg\rkl\r\n'.splitlines()
['ab c', '', 'de fg', 'kl']

>>> 'ab c\n\nde fg\rkl\r\n'.splitlines(keepends=True)
['ab c\n', '\n', 'de fg\r', 'kl\r\n']
```

将split()和splitlines()相比较一下：
```
#### split()
>>> ''.split('\n')
['']            # 因为没换行符可分割

>>> 'One line\n'.split('\n')
['One line', '']

#### splitlines()
>>> "".splitlines()
[]              # 因为没有换行符可分割

>>> 'Two lines\n'.splitlines()
['Two lines']
```

<a name="blog7"></a>
# 7.join

>S.join(iterable)

将可迭代对象(iterable)中的元素使用S连接起来。注意，iterable中必须全部是字符串类型，否则报错。

如果你还是python的初学者，还不知道iterable是什么，却想来看看join的具体语法，那么你可以暂时将它理解为：字符串string、列表list、元组tuple、字典dict、集合set。

例如：

1. 字符串
```
>>> L='python'
>>> '_'.join(L)
'p_y_t_h_o_n'
```
2. 元组
```
>>> L1=('1','2','3')
>>> '_'.join(L1)
'1_2_3'
```
3. 集合。注意，集合无序。
```
>>> L2={'p','y','t','h','o','n'}
>>> '_'.join(L2)
'n_o_p_h_y_t'
```
4. 列表
```
>>> L2=['py','th','o','n']
>>> '_'.join(L2)
'py_th_o_n'
```
5. 字典
```
>>> L3={'name':"malongshuai",'gender':'male','from':'China','age':18}
>>> '_'.join(L3)
'name_gender_from_age'
```
6. iterable参与迭代的每个元素必须是字符串类型，不能包含数字或其他类型。
```
>>> L1=(1,2,3)
>>> '_'.join(L1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: sequence item 0: expected str instance, int found
```
以下两种也不能join。
```
>>> L1=('ab',2)
>>> L2=('AB',{'a','cd'})
```

将join()时的元素连接符指定为空时，则会将可迭代对象的每个元素组成一个连接起来的字符串。有时候，这是很有用的。
```
>>> L=['a','b','c','d']
>>> ''.join(L)
'abcd'
```

<a name="blog8"></a>
# 8.修剪：strip、lstrip和rstrip

>S.strip([chars])
>S.lstrip([chars])
>S.rstrip([chars])

分别是移除左右两边、左边、右边的字符char。如果不指定chars或者指定为`None`，则默认移除空白(空格、制表符、换行符)。

唯一需要注意的是，chars可以是多个字符序列。在移除时，只要是这个序列中的字符，都会被移除。

例如：

1. 移除单个字符或空白。
```
>>> '   spacious   '.lstrip()
'spacious   '
　
>>> '   spacious   '.rstrip()
'   spacious'
　
>>> 'spacious   '.lstrip('s')
'pacious   '
　
>>> 'spacious'.rstrip('s')
'spaciou'
```
2.移除字符序列中的字符。
```
>>> print('www.example.com'.lstrip('cmowz.'))
example.com
>>> print('wwwz.example.com'.lstrip('cmowz.'))
example.com
>>> print('wwaw.example.com'.lstrip('cmowz.'))
aw.example.com
>>> print('www.example.com'.strip('cmowz.'))
'example'
```

由于`www.example.com`的前4个字符都是字符序列`cmowz.`中的字符，所以都被移除，而第五个字符e不在字符序列中，所以修剪到此结束。同理`wwwz.example.com`。

`wwaw.example.com`中第3个字符a不是字符序列中的字符，所以修剪到此结束。



