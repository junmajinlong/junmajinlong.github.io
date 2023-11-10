---
title: dd、split、csplit命令
p: shell/dd_split_csplit.md
date: 2019-07-06 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# dd、split、csplit命令

在Linux最常用的文件生成和切片工具是dd，它功能比较全面，但无法以行为单位提取文件数据，也无法直接将文件按大小或行数进行均分(除非借助循环)。另两款数据分割工具split和csplit能够比较轻松地实现这些需求。csplit是split的升级版。

在处理很大的文件时，一个非常高效的思路是将大文件切割成多个小文件片段，然后再通过多个进程/线程对各个小文件进行操作，最后合并总数居。就像sort命令，它在实现排序时，底层算法就涉及到了将一个大文件切割成多个临时小文件。

## dd命令

从if指定的文件读取数据，写入到of指定的文件。使用bs指定读取和写入的块大小，使用count指定读取和写入的数据块数量，bs和count相乘就是文件总大小。可以指定skip忽略读取if指定文件的前多少个块，seek指定写入到of指定文件时忽略前多少个块。

```
dd if=/dev/zero of=/tmp/abc.1 bs=1M count=20
```

if是input file，of是output file；bs有c(1byte)、w(2bytes)、b(512bytes)、kB(1000bytes)、K(1024bytes)、MB(1000)、M(1024)和GB、G等几种单位。因此，不要随意在单位后加上字母B。

假设现有文件CentOS.iso的大小1.3G，需要将其切分后还原，切分的第一个小文件大小为500M。

```
dd if=/tmp/CentOS.iso of=/tmp/CentOS1.iso bs=2M count=250
```

![](/img/referer.jpg)

生成第二个小文件，由于第二个小文件不知道具体大小，所以不指定count选项。由于第二个小文件要从第500M处开始切分，于是需要忽略CentOS.iso的前500M。假设bs=2M，于是skip掉的数据块数量为250。

```
dd if=/tmp/CentOS.iso of=/tmp/CentOS2.iso bs=2M skip=250
```

现在CentOS.iso=CentOS1.iso+CentOS2.iso。可以将CentOS[1-2].iso还原。

```
cat CentOS1.iso CentOS2.iso >CentOS_m.iso
```

比较CentOS_m.iso和CentOS.iso的md5值，它们是完全一样的。

```
$ md5sum CentOS_m.iso CentOS.iso
504dbef14aed9b5990461f85d9fdc667  CentOS_m.iso
504dbef14aed9b5990461f85d9fdc667  CentOS.iso
```

那么seek选项呢？和skip有什么区别？skip选项是忽略读取时的前N个数据块，而seek是忽略写入文件的前N个数据块。假如要写入的文件为a.log，则seek=2时，将从a.log的第3个数据块开始追加数据，如果a.log文件本身大小就不足2个数据块，则缺少的部分自动使用/dev/zero填充。

于是，在有了CentOS1.iso的基础上，要将其还原为和CentOS.iso相同的文件，可以使用下面的方法：

```
dd if=/tmp/CentOS.iso of=/tmp/CentOS1.iso bs=2M skip=250 seek=250
```

还原后，它们的md5值也是相同的。

```
$ md5sum CentOS1.iso CentOS.iso
504dbef14aed9b5990461f85d9fdc667  CentOS1.iso
504dbef14aed9b5990461f85d9fdc667  CentOS.iso
```

![](/img/referer.jpg)

## split命令

split工具的功能是将文件切分为多个小文件。既然要生成多个小文件，必然要指定切分文件的单位，支持按行切分以及按文件大小切分，另外还需解决小文件命名的问题。例如，文件名前缀、后缀。如果未明确指定前缀，则默认的前缀为"x"。

以下是命令的语法说明：

