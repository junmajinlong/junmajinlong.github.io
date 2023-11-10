---
title: 快速删除大量小文件(多种方式速度对比)
p: shell/rm_many_smallfiles.md
date: 2021-03-13 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# 快速删除大量小文件(多种方式速度对比)

要测试删除大量小文件，首先需要先创建大量小文件，比如创建50W个txt文件：

```shell
mkdir /tmp/temp && cd /tmp/temp
seq -f "%g.txt" 1 500000 | xargs -P 4 -n 10000 touch
```

最快的是直接删除目录(不是瞬间的，只是跟测试的几种方式比，更快)，不测试了。

方法一：使用rsync

```shell
$ mkdir /tmp/empty
$ time rsync -r --delete /tmp/empty/ /tmp/temp/

real    0m8.556s
user    0m0.239s
sys     0m7.380s
```

方法二：使用Perl

```shell
$ cd /tmp/temp/
$ time perl -e 'unlink for (<"*">)'

real    0m8.773s
user    0m0.281s
sys     0m7.582s
```

方法三：使用Ruby

```shell
$ cd /tmp/temp/
$ time ruby -e 'Dir["*.txt"].each {|x| File.unlink x}'

real    0m8.882s
user    0m0.613s 
sys     0m7.429s

$ time ruby -e 'Dir.each_child(".") {|f| File.unlink "/tmp/temp/"+f}'

real    0m6.999s
user    0m0.443s
sys     0m5.912s
```

方法四：使用Python

```shell
$ cat a.py
import os
dir = "/tmp/temp/"
for file in os.listdir(dir):
  os.unlink(dir+file)

$ time python3 a.py

real    0m6.417s
user    0m0.339s
sys     0m5.451s
```

方法五：使用find结合xargs

```shell
# 4个rm进程(我只有4核)，每次删一个文件
$ time bash -c 'find /tmp/temp/ -type f -print0 | xargs -0 -P4 -i rm -rf {}' 

real    2m51.573s
user    0m3.104s
sys     3m59.286s

# 4个rm进程(我只有4核)，每次删20000个文件
$ time bash -c 'find /tmp/temp/ -type f -print0 | xargs -0 -P4 -n20000 rm -rf'     

real    0m12.746s
user    0m0.282s
sys     0m19.133s
```

