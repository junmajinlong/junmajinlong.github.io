---
title: 精通awk系列(25)：awk选项和内置变量的含义
p: shell/awk/awk_option_builtin_variable.md
date: 2020-04-12 15:45:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# awk选项、内置变量

## 选项

```
-e program-text
--source program-text
指定awk程序表达式，可结合-f选项同时使用
在使用了-f选项后，如果不使用-e，awk program是不会执行的，它会被当作ARGV的一个参数

-f program-file
--file program-file
从文件中读取awk源代码来执行，可指定多个-f选项

-F fs
--field-separator fs
指定输入字段分隔符(FS预定义变量也可设置)

-n
--non-decimal-data
识别文件输入中的8进制数(0开头)和16进制数(0x开头)
echo '030' | awk -n '{print $1+0}'

-o [filename]
格式化awk代码。
不指定filename时，则默认保存到awkprof.out
指定为`-`时，表示输出到标准输出

-v var=val
--assign var=val
在BEGIN之前，声明并赋值变量var，变量可在BEGIN中使用
```

## 预定义变量

预定义变量分为两类：控制awk工作的变量和携带信息的变量。

**第一类：控制AWK工作的预定义变量**  

- `RS`：输入记录分隔符，默认为换行符`\n`，参考[RS](#RS)  
- `IGNORECASE`：默认值为0，表示所有的正则匹配不忽略大小写。设置为非0值（例如1），之后的匹配将忽略大小写。例如在BEGIN块中将其设置为1，将使FS、RS都以忽略大小写的方式分隔字段或分隔record  
- `FS`：读取记录后，划分为字段的字段分隔符。参考[FS](#FS)  
- `FIELDWIDTHS`：以指定宽度切割字段而非按照FS。参考[FIELDWIDTHS](#FIELDWIDTHS)  
- `FPAT`：以正则匹配匹配到的结果作为字段，而非按照FS划分。参考[FPAT](#FPAT)  
- `OFS`：print命令输出各字段列表时的输出字段分隔符，默认为空格" "  
- `ORS`：print命令输出数据时在尾部自动添加的记录分隔符，默认为换行符`\n`  
- `CONVFMT`：在awk中数值隐式转换为字符串时，将根据CONVFMT的格式按照sprintf()的方式自动转换为字符串。默认值为"%.6g  
- `OFMT`：在print中，数值会根据OFMT的格式按照sprintf()的方式自动转换为字符串。默认值为"%.6g  

**第二类：携带信息的预定义变量**  

- `ARGC`和`ARGV`：awk命令行参数的数量、命令参数的数组。参考[ARGC和ARGV](#ARGC_ARGV)  
- `ARGIND`：awk当前正在处理的文件在ARGV中的索引位置。所以，如果awk正在处理命令行参数中的某文件，则`ARGV[ARGIND] == FILENAME`为真  
- `FILENAME`：awk当前正在处理的文件(命令行中指定的文件)，所以在BEGIN中该变量值为空  
- `ENVIRON`：保存了Shell的环境变量的数组。例如`ENVIRON["HOME"]`将返回当前用户的家目录  
- `NR`:当前已读总记录数，多个文件从不会重置为0，所以它是一直叠加的  
   - 可以直接修改NR，下次读取记录时将在此修改值上自增  
- `FNR`:当前正在读取文件的第几条记录，每次打开新文件会重置为0  
   - 可以直接修改FNR，下次读取记录时将在此修改值上自增  
- `NF`：当前记录的字段数，参考[NF](#NF)  
- `RT`：在读取记录时真正的记录分隔符，参考[RT](#RT)  
- `RLENGTH`：match()函数正则匹配成功时，所匹配到的字符串长度，如果匹配失败，该变量值为-1  
- `RSTART`：match()函数匹配成功时，其首字符的索引位置，如果匹配失败，该变量值为0  
- `SUBSEP`：`arr[x,y]`中下标分隔符构建成索引时对应的字符，默认值为`\034`，是一个不太可能出现在字符串中的不可打印字符。参考[复杂数组](#SUBSEP)  

