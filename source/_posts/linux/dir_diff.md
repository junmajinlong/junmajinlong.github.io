---
title: Linux下快速比较两个目录的不同
p: linux/dir_diff.md
date: 2019-07-09 18:20:42
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# Linux下快速比较两个目录的不同

曾多次想要在Linux下比较目录a和目录b中**文件列表**的差别，然后对目录a比目录b中多出的文件、少掉的文件分别做处理。但是，在网上搜索了多次也都没找到能直接处理好的工具。

所以想了很多不少方法，自我感觉都不错，而且网上似乎没有这方面的文章，所以分享出来给大家。如果各位有更好的工具或者方法，盼请留下说明(本文第2部分：`图形化的比较结果`搜集自网上，我也没有在图形化界面下操作的需要，所以没有多做介绍)

以下是本文有些地方涉及到的目录结构。
```
[root@node1 ~]# tree directory1 directory2
directory1
├── 1.png
├── 2.png
└── 3.png
directory2
├── 2.png
├── 3.png
└── 4.png
```

<a name="blog1"></a>

# 1.命令行输出的结果

## 方法一：使用diff
```
diff -r directory1 directory2
```

但是diff会对每个文件中的每一行都做比较，所以文件较多或者文件较大的时候会非常慢。请谨慎使用。

## 方法二：使用diff结合tree
```
diff <(tree -Ci --noreport /mnt/f/自然马) <(tree -Ci --noreport /mnt/i/自然马)
< /mnt/f/自然马
---
> /mnt/i/自然马
87a88
> xyz.avi
488d488
< rsync.txt
534d533
< 542D0.mp4
```
说明：  
1. tree的`-C`选项是输出颜色，如果只是看一下目录的不同，可以使用该选项，但在结合其他命令使用的时候建议不要使用该选项，因为颜色也会转换为对应的编码而输出；  
2. `-i`是不缩进，建议不要省略`-i`，否则diff的结果很难看，也不好继续后续的文件操作；  
3. `--noreport`是不输出报告结果，建议不要省略该选项。  
4. 该方法效率很高。

## 方法三：find结合diff
```
find directory1 -printf "%P\n" | sort > file1
find directory2 -printf "%P\n" | sort | diff file1 -
2d1
< 1.png
4a4
> 4.png
```
说明：  
1. `<`代表的行是directory1中有而directory2没有的文件，`>`则相反，是directory2中有而directory1中没有。  
2. 不要省略`-printf "%P\n"`，此处的%P表示find的结果中去掉前缀路径，详细内容`man find`。例如，`find /root/ -printf "%P\n"`的结果中将显示/root/a/xyz.txt中去掉/root/后的结果：a/xyz.txt。
3. 效率很高，输出也简洁。

如果不想使用`-printf`，那么先进入各目录再find也行。
```
[root@node1 ~]# (cd /root/a;find . | sort >/tmp/file1)       
[root@node1 ~]# (cd /root/b;find . | sort | diff /tmp/file1 -)
2d1
< ./1.png
4a4
> ./4.png
```
上面将命令放进括号中执行是为了在子shell中切换目录，不用影响当前所在目录。


## 方法四：使用rsync
```
rsync -rvn --delete directory1/ directory2 | sed -n '2,/^$/{/^$/!p}'
deleting a/xyz.avi
rsync.txt
新建文件夹/542D0.mp4
```
其中deleting所在的行就是directory2中多出的文件。其他的都是directory中多出的文件。

如果想区分出不同的是目录还是文件。可以加上"-i"选项。
```
rsync -rvn -i --delete directory1/ directory2 | sed -n '2,/^$/{/^$/!p}'
*deleting   a/xyz.avi
>f+++++++++ rsync.txt
>f+++++++++ 新建文件夹/542D0.mp4
```
其中`>f+++++++++`中的f代表的是文件，d代表的目录。

上面的rsync比较目录有几点要说明：  
1. 一定不能缺少-n选项，它表示dry run，也就是试着进行rsync同步，但不会真的同步。  
2. 第一个目录(directory1/)后一定不能缺少斜线，否则表示将directory1整个目录同步到directory2目录下。  
3. 其它选项，如`-r -v --delete`也都不能缺少，它们的意义想必都知道。  
4. sed的作用是过滤掉和文件不相关的内容。
5. 以上rsync假定了比较的两个目录中只有普通文件和目录，没有软链接、块设备等特殊文件。如果有，请考虑加上对应的选项或者使用`-a`替代`-r`，否则结果中将出现skipping non-regular file的提示。但请注意，如果有软链接，且加了对应选项(`-l`或`-a`或其他相关选项)，则可能会出现`fileA-->fileB`的输出。
6. 效率很高，因为rsync的原因，筛选的可定制性也非常强。

