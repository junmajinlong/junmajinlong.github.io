---
title: comm命令求出文件的交集、差集
p: shell/comm.md
date: 2020-05-13 14:17:18
tags: Shell
categories: Shell
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# comm命令求出文件的交集、差集

A(1,2,3)和B(3,4,5)，A和B的交集是3，A对B的差集是1和2，B对A的差集是4和5，A和B求差的结果是1、2、4、5。

在Linux中可以使用comm命令求出这些集。

```
[root@xuexi tmp]# cat <<eof>set1.txt
> orange
> gold
> apple
> sliver
> steel
> iron
> eof
[root@xuexi tmp]# cat <<eof>set2.txt
> orange
> gold
> cookiee
> carrot
> eof
```

使用comm命令。

```
[root@xuexi tmp]# comm set1.txt set2.txt
apple
                orange
comm: file 1 is not in sorted order
comm: file 2 is not in sorted order
                gold
        cookiee
        carrot
silver
steel
iron
```

提示没有排序，所以comm必须要保证比较的文件是有序的。

```
[root@xuexi tmp]# sort set1.txt -o set1.txt;sort set2.txt -o set2.txt
[root@xuexi tmp]# comm set1.txt set2.txt                             
apple
        carrot
        cookiee
                gold
iron
                orange
silver
steel
```

结果中输出了3列，每一列使用制表符`\t`隔开。第一列是set1.txt中有而set2.txt中没有的，第二列则是set2.txt中有而set1.txt中没有的，第三列是set1.txt和set2.txt中都有的。

根据这三列就可以求出交集、差集和求差。

交集就是第三列。使用-1和-2分别删除第一第二列就是第三列的结果。

```
[root@xuexi tmp]# comm set1.txt set2.txt -1 -2
gold
orange
```

A对B的差集就是第一列，B对A的差集就是第二列。

```
[root@xuexi tmp]# comm set1.txt set2.txt  -2 -3  # A对B的差集
apple
iron
silver
steel
[root@xuexi tmp]# comm set1.txt set2.txt  -1 -3   # B对A的差集
carrot
cookiee
```

A和B的求差就是第一列和第二列的组合。

```
[root@xuexi tmp]# comm set1.txt set2.txt  -3  
apple
        carrot
        cookiee
iron
silver
steel
```

但是这样分两列的结果不方便查看，应该进行处理使它们显示在同一列上。

```
[root@xuexi tmp]# comm set1.txt set2.txt  -3 | tr "\t" "\0"
apple
carrot
cookiee
iron
silver
steel
```