```
split [OPTION]... [INPUT [PREFIX]]

-a N：生成长度为N的后缀，默认N=2

-b N: 每个小文件的N，即按文件大小切分文件。支持K,M,G,T(换算单位1024)
    : 或KB,MB,GB(换算单位1000)等，默认单位为字节
    
-l N：每个小文件中有N行，即按行切分文件

-d N：指定生成数值格式的后缀替代默认的字母后缀，数值从N开始，默认为0
    : 例如两位长度的后缀01/02/03

--additional-suffix=string
          : 为每个小文件追加额外的后缀，例如加上
          : ".log"。有些老版本不支持，在CentOS 7.2上已支持

-n CHUNKS：将文件按照指定的CHUNKS方式进行分割。CHUNKS的有效形式
         : 有(具体用法见后文)：N、l/N、l/K/N、K/N、r/N、r/K/N

--filter CMD：不再直接切割输出到文件，而是切割后作为管道的输入，
            : 通过管道将切割的数据传递给CMD执行。如果需要指定文件，
            : 则split自动使用$FILE变量。见后文示例

INPUT：指定待切分的输入文件，如要切分标准输入，则使用"-"
PREFIX：指定小文件的前缀，如果未指定，则默认为"x"
```

### 基本用法

例如，将/etc/fstab按行切分，每5行切分一次，并指定小文件的前缀为"fs_"，后缀为数值后缀，且后缀长度为2。

```shell
$ split -l 5 -d -a 2 /etc/fstab fs_

$ ls
fs_00  fs_01  fs_02
```

查看任一小文件。

```shell
$ cat fs_01
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=b2a70faf-aea4-4d8e-8be8-c7109ac9c8b8 /                       xfs     defaults        0 0
UUID=367d6a77-033b-4037-bbcb-416705ead095 /boot                   xfs     defaults        0 0
```

可以将这些切分后的小文件重新组装还原。例如，将上面的三个小文件还原为~/fstab.bak。

```shell
$ cat fs_0[0-2] >~/fstab.bak
```

还原后，它们的内容是完全一致的。可以使用md5sum比较。

```shell
$ md5sum /etc/fstab ~/fstab.bak
29b94c500f484040a675cb4ef81c87bf  /etc/fstab
29b94c500f484040a675cb4ef81c87bf  /root/fstab.bak
```

还可以将标准输入的数据进行切分，并分别写入到小文件中。例如：

```shell
$ seq 1 2 15 | split -l 3 -d - new_

$ ls new*
new_00  new_01  new_02
```

可以为每个小文件追加额外的后缀。有些老版本的split不支持该选项，而是在csplit上支持的，但是新版本的split已经支持。例如，加上".log"。

```shell
$ seq 1 2 20 | split -l 3 -d -a 3 --additional-suffix=".log" - new1_

$ ls new1*
new1_000.log  new1_001.log  new1_002.log  new1_003.log
```

![](/img/referer.jpg)

### 按CHUNKS进行分割

split的"-n"选项是按照CHUNK的方式进行文件切割：

```shell
'-e' 当使用-n时，不要生成空文件。例如5行数据，却要求按行切割成100个文件，显然第5个文件之后都是空文件

'-u --unbuffered' 当使用-n的r模式时，不要缓冲input，每读一行就立即拷贝给输出，所以该选项可能会比较慢

'-n CHUNKS'
'--number=CHUNKS'
     Split INPUT to CHUNKS output files where CHUNKS may be:
          N      根据文件大小均分为N个文件(最后一个文件可能大小不均)
          K/N    输出(打印到屏幕、标准输出)N个文件中的第K个文件内容
                 (不是切割后再找到这个文件进行输出，而是切割到这个文件时立即输出)
          l/N    按行的方式均分为N个文件(最后一个文件可能行数不均)
          l/K/N  按l/N切割的同时，输出属于第K个文件的内容
          r/N    类似于l的形式，但对行进行轮询方式切割。例如第一行到
                 第一个文件，第二行到第二个文件
          r/K/N  按照r/N切割时，输出第K个文件的内容
 
其中大写字母K和N是我们按需指定的数值，l或r是代表模式的字母
```

可能不是很好理解，给几个示例就很清晰了。

假如文件1.txt中有a-z，每个字母占一行，共26行。将这个文件进行切割：

**1.指定CHUNK=N或K/N时**

将1.txt均分为5个文件。由于1.txt共52字节(每行中，字母一个字节，换行符一个字节)，所以分割后每个文件10字节，最后一个文件12字节。

```shell
$ split -n 5 1.txt fs_

$ ls -l
total 24
-rw-r--r-- 1 root root 52 Oct  6 15:23 1.txt
-rw-r--r-- 1 root root 10 Oct  6 16:07 fs_aa
-rw-r--r-- 1 root root 10 Oct  6 16:07 fs_ab
-rw-r--r-- 1 root root 10 Oct  6 16:07 fs_ac
-rw-r--r-- 1 root root 10 Oct  6 16:07 fs_ad
-rw-r--r-- 1 root root 12 Oct  6 16:07 fs_ae
```

