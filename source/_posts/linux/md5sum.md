---
title: Linux中文件MD5校验
p: linux/md5sum.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Linux中文件MD5校验

md5sum命令用于生成文件的md5数字摘要，并可以验证文件内容是否发生了改变，间接地还可以检验两个文件内容是否完全相同。因为md5sum是读取文件内容来计算校验码的，因此只能验证文件内容，而无法验证文件属性。

```
[root@xuexi ~]# cp -a /etc/fstab /tmp/fstab

[root@xuexi ~]# cp -a /etc/fstab /tmp/fstab1
```

生成文件的md5值。

```
[root@xuexi ~]# md5sum /tmp/fstab /tmp/fstab1
a612cd5d162e4620b442b0ff3474bf98  /tmp/fstab
a612cd5d162e4620b442b0ff3474bf98  /tmp/fstab1
```

发现这两个文件md5值完全一样，也就说明这两个文件完全相同。

由于生成的md5信息中，每个md5值后都紧跟着对应的文件的路径(可能是相对路径)，于是将生成的md5保存到某个文件中，以后可以使用该文件来检查md5值对应文件内容是否发生了修改。

例如，将上述两个文件的md5信息保存到fs.md5sum中，然后使用`md5sum -c`可以检查源文件是否完整或是否被修改过。这个检查是内容上的，权限和属性等的改变不会影响md5值，所以不会检测出问题。

```
[root@xuexi ~]# md5sum /tmp/fstab /tmp/fstab1 >/tmp/fs.md5sum

[root@xuexi ~]# md5sum -c /tmp/fs.md5sum
/tmp/fstab: OK
/tmp/fstab1: OK
```

修改/tmp/fstab1的内容，然后再检测。

```
[root@xuexi tmp]# echo aaa >>/tmp/fstab1

[root@xuexi tmp]# md5sum -c /tmp/fs.md5sum
/tmp/fstab: OK
/tmp/fstab1: FAILED
md5sum: WARNING: 1 of 2 computed checksums did NOT match
```

当使用了"-c"选项时，还支持以下选项：
```
--quiet：不显示验证结果为OK的记录
--status：完全不显示任何信息，只能通过命令的退出状态码判断验证结果是否有failed。只要有一条failed记录，则状态码为1，否则为0。
```
例如：
```
[root@xuexi tmp]# md5sum --status -c /tmp/fs.md5sum

[root@xuexi tmp]# echo $?
1
```