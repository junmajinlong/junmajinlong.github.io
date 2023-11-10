---
title: seq命令
p: shell/seq.md
date: 2023-07-11 08:20:45
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# seq命令

seq命令用于输出数字序列。支持正数序列、负数序列、小数序列。

```bash
seq [OPTION]... LAST                  # 指定输出的结尾数字，初始值和步长默认都为1
seq [OPTION]... FIRST LAST            # 指定开始和结尾数字，步长默认为1
seq [OPTION]... FIRST INCREMENT LAST  # 指定开始值、步长和结尾值

OPTION：
-s：指定分隔符，默认是\n。
-w：使用0填充左边达到数字的最大宽度。
```

使用示例：

```bash
[root@xuexi ~]# seq 5
1
2
3
4
5

# 指定使用减号作为分隔符
[root@xuexi ~]# seq -s "-" 5
1-2-3-4-5

# 指定开始和结尾
[root@xuexi ~]# seq -s "-" 5 10
5-6-7-8-9-10

# 指定开始、步长和结尾
[root@xuexi ~]# seq -s "-" 5 2 20
5-7-9-11-13-15-17-19

# 小数序列
[root@xuexi ~]# seq 3.1 2 10
3.1
5.1
7.1
9.1

[root@xuexi ~]# seq 3.1 2.3 10
3.1
5.4
7.7
10.0

# 左边使用0填充
[root@xuexi ~]# seq -w 99 100 1200
0099
0199
0299
0399
0499
0599
0699
0799
0899
0999
1099
1199
```