如果再指定K/N的K，例如指定为2，则输出属于fs_ab文件中的内容：

```shell
$ split -n 2/5 1.txt fs_
f
g
h
i
j
```

**2.指定CHUNK=l/N或l/K/N时**

这时将按照行的总数量进行均分。例如，将1.txt中的26行分割成5个文件，前4个文件将各有5行，第5个文件将有6行：

```shell
$ split -n l/5 1.txt fs_

$ wc -l fs_*
 5 fs_aa
 5 fs_ab
 5 fs_ac
 5 fs_ad
 6 fs_ae
26 total
```

如果指定K，则输出属于第K个文件的内容：

```shell
$ split -n l/2/5 1.txt fs_
f
g
h
i
j
```

**3.指定CHUNK=r/N或r/K/N时**

使用r时，将每切割一行就轮询到下一个文件，轮完所有文件再返回来切割到第一个文件。

直接看结果：

```shell
$ split -n r/5 1.txt fs_  

$ head -n 2 fs*
==> fs_aa <==
a
f
 
==> fs_ab <==
b
g
 
==> fs_ac <==
c
h
 
==> fs_ad <==
d
i
 
==> fs_ae <==
e
j
```

a输出到第1个文件，b输出到第2个文件，c输出到第3个文件，依此类推。

指定K时，将输出第K个文件的内容：

```shell
$ split -n r/2/5 1.txt fs_
b
g
l
q
v
```

![](/img/referer.jpg)

### filter CMD将切割结果传递给指定命令

默认情况下split是将文件切割传递到各文件片段中，如果使用--filter选项，则不再切割到文件片段中，而是通过管道的形式传递给CMD进行处理。CMD处理时，有可能并不需要将数据存储到文件中，但如果需要将处理后的数据再分片段保存到多个文件中，则可以使用$FILE来表示(这是split识别的变量，要避免被shell解析)，就像split的普通切割模式一样。

例如，读取1.txt中的26行，每5行管道传递一次，然后使用echo将它们输出：

```shell
$ split -n l/5 --filter='xargs -i echo ---:{}' 1.txt
---:a
---:b
---:c
---:d
---:e   # 第1次通过管道传递的内容
---:f
---:g
---:h
---:i
---:j   # 第2次通过管道传递的内容
---:k
---:l
---:m
---:n
---:o
---:p
---:q
---:r
---:s
---:t
---:u
---:v
---:w
---:x
---:y
---:z
```

这时split是没有生成任何小文件片段的。如果想要将上面的输出保存到小文件片段中，使用$FILE，这个变量是split内置变量，不能被shell解析，所以出现$split的地方必须使用单引号保护起来：

```shell
$ split -n l/5 --filter='xargs -i echo ---:{} >$FILE.log' 1.txt fs_

$ ls
1.txt  fs_aa.log  fs_ab.log  fs_ac.log  fs_ad.log  fs_ae.log

$ cat fs_aa.log
---:a
---:b
---:c
---:d
---:e
```

其中的$FILE就是split进行命名的部分。上面的小文件中，都是以fs_为前缀，以".log"为后缀。

filter有时候是很有用的，例如将一个大的压缩文件，切割成多个小的压缩文件。

```shell
xz -dc BIG.xz | split -b200G --filter='xz > $FILE.xz' - big-
```

`xz -dc`表示解压到标准输出，解压的数据流将传递给split，然后每200G就通过filter中的xz命令进行压缩，压缩后的文件名格式类似于`big-aa.xz big-ab.xz`。

## csplit命令

split只能按行或按照大小进行切分，无法按段落切分。csplit是split的变体，功能更多，它主要是按指定上下文按段落分割文件。

