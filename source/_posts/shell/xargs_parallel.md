---
title: shell高效处理文本(1)：xargs并行处理
p: shell/xargs_parallel.md
date: 2019-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

xargs具有并行处理的能力，在处理大文件时，如果应用得当，将大幅提升效率。

xargs详细内容(全网最详细)：[https://www.junmajinlong.com/shell/xargs](https://www.junmajinlong.com/shell/xargs/#xargs_parallel)

# 效率提升测试结果

先展示一下使用xargs并行处理提升的效率，稍后会解释下面的结果。

测试环境：  
1. win10子系统上  
2. 32G内存  
3. 8核心cpu  
4. 测试对象是一个放在固态硬盘上行的10G文本文件(如果你需要此测试文件，[点此下载](https://pan.baidu.com/s/1l17GJl0tTBFjDfZ_fRxDqA)，提取码: semu)  

下面是正常情况下`wc -l`统计这个10G文件行数的结果，**花费16秒，多次测试，cpu利用率基本低于80%**。
```shell
$ /usr/bin/time wc -l 9.txt
999999953 9.txt
4.56user 3.14system 0:16.06elapsed 47%CPU (0avgtext+0avgdata 740maxresident)k
0inputs+0outputs (0major+216minor)pagefaults 0swaps
```

通过分割文件，使用xargs的并行处理功能进行统计，**花费时间1.6秒，cpu利用率752%**：
```shell
$ /usr/bin/time ./b.sh
999999953
7.67user 4.54system 0:01.62elapsed 752%CPU (0avgtext+0avgdata 1680maxresident)k
0inputs+0outputs (0major+23200minor)pagefaults 0swaps
```

用grep从这个10G的文本文件中筛选数据，**花费时间24秒，cpu利用率36%**：
```shell
$ /usr/bin/time grep "10000" 9.txt >/dev/null
6.17user 2.57system 0:24.19elapsed 36%CPU (0avgtext+0avgdata 1080maxresident)k
0inputs+0outputs (0major+308minor)pagefaults 0swaps
```

通过分割文件，使用xargs的并行处理功能进行统计，**花费时间1.38秒，cpu利用率746%**：
```shell
$ /usr/bin/time ./a.sh
6.01user 4.34system 0:01.38elapsed 746%CPU (0avgtext+0avgdata 1360maxresident)k
0inputs+0outputs (0major+31941minor)pagefaults 0swaps
```

速度提高的不是一点点。

![](/img/referer.jpg)

# xargs并行处理简单示例

要使用xargs的并行功能，只需使用"-P N"选项即可，其中N是指定要运行多少个并行进程，如果指定为0，则使用尽可能多的并行进程数量。

需要注意的是：  
- 既然要并行，那么xargs必须得分批传送管道的数据，xargs的分批选项有"-n"、"-i"、"-L"，如果不知道这些内容，看本文开头给出的文章。  
- 并行进程数量应该设置为cpu的核心数量。如果设置为0，在处理时间较长的情况下，很可能会并发几百个甚至上千个进程。在我测试一个花费2分钟的操作时，创建了500多个进程。  
- 在本文后面，还给出了其它几个注意事项。  

例如，一个简单的sleep命令，在不使用"-P"的时候，默认是一个进程按批的先后进行处理：
```shell
$ time echo {1..4} | xargs -n 1 sleep
 
real    0m10.011s
user    0m0.000s
sys     0m0.011s
```
总共用了10秒，因为每批传一个参数，第一批睡眠1秒，然后第二批睡眠2秒，依次类推，还有3秒、4秒，共1+2+3+4=10秒。

如果使用-P指定4个处理进程，它将以处理时间最长的为准：
```shell
$ time echo {1..4} | xargs -n 1 -P 4 sleep
 
real    0m4.005s
user    0m0.000s
sys     0m0.007s
```

再例如，find找到一大堆文件，然后用grep去筛选：
```shell
find /path -name "*.log" | xargs -i grep "pattern" {}
find /path -name "*.log" | xargs -P 4 -i grep "pattern" {}
```
上面第一个语句，只有一个grep进程，一次处理一个文件，每次只被其中一个cpu进行调度。也就是说，它无论如何，都只用到了一核cpu的运算能力，在极端情况下，cpu的利用率是100%。

上面第二个语句，开启了4个并行进程，一次可以处理从管道传来的4个文件，在同一时刻这4个进程最多可以被4核不同的CPU进行调度，在极端情况下，cpu的利用率是400%。

![](/img/referer.jpg)

# 并行处理示例

下面是文章开头给出的实验结果对应的示例。一个10G的文本文件9.txt，这个文件里共有9.9亿(具体的是999999953)行数据。

首先一个问题是，怎么统计这么近10亿行数据的？`wc -l`，看看时间花费。
```shell
$ /usr/bin/time wc -l 9.txt
999999953 9.txt
4.56user 3.14system 0:16.06elapsed 47%CPU (0avgtext+0avgdata 740maxresident)k
0inputs+0outputs (0major+216minor)pagefaults 0swaps
```

总共**花费了16.06秒，cpu利用率是47%**。

随后，我把这10G数据用split切割成了100个小文件，在提升效率方面，split切割也算是妙用无穷：
```shell
split -n l/100 -d -a 3 9.txt fs_
```

这100个文件，每个105M，文件名都以"fs_"为前缀：
```shell
$ ls -lh fs* | head -n 5
-rwxrwxrwx 1 root root 105M Oct  6 17:31 fs_000
-rwxrwxrwx 1 root root 105M Oct  6 17:31 fs_001
-rwxrwxrwx 1 root root 105M Oct  6 17:31 fs_002
-rwxrwxrwx 1 root root 105M Oct  6 17:31 fs_003
-rwxrwxrwx 1 root root 105M Oct  6 17:31 fs_004
```

然后，用xargs的并行处理来统计，以下是统计脚本`b.sh`的内容：
```shell
#!/usr/bin/env bash

find /mnt/d/test -name "fs*" |\
 xargs -P 0 -i wc -l {} |\
 awk '{sum += $1}END{print sum}'
```

上面用`-P 0`选项指定了尽可能多地开启并发进程数量，如果要保证最高效率，应当设置并发进程数量等于cpu的核心数量(在我的机器上，应该设置为8)，因为在操作时间较久的情况下，可能会并行好几百个进程，这些进程之间进行切换也会消耗不少资源。

然后，用这个脚本去统计测试：
```shell
$ /usr/bin/time ./b.sh
999999953
7.67user 4.54system 0:01.62elapsed 752%CPU (0avgtext+0avgdata 1680maxresident)k
0inputs+0outputs (0major+23200minor)pagefaults 0swaps
```

只**花了1.62秒，cpu利用率752%**。和前面单进程处理相比，时间是原来的16分之1，cpu利用率是原来的好多好多倍。

再来用grep从这个10G的文本文件中筛选数据，例如筛选包含"10000"字符串的行：
```shell
$ /usr/bin/time grep "10000" 9.txt >/dev/null
6.17user 2.57system 0:24.19elapsed 36%CPU (0avgtext+0avgdata 1080maxresident)k
0inputs+0outputs (0major+308minor)pagefaults 0swaps
```

**24秒，cpu利用率36%**。

再次用xargs来处理，以下是脚本：
```shell
#!/usr/bin/env bash

find /mnt/d/test -name "fs*" |\
 xargs -P 8 -i grep "10000" {} >/dev/null
```
测试结果：
```shell
$ /usr/bin/time ./a.sh
6.01user 4.34system 0:01.38elapsed 746%CPU (0avgtext+0avgdata 1360maxresident)k
0inputs+0outputs (0major+31941minor)pagefaults 0swaps
```

**花费时间1.38秒，cpu利用率746%**。

这比用什么ag、ack替代grep有效多了。

![](/img/referer.jpg)

# 提升哪些效率以及注意事项

xargs并行处理用的好，能大幅提升效率，但这是有条件的。 

首先要知道，xargs是如何提升效率的，以grep命令为例：

```shell
ls fs* | xargs -i -P 8 grep 'pattern' {}
```

之所以xargs能提高效率，是因为xargs可以分批传递管道左边的结果给不同的并发进程，也就是说，xargs要高效，得有多个文件可处理。对于上面的命令来说，ls可能输出了100个文件名，然后1次传递8个文件给8个不同的grep进程。

还有一些注意事项：  
1.**如果只有单核心cpu，想提高效率，没门**  
2.**xargs的高效来自于处理多个文件，如果你只有一个大文件，那么需要将它切割成多个小片段**  
3.**由于是多进程并行处理不同的文件，所以命令的多行输出结果中，顺序可能会比较随机**  

例如，统计行数时，每个文件的出现顺序是不受控制的。
```
10000000 /mnt/d/test/fs_002
9999999 /mnt/d/test/fs_001
10000000 /mnt/d/test/fs_000
10000000 /mnt/d/test/fs_004
9999999 /mnt/d/test/fs_005
9999999 /mnt/d/test/fs_003
10000000 /mnt/d/test/fs_006
9999999 /mnt/d/test/fs_007
```

不过大多数时候这都不是问题，将结果排序一下就行了。

**4.xargs提升效率的本质是cpu的利用率，因此会有内存、磁盘速度的瓶颈。如果内存小，或者磁盘速度慢(将因为加载数据到内存而长时间处于io等待的睡眠状态)，xargs的并行处理基本无效**。  

例如，将上面10G的文本文件放在虚拟机上，机械硬盘，内存2G，将会发现使用xargs并行和普通的命令处理几乎没有差别，因为绝大多数时间都花在了加载文件到内存的io等待上。  

后面将介绍GNU parallel并行处理工具，它的功能更丰富，效果更强大。