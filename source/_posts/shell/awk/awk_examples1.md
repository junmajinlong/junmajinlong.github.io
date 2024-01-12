---
title: 精通awk系列(10)：awk筛选行和处理字段的示例
p: shell/awk/awk_examples1.md
date: 2019-11-23 10:37:38
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# awk数据筛选示例

## 筛选行

```
# 1.根据行号筛选
awk 'NR==2' a.txt   # 筛选出第二行
awk 'NR>=2' a.txt   # 输出第2行和之后的行

# 2.根据正则表达式筛选整行
awk '/qq.com/' a.txt       # 输出带有qq.com的行
awk '$0 ~ /qq.com/' a.txt  # 等价于上面命令
awk '/^[^@]+$/' a.txt      # 输出不包含@符号的行
awk '!/@/' a.txt           # 输出不包含@符号的行

# 3.根据字段来筛选行
awk '($4+0) > 24{print $0}' a.txt  # 输出第4字段大于24的行
awk '$5 ~ /qq.com/' a.txt   # 输出第5字段包含qq.com的行

# 4.将多个筛选条件结合起来进行筛选
awk 'NR>=2 && NR<=7' a.txt 
awk '$3=="male" && $6 ~ /^170/' a.txt       
awk '$3=="male" || $6 ~ /^170/' a.txt  

# 5.按照范围进行筛选 flip flop
# pattern1,pattern2{action}
awk 'NR==2,NR==7' a.txt        # 输出第2到第7行
awk 'NR==2,$6 ~ /^170/' a.txt
```

## 处理字段

修改字段时，一定要注意，可能带来的联动效应：即使用OFS重建$0。

```
awk 'NR>1{$4=$4+5;print $0}' a.txt
awk 'BEGIN{OFS="-"}NR>1{$4=$4+5;print $0}' a.txt
awk 'NR>1{$6=$6"*";print $0}' a.txt
```

## awk运维面试试题

从ifconfig命令的结果中筛选出除了lo网卡外的所有IPv4地址。

```
# 1.法一：多条件筛选
ifconfig | awk '/inet / && !($2 ~ /^127/){print $2}'

# 2.法二：按段落读取，然后取IPv4字段
ifconfig | awk 'BEGIN{RS=""}!/lo/{print $6}'

# 3.法三：按段落读取，每行1字段，然后取IPv4字段
ifconfig | awk 'BEGIN{RS="";FS="\n"}!/lo/{$0=$2;FS=" ";$0=$0;print $2;FS="\n"}'
```

