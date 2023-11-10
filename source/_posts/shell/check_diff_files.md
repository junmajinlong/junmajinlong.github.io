---
title: 批量比较多个文件的内容是否相同
p: shell/check_diff_files.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# 批量比较多个文件的内容是否相同

要比较两个文件的内容是否完全一致，可以简单地使用diff命令。例如：

```
diff file1 file2 &>/dev/null;echo $?
```

但是diff命令只能给定两个文件参数，因此无法一次性比较多个文件(目录也被当作文件)，而且diff比较非文本类文件或者极大的文件时效率极低。

这时可以使用md5sum来实现，相比diff的逐行比较，md5sum的速度快的多的多。

md5sum的使用方法见：[Linux中文件MD5校验](https://www.junmajinlong.com/linux/md5sum)。

但md5sum只能通过查看md5值来间接比较文件是否相同，要实现批量自动比较，则需要写成循环。脚本如下：

```
#!/bin/bash

# filename: md5.sh
# Usage: $0 file1 file2 file3 ...

IFS=$'\n'
declare -A md5_array

# If use while read loop, the array in while statement will
# auto set to null after the loop, so i use for statement
# instead the while, and so, i modify the variable IFS to
# $'\n'.

# md5sum format: MD5  /path/to/file
# such as:80748c3a55b726226ad51a4bafa1c4aa /etc/fstab
for line in `md5sum "$@"`
do
    index=${line%% *}
    file=${line##* }
    md5_array[$index]="$file ${md5_array[$index]}"
done

# Traverse the md5_array
for i in ${!md5_array[@]}
do
    echo -e "the same file with md5: $i\n--------------\n`echo ${md5_array[$i]}|tr ' ' '\n'`\n"
done 
```

为了测试该脚本，先复制几个文件，并修改其中几个文件的内容，例如：

```
[root@xuexi ~]# for i in `seq -s' ' 6`;do cp -a /etc/fstab /tmp/fs$i;done
[root@xuexi ~]# echo ha >>/tmp/fs4
[root@xuexi ~]# echo haha >>/tmp/fs5
```

现在，/tmp目录下有6个文件fs1、fs2、fs3、fs4、fs5和fs6，其中fs4和fs5被修改，剩余4个文件内容完全相同。

```
[root@xuexi tmp]# ./md5.sh /tmp/fs[1-6]
the same file with md5: a612cd5d162e4620b442b0ff3474bf98
--------------------------
/tmp/fs6
/tmp/fs3
/tmp/fs2
/tmp/fs1

the same file with md5: 80748c3a55b726226ad51a4bafa1c4aa
--------------------------
/tmp/fs4

the same file with md5: 30dd43dba10521c1e94267bbd117877b
--------------------------
/tmp/fs5
```

更具通用性地比较方法：比较多个目录下的同名文件。

```
[root@xuexi tmp]# find /tmp -type f -name "fs[0-9]" -print0 | xargs -0 ./md5.sh  
the same file with md5:a612cd5d162e4620b442b0ff3474bf98
--------------------------
/tmp/fs6
/tmp/fs3
/tmp/fs2
/tmp/fs1

the same file with md5:80748c3a55b726226ad51a4bafa1c4aa
--------------------------
/tmp/fs4

the same file with md5:30dd43dba10521c1e94267bbd117877b
--------------------------
/tmp/fs5
```

脚本说明：

(1).md5sum计算的结果格式为"MD5 /path/to/file"，因此要在结果中既输出MD5值，又输出相同MD5对应的文件，考虑使用数组。

(2).一开始的时候我使用while循环，从标准输入中读取每个文件md5sum的结果。语句如下：

```
md5sum "$@" | while read index file;do
    md5_array[$index]="$file ${md5_array[$index]}"
done
```

但由于管道使得while语句在子shell中执行，于是while中赋值的数组md5_array在循环结束时将失效。所以可改写为：

```
while read index file;do
    md5_array[$index]="$file ${md5_array[$index]}"
done <<<"$(md5sum "$@")"
```

不过我最终还是使用了更繁琐的for循环：

```
IFS=$'\n'
for line in `md5sum "$@"`
do
    index=${line%% *}
    file=${line##* }
    md5_array[$index]="$file ${md5_array[$index]}"
done
```

但md5sum的每行结果中有两列，而for循环采用默认的IFS会将这两列分割为两个值，因此还修改了IFS变量的值为`$'\n'`，使得一行赋值一次变量。

(3).index和file变量是为了将md5sum的每一行结果拆分成两个变量，MD5部分作为数组的index，file作为数组变量值的一部分。因此，数组赋值语句为：

```
md5_array[$index]="$file ${md5_array[$index]}"
```

(4).数组赋值完成后，开始遍历数组。遍历的方法有多种。我采用的是遍历数组的index列表，即每行的MD5值。

```
# Traverse the md5_array
for i in ${!md5_array[@]}
do
    echo -e "the same file with md5: $i\n--------------\n`echo ${md5_array[$i]}|tr ' ' '\n'`\n"
done  
```