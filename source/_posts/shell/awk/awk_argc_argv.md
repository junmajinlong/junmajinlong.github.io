---
title: 精通awk系列(21)：awk ARGC和ARGV详解
p: shell/awk/awk_argc_argv.md
date: 2020-04-12 14:40:35
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# awk ARGC和ARGV

预定义变量ARGV是一个数组，包含了所有的命令行参数。该数组使用从0开始的数值作为索引。

预定义变量ARGC初始时是ARGV数组的长度，即命令行参数的数量。

ARGV数组的数量和ARGC的值只有在awk刚开始运行的时候是保证相等的。

```
$ awk -va=1 -F: '
  BEGIN{
    print ARGC;
    for(i in ARGV){
      print "ARGV[" i "]= " ARGV[i]
    }
}' b=3 a.txt b.txt

4
ARGV[0]= awk
ARGV[1]= b=3
ARGV[2]= a.txt
ARGV[3]= b.txt
```

awk读取文件是根据ARGC的值来进行的，有点类似于如下伪代码形式：
```
while(i=1;i<ARGC;i++){
    read from ARGV[i]
}
```

默认情况下，awk在读完ARGV中的一个文件时，会自动从它的下一个元素开始读取，直到读完所有文件。

直接减小ARGC的值，会导致awk不会读取尾部的一些文件。此外，增减ARGC的值，都不会影响ARGV数组，仅仅只是影响awk读取文件的数量。
```
# 不会读取b.txt
awk 'BEGIN{ARGC=2}{print}' a.txt b.txt

# 读完b.txt后自动退出
awk 'BEGIN{ARGC=5}{print}' a.txt b.txt
```

可以将ARGV中某个元素赋值为空字符串""，awk在选择下一个要读取的文件时，会自动忽略ARGV中的空字符串元素。

也可以`delete ARGV[i]`的方式来删除ARGV中的某元素。

用户手动增、删ARGV元素时，不会自动修改ARGC，而awk读取文件时是根据ARGC值来确定的。所以，在增加ARGV元素之后，要手动的去增加ARGC的值。
```
# 不会读取b.txt文件
$ awk 'BEGIN{ARGV[2]="b.txt"}{print}' a.txt

# 会读取b.txt文件
$ awk 'BEGIN{ARGV[2]="b.txt";ARGC++}{print}' a.txt 
```

## 对awk ARGC和ARGV进行操刀

### awk判断命令行中给定文件是否可读

awk命令行中可能会给出一些不存在或无权限或其它原因而无法被awk读取的文件名，这时可以判断并从中剔除掉不可读取的文件。

1. 排除命令行尾部(非选项型参数)的var=val、-、和/dev/stdin这3种特殊情况  
2. 如果不可读，则从ARGV中删除该参数  
3. 剩下的都是可在main代码段正常读取的文件  

```
BEGIN{
  for(i=1;i<ARGC;i++){
    if(ARGV[i] ~ /[a-zA-Z_][a-zA-Z0-9_]*=.*/ \
    || ARGV[i]=="-" || ARGV[i]=="/dev/stdin"){
      continue
    } else if((getline var < ARGV[i]) < 0){
      delete ARGV[i]
    } else{
      close(ARGV[i])
    }
  }
}
```