```
csplit [OPTION]... FILE PATTERN...
 
描述：按照PATTERN将文件切分为"xx00","xx01", ...，
     并在标准输出中输出每个小文件的字节数。
 
选项说明：
-b FORMAT: 指定文件后缀格式，格式为printf的格式，
         : 默认为%02d。表示后缀以2位数值，且不足处以0填充。
-f PREFIX：指定前缀，不指定是默认为"xx"。
-k：用于突发情况。表示即使发生了错误，也不删除已经分割完成的小文件。
-m：明确禁止文件的行去匹配PATTERN。
-s：(silent)不打印小文件的文件大小。
-z：如果切分后的小文件中有空文件，则删除它们。
 
FILE：待切分的文件，如果要切分标准输入数据，则使用"-"。
 
PATTERNs：
 
INTEGER         ：数值，假如为N，表示拷贝1到N-1行的内容到一个小
                ：文件中，其余内容到另一个小文件中。
/REGEXP/[OFFSET]：从匹配到的行开始按照偏移量拷贝指定行数的内容到小文件中。
                ：其中OFFSET的格式为"+N"或"-N"，表示向后和向前拷贝N行
%REGEXP%[OFFSET]：匹配到的行被忽略。
{INTEGER}       ：假如值为N，表示重复N此前一个模式匹配。
{*}             ：表示一直匹配到文件结尾才停止匹配。
```

假设文件内容如下：

```shell
$ cat test.txt
SERVER-1
[connection] 192.168.0.1 success
[connection] 192.168.0.2 failed
[disconnect] 192.168.0.3 pending
[connection] 192.168.0.4 success
SERVER-2
[connection] 192.168.0.1 failed
[connection] 192.168.0.2 failed
[disconnect] 192.168.0.3 success
[CONNECTION] 192.168.0.4 pending
SERVER-3
[connection] 192.168.0.1 pending
[connection] 192.168.0.2 pending
[disconnect] 192.168.0.3 pending
[connection] 192.168.0.4 failed
```

假设每个SERVER-n表示一个段落，于是要按照段落切分该文件，使用以下语句：

```shell
$ csplit -f test_  -b %04d.log test.txt /SERVER/ {*}
0
140
139
140
```

"-f test_"指定小文件前缀为"test_"，`-b %04d.log`指定文件后缀格式"00xx.log"，它自动为每个小文件追加额外的后缀".log"，"/SERVER/"表示匹配的模式，每匹配到一次，就生成一个小文件，且匹配到的行是该小文件中的内容，`{*}`表示无限匹配前一个模式即/SERVER/直到文件结尾，假如不知道`{*}`或指定为`{1}`，将匹配一次成功后就不再匹配。

```shell
$ ls test_*
test_0000.log  test_0001.log  test_0002.log  test_0003.log
```

上面的文件中虽然只有三个段落：SERVER-1，SERVER-2，SERVER-3，但切分的结果生成了4个小文件，并且注意到第一个小文件大小为0字节。为什么会如此？因为在模式匹配的时候，每匹配到一行，这一行就作为下一个小文件的起始行。由于此文件第一行"SERVER-1"就被/SERVER/匹配到了，因此这一行是作为下一个小文件的内容，在此小文件之前还自动生成一个空文件。

生成的空文件可以使用"-z"选项来删除。

```shell
$ csplit -f test1_ -z -b %04d.log test.txt /SERVER/ {*}
140
139
140
```

![](/img/referer.jpg)


还可以指定只拷贝匹配到的行偏移数量。例如，匹配到行时，只拷贝它后面的1行(包括它自身共两行)，但多余的行将放入下一个小文件中。

```shell
$ csplit -f test2_ -z -b %04d.log test.txt /SERVER/+2 {*}
42
139
140
98
```

第一个小文件只有两行。

```shell
$ cat test2_0000.log 
SERVER-1
[connection] 192.168.0.1 success
```

SERVER-1段落的其余内容放入到了第二个小文件中。

```shell
$ cat test2_0001.log
[connection] 192.168.0.2 failed
[disconnect] 192.168.0.3 pending
[connection] 192.168.0.4 success
SERVER-2
[connection] 192.168.0.1 failed
```

同理第三个小文件也一样，直到最后一个小文件中存放剩余所有无法匹配的内容。

```shell
$ cat test2_0003.log 
[connection] 192.168.0.2 pending
[disconnect] 192.168.0.3 pending
[connection] 192.168.0.4 failed
```

指定"-s"或"-q"选项以静默模式运行，将不会输出小文件的大小信息。

```shell
$ csplit -q -f test3_ -z -b %04d.log test.txt /SERVER/+2 {*}
```