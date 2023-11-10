---
title: 精通awk系列(3)：铺垫知识：读取文件的几种方式
p: shell/awk/file_read.md
date: 2019-11-23 10:37:32
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# 读取文件的几种方式

读取文件有如下几种常见的方式：  

![](/img/shell/awk/733013-20191123144705344-967361188.jpg)

下面使用Shell的read命令来演示前4种读取文件的方式(第五种按字节数读取的方式read不支持)。

## 按字符数量读取

read的-n选项和-N选项可以指定一次性读取多少个字符。

```
# 只读一个字符
read -n 1 data <a.txt

# 读100个字符，但如果不足100字符时遇到换行符则停止读取
read -n 100 data < a.txt

# 强制读取100字符，遇到换行符也不停止
read -N 100 data < a.txt
```

如果按照字符数量读取，直到把文件读完，则使用while循环，且将文件放在while结构的后面，而不能放在while循环的条件位置：
```
# 正确
while read -N 3 data;do
  echo "$data"
done <a.txt


# 错误
while read -N 3 data < a.txt;do
  echo "$data"
done
```

## 按分隔符读取

read命令的-d选项可以指定读取文件时的分隔符。

```
# 一直读取，直到遇到字符m才停止，并将读取的数据保存到data变量中
read -d "m" data <a.txt
```
如果要按分隔符读取并读完整个文件，则使用while循环：
```
while read -d "m" data ;do
  echo "$data"
done <a.txt
```

## 按行读取

read默认情况下就是按行读取的，一次读取一行。
```
# 从a.txt中读取第一行保存到变量data中
read line <a.txt
```

如果要求按行读取完整个文件，则使用while循环：
```
while read line;do
  echo "$line"
done <a.txt
```

## 一次性读整个文件

要一次性读取完整个文件，有两种方式：  
- 按照字符数量读取，且指定的字符数要大于文件的总大小  
- 按分隔符读取，且指定的分隔符是文件中不存在的字符，这样的话会一直读取，因为找不到分隔符而读完整个文件  

```
# 指定超出文件大小的字符数量
read -N 1000000 data <a.txt
echo "$data"

# 指定文件中不存在的字符作为分隔符
read -d "_" data <a.txt
echo "$data"
```

