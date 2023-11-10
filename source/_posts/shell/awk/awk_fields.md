---
title: 精通awk系列(8)：awk划分字段的3种方式
p: shell/awk/awk_fields.md
date: 2019-11-23 10:37:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------



# 详细分析awk字段分割

awk读取每一条记录之后，会将其赋值给`$0`，同时还会对这条记录按照**预定义变量FS**划分字段，将划分好的各个字段分别赋值给`$1 $2 $3 $4...$N`，同时将划分的字段数量赋值给**预定义变量NF**。

## 引用字段的方式

`$N`引用字段：  

- `N=0`：即`$0`，引用记录本身  
- `0<N<=NF`：引用对应字段  
- `N>NF`：表示引用不存在的字段，返回空字符串  
- `N<0`：报错  

可使用变量或计算的方式指定要获取的字段序号。

```
awk '{n = 5;print $n}' a.txt
awk '{print $(2+2)}' a.txt   # 括号必不可少，用于改变优先级
awk '{print $(NF-3)}' a.txt
```

## 分割字段的方式

读取record之后，将使用预定义变量FS、FIELDWIDTHS或FPAT中的一种来分割字段。分割完成之后，再进入main代码段(所以，在main中设置FS对本次已经读取的record是没有影响的，但会影响下次读取)。

<a name="FS"></a>

### 划分字段方式(一)：FS或-F

`FS`或者`-F`：字段分隔符  

- FS为单个字符时，该字符即为字段分隔符  
- FS为多个字符时，则采用正则表达式模式作为字段分隔符  
- 特殊的，也是FS默认的情况，**FS为单个空格时，将以连续的空白（空格、制表符、换行符）作为字段分隔符**  
- 特殊的，FS为空字符串""时，将对每个字符都进行分隔，即每个字符都作为一个字段  
- 设置预定义变量IGNORECASE为非零值，正则匹配时表示忽略大小写(只影响正则，所以FS为单字时无影响)  
- 如果record中无法找到FS指定的分隔符(例如将FS设置为"\n")，则整个记录作为一个字段，即`$1`和`$0`相等  

```
# 字段分隔符指定为单个字符
awk -F":" '{print $1}' /etc/passwd
awk 'BEGIN{FS=":"}{print $1}' /etc/passwd

# 字段分隔符指定为正则表达式
awk 'BEGIN{FS=" +|@"}{print $1,$2,$3,$4,$5,$6}' a.txt
```

<a name="FIELDWIDTHS"></a>

### 划分字段方式(二)：FIELDWIDTHS

指定预定义变量FIELDWIDTHS按字符宽度分割字段，这是gawk提供的高级功能。在处理某字段缺失时非常好用。

用法：  

![](/img/shell/awk/733013-20191123154647498-149120577.jpg)

示例1：

```
# 没取完的字符串DDD被丢弃，且NF=3
$ awk 'BEGIN{FIELDWIDTHS="2 3 2"}{print $1,$2,$3,$4}' <<<"AABBBCCDDDD"
AA BBB CC 

# 字符串不够长度时无视
$ awk 'BEGIN{FIELDWIDTHS="2 3 2 100"}{print $1,$2,$3,$4"-"}' <<<"AABBBCCDDDD"
AA BBB CC DDDD-

# *号取剩余所有，NF=3
$ awk 'BEGIN{FIELDWIDTHS="2 3 *"}{print $1,$2,$3}' <<<"AABBBCCDDDD"      
AA BBB CCDDDD

# 字段数多了，则取完字符串即可，NF=2
$ awk 'BEGIN{FIELDWIDTHS="2 30 *"}{print $1,$2,NF}' <<<"AABBBCCDDDD"  
AA BBBCCDDDD 2
```

示例2：处理某些字段缺失的数据。

如果按照常规的FS进行字段分割，则对于缺失字段的行和没有缺失字段的行很难统一处理，但使用FIELDWIDTHS则非常方便。

假设a.txt文本内容如下：

```
ID  name    gender  age  email          phone
1   Bob     male    28   abc@qq.com     18023394012
2   Alice   female  24   def@gmail.com  18084925203
3   Tony    male    21   aaa@163.com    17048792503
4   Kevin   male    21   bbb@189.com    17023929033
5   Alex    male    18                  18185904230
6   Andy    female  22   ddd@139.com    18923902352
7   Jerry   female  25   exdsa@189.com  18785234906
8   Peter   male    20   bax@qq.com     17729348758
9   Steven  female  23   bc@sohu.com    15947893212
10  Bruce   female  27   bcbd@139.com   13942943905
```

因为email字段有的是空字段，所以直接用FS划分字段不便处理。可使用FIELDWIDTHS。

```
# 字段1：4字符
# 字段2：8字符
# 字段3：8字符
# 字段4：2字符
# 字段5：先跳过3字符，再读13字符，该字段13字符
# 字段6：先跳过2字符，再读11字符，该字段11字符
awk '
BEGIN{FIELDWIDTHS="4 8 8 2 3:13 2:11"}
NR>1{
    print "<"$1">","<"$2">","<"$3">","<"$4">","<"$5">","<"$6">"
}' a.txt

# 如果email为空，则输出它
awk '
BEGIN{FIELDWIDTHS="4 8 8 2 3:13 2:11"}
NR>1{
    if($5 ~ /^ +$/){print $0}
}' a.txt
```
<a name="FPAT"></a>

### 划分字段方式(三)：FPAT

FS是指定字段分隔符，来取得除分隔符外的部分作为字段。

FPAT是取得匹配的字符部分作为字段。它是gawk提供的一个高级功能。

FPAT根据指定的正则来全局匹配record，然后将所有匹配成功的部分组成`$1、$2...`，不会修改`$0`。

- `awk 'BEGIN{FPAT="[0-9]+"}{print $3"-"}' a.txt`  
- 之后再设置FS或FPAT，该变量将失效  

FPAT常用于字段中包含了字段分隔符的场景。例如，CSV文件中的一行数据如下：

```
Robbins,Arnold,"1234 A Pretty Street, NE",MyTown,MyState,12345-6789,USA
```

其中逗号分隔每个字段，但双引号包围的是一个字段整体，即使其中有逗号。

这时使用FPAT来划分各字段比使用FS要方便的多。

```
echo 'Robbins,Arnold,"1234 A Pretty Street, NE",MyTown,MyState,12345-6789,USA' |\
awk '
	BEGIN{FPAT="[^,]*|(\"[^\"]*\")"}
	{
        for (i=1;i<NF;i++){
            print "<"$i">"
        }
	}
'
```

最后，patsplit()函数和FPAT的功能一样。

## 检查字段划分的方式

有FS、FIELDWIDTHS、FPAT三种获取字段的方式，可使用`PROCINFO`数组来确定本次使用何种方式获得字段。

PROCINFO是一个数组，记录了awk进程工作时的状态信息。

如果： 

- `PROCINFO["FS"]=="FS"`，表示使用FS分割获取字段  
- `PROCINFO["FPAT"]=="FPAT"`，表示使用FPAT匹配获取字段  
- `PROCINFO["FIELDWIDTHS"]=="FIELDWIDTHS"`，表示使用FIELDWIDTHS分割获取字段  

例如：  

```
if(PROCINFO["FS"]=="FS"){
    ...FS spliting...
} else if(PROCINFO["FPAT"]=="FPAT"){
    ...FPAT spliting...
} else if(PROCINFO["FIELDWIDTHS"]=="FIELDWIDTHS"){
    ...FIELDWIDTHS spliting...
}
```
