---
title: uniq命令
p: shell/uniq.md
date: 2023-07-11 08:20:42
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# uniq命令

uniq用于去重，但注意，不相邻的行不算重复值。
```
uniq [OPTION]... [INPUT [OUTPUT]]

选项说明：
-c：统计出现的次数（count）。
-d：只显示被计算为重复的**行**。
-D：显示所有被计算为重复的**行**。
-u：显示唯一值，即没有重复值的**行**。
-i：忽略大小写。
-z：在末尾使用\0，而不是换行符。
-f：跳过多少个字段(field)开始比较重复值。
-s：跳过多少个字符开始比较重复值。
-w：比较重复值时每行比较的最大长度。即对每行多长的字符进行比较。
```

示例：

```bash
[root@xuexi tmp]# cat uniq.txt
111
223
56
111
111
567
223
```

下面的命令删除了相邻的重复行，但是第一行111没有删除。

```bash
[root@xuexi tmp]# uniq uniq.txt
111
223
56
111   # 删除了重复的111
567
223
```

排序后去重。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq
111
223
56
567
```

使用`-d`显示重复的行。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq  -d
111
223
```

使用`-D`显示所有重复过的行。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq  -D
111
111
111
223
223
```

使用`-u`显示唯一行。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq  -u
56
567
```

使用`-c`统计哪些记录出现的次数。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq  -c  
      3 111
      2 223
      1 56
      1 567
```

使用`-d -c`统计重复行出现的次数。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq  -d -c
      3 111
      2 223
```

`-c`不能和`-D`一起使用。结果说显示所有重复行再统计重复次数是毫无意义的行为。

```bash
[root@xuexi tmp]# sort uniq.txt | uniq  -D -c
uniq: printing all duplicated lines and repeat counts is meaningless
Try `uniq --help' for more information.
```