## 方法五：把文件列表装进数组进行比较

把目录1内的所有文件放进数组1，目录2内的所有文件放进数组2，比较数组1和数组2即可。这种方式也可以考虑使用脚本语言快速实现。

当然，也可以放进关联数组或hash结构。下面是shell脚本的实现方式：

```shell
#!/usr/bin/env bash

declare -A arr1 arr2

old_ifs="$IFS"
IFS=$'\n'
for i in $(ls -1A "$1");do arr1["$i"]=1; done
for i in $(ls -1A "$2");do arr2["$i"]=1; done
IFS="$old_ifs"

echo "--- files in in '$1' and not in '$2' ---"
for key in "${!arr1[@]}";do 
  [ "${arr2[$key]}" ] || echo "$key"
done

echo "--- files in in '$2' and not in '$1' ---"
for key in "${!arr2[@]}";do 
  [ "${arr1[$key]}" ] || echo "$key"
done
```

执行：

```shell
$ bash a.sh directory1 directory2
```

<a name="blog2"></a>

# 2.图形化的比较结果

## 方法一：使用vimdiff

```
vimdiff <(cd directory1; find . | sort) <(cd directory2; find . | sort)
# 或者
vimdiff <(find directory1 -printf "%P\n"| sort) <(find directory2 -printf "%P\n"| sort)
```

![](/img/linux/733013-20180522104306656-1627820234.png)


## 方法二：使用meld

meld是python写的一个图形化文件/目录比较工具，所以必须先安装图形界面或设置好图形界面接受协议。它的功能非常丰富，和win下的beyond compare有异曲同工之妙。

meld具体的使用方式就不介绍了。

<a name="blog3"></a>
# 3.将两目录中不同的文件筛选出来

个人建议使用[命令行输出的结果](#blog1)中的方法方法三和方法四，因为它们都能很好地保留目录前缀。

以方法三为例：
```
find directory1 -printf "%P\n" | sort > file1
find directory2 -printf "%P\n" | sort | diff file1 -
```

以下是实验所需目录结构：
```
[root@node1 ~]# tree /root/a;tree /root/b 
/root/a
├── 1.png
├── 2.png
└── 3.png

0 directories, 3 files
/root/b
├── 2.png
├── 3.png
├── 4.png
└── xen
    └── scripts
        └── block-drbd
```

首先比较这两个目录得到文件列表的差异。
```
find /root/a -printf "%P\n" | sort > /tmp/file1
find /root/b -printf "%P\n" | sort | diff /tmp/file1 - >diff.txt
```
然后从diff.txt中过滤出/root/a中多出的文件和/root/b中多出的文件。
```
# /root/a中多出的文件
awk '/</{printf("%s%s\n","/tmp/etc/",$2)}' diff.txt
/tmp/etc/1.png

# /root/b中多出的文件
awk '/>/{printf("%s%s\n","/tmp/etc/",$2)}' diff.txt
/tmp/etc/4.png
/tmp/etc/xen
/tmp/etc/xen/scripts
/tmp/etc/xen/scripts/block-drbd
```

需要注意的是，如果多了某个目录，则这个目录和其内所有文件都会列出来。如果要将多出的文件复制到其他地方，应当要注意这一点。

如果只想要比较出/root/a和/root/b下的文件和目录的不同，不再递归到子目录中比较。那么可以在find上继续加工一番：
```
find /root/a -maxdepth 1 -printf "%P\n" | sort > /tmp/file1
find /root/b -maxdepth 1 -printf "%P\n" | sort | diff /tmp/file1 - >diff.txt
# /root/a中多出的文件
awk '/</{printf("%s%s\n","/tmp/etc/",$2)}' diff.txt
/tmp/etc/1.png

# /root/b中多出的文件
awk '/>/{printf("%s%s\n","/tmp/etc/",$2)}' diff.txt
/tmp/etc/4.png
/tmp/etc/xen
```

这样一来，/root/b中多出的文件就是4.png和xen，xen目录中的文件不再列出。
