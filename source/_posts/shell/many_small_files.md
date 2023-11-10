---
title: Linux快速生成大量随机大小的小文件
p: shell/many_small_files.md
date: 2021-03-13 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Linux快速生成大量随机大小的小文件

```
# 在当前目录下，生成50W个大小0-8K的随机txt文件
n=500000
size=8
time perl -E '
  $n=shift;
  $max_size=1024 * shift;
  for(1..$n){
    open $f, ">", "$_.txt" or die "open failed: $!";
    print {$f} "0" x int(rand($max_size));
    close $f or die "close failed: $!";
  }
' $n $size

real    0m8.073s
user    0m1.618s
sys     0m6.313s
```

