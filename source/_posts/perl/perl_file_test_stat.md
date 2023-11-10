---
title: Perl文件测试操作和stat函数
p: perl/perl_file_test_stat.md
date: 2019-07-06 17:37:56
tags: Perl
categories: Perl
---

# Perl文件测试操作和stat函数

在shell中通过test命令或者中括号`[]`可以进行文件测试以及其它类型的测试，例如判断文件是否存在，比较操作是否为真等等。perl作为更强大的文本处理语言，它也有文件测试类表达式，而且和shell的文件测试用的字母符号都类似。

perl中测试文件的属性来源是perl的内置函数stat，它可以获得文件的13项属性。后文会介绍该函数。

## 测试符

测试符号都是短横线开头，加一个字母。例如，测试文件是否存在`-e "a.log"`。在可能产生歧义的情况下，这些测试符可以用括号包围，例如：`(-e "a.log")`。

注意，perl主要应用于Unix类系统，而Unix中一切皆文件，所以下表中出现的【文件】如非特地指明，否则它既可表示普通文件，也表示目录，还表示其它类型的文件。

**以下是文件大小测试符**：

```
符号   意义
--------------------------------------
-e    文件是否存在
-z    文件是否存在且为空(对目录而言，永远为假)
-s    文件是否存在且不为空，返回值是文件大小，单位为字节
```

其中`-s`会返回文件的字节数大小。对于真假的判断，通过这个返回值也可以直接判断，大于0表示非空，等于0表示空。

例如：
```
print "not empty" if "-s /tmp/a.log";
print "empty" unless "! -s /tmp/a.log";
print ((-s "/usr/bin/passwd") + 10);   # 返回文件字节数+10
```

**以下是文件类型测试符**：
```
符号    意义(同时检测文件的存在性)
---------------------------------------
-f     文件是否为普通文件
-d     文件是否为目录文件
-l     文件是否为软链接(字符链接)
-b     文件是否为块设备
-c     文件是否是字符设备文件
-p     文件是否为命名管道
-S     文件是否为socket文件
```

**以下是权限类测试符**：

首先区分一下Effective uid和real uid：
- real uid：文件调用者的uid
- effective uid：文件调用最后生效的uid

如未对文件进行特殊设置，real uid和effective uid是一致的。但是如果设置了setuid属性，那么文件在执行的时候会提升为某用户的权限(一般是提升为root)，这时候effective uid就是root，而real uid则是文件调用者的uid。

比如longshuai用户读取一个文件，则这个文件的real uid就是longshuai。比如/usr/bin/passwd这个文件设置了setuid，调用这个程序的用户是real uid，最后执行的时候提权为root用户，那么这个程序的effective uid就是root。

一般来说，只会去检测real uid的权限属性。只有极少数文件会设置setuid/setgid/sticky，去检测这类文件的权限的机会就更小了。

```
符号  意义(同时检测文件的存在性)
-------------------------------------
-r   文件(对effective uid)是否可读
-w   文件(对effective uid)是否可写
-x   文件(对effective uid)是否可执行
-o   文件(对effective uid)的所有者

-R   文件(对real uid)是否可读
-W   文件(对real uid)是否可写
-X   文件(对real uid)是否可执行
-O   文件(对real uid)的所有者

-u   文件是否设置了setuid (setuid只对可执行普通文件有效)
-g   文件是否设置了setgid (setgid只对普通文件或目录有效)
-k   文件是否设置了sticky (sticky属性只对目录有效)
```

例如：
```
print "readable" if -r "/tmp/a.log";
```

**以下是其它测试符**：
```
符号   意义
--------------------------------------
-e    文件是否存在
-z    文件是否存在且为空(对目录而言，永远为假)
-s    文件是否存在且不为空，返回值是文件大小，单位为字节

-M    最后一次修改(mtime)距离目前的天数
-A    最后一次访问(atime)距离目前的天数
-C    最后一次inode修改(ctime)距离目前的天数

-T    文件看起来像文本文件
-B    文件看起来像二进制文件
-t    文件句柄是否为TTY设备(该测试只对文件句柄有效)
```

上面`-M/-A/-C`会计算天数，它是(小时数/24)来计算的。例如，6小时前修改的文件，它的天数就是0.25天。

上面的`-T/-B`是mime类型猜测，perl会根据文件的前几个字节来猜测这个文件，当然，它不一定能猜对，但大多数时候是没什么问题的。

例如：
```
# 修改文件mtime时间为6小时前
touch -m -t "6 hours ago" /tmp/b.log
```

如果perl程序内容为：
```
print (-M "/tmp/b.log");
```
它的执行结果将输出0.25：
```
0.250162037037037
```

以下是文件mime类型猜测。脚本内容如下：
```
print "text file\n" if -T "$ARGV[0]";
print "Binary file\n" if -B "$ARGV[0]";
```
执行结果如下：
```
$ ./19.pl 1.plx 
text file
$ ./19.pl /etc
Binary file
$ ./19.pl initramfs-3.10.0-327.el7.x86_64.img
Binary file
$ ./19.pl /bin/ls
Binary file
```

## 测试符的陷阱

测试符操作可以省略参数，这时它的操作对象是默认变量`$_`。但是对于`-t`测试符来说例外，它的默认操作对象是`<STDIN>`，因为它的对象是文件句柄，而非文件名。

例如：
```
#!/usr/bin/perl

foreach (`ls`){
    chomp;
    print "$_ is executable\n" if -x;
}
```

但是省略参数的时候，很容易出错。操作符会把后面任何一个非空格字符(串)当作它的参数。例如，`-s`返回的是字节大小，你想让它按kb显示。
```
print (-s / 1024);
```
但这时的"-s"会把`/`当作它的测试对象参数，而不是`$_`。所以，省略参数的时候，建议将测试符用括号包围起来：
```
print ((-s) / 1024);
```

## 测试文件多个属性符(1)：缓存文件测试信息

如果想要同时测试文件的可读性、可写性：
```
print "writable and readable\n" if -w "/tmp/a.log" and -r "/tmp/a.log";
```

但是这不是最佳方式，因为perl每执行一次测试符表达式，都需要对文件执行一次stat函数，但实际上第二次测试执行的stat是多余的，因为一次测试就可以获取到文件的所有属性。

perl中有一个特殊的缓存文件句柄`_`(就是一个下划线)，它可以最近的缓存文件属性信息。

下面是等价的操作：
```
print "writable and readable\n" if -w "/tmp/a.log" and -r _;
```
`_`的缓存周期可以延续，直到测试下一个文件，所以将两次测试分开写也可以。但需要注意的是，在使用`_`的时候要确保它缓存的对象正是所需要的文件属性。
```
print "a.log writable\n" if -w "/tmp/a.log";
print "b.log writable\n" if -w "/tmp/b.log";
print "b.log writable\n" if -r _;
```
上面第三个语句中`_`缓存的是/tmp/b.log文件的属性。

## 测试文件多个属性符(2)：栈式文件测试

可以将多个测试操作符连在一起写。

- 连写的时候，从右向左依次执行，并按照and逻辑运算符判断真假。也就是说，先测试靠近文件名的操作符
- 对于返回真/假值的测试符，连写的测试符前后顺序不会影响结果
- 对于返回非真/假值的测试符(即-s/-M/-A/-C)，连写测试符时应尽量谨慎，最保险的方式是不要连写

例如下面两个语句，它们在最终测试结果上是等价的。第一个语句先测试可写性，再测试可读性，只有两者均为真时if条件才为真。
```
print "writable and readable\n" if -r -w "/tmp/a.log";
print "writable and readable\n" if -w -r "/tmp/a.log";
```

但是，返回非真/假值的测试符，需要小心小心再小心。例如，`-s`返回的是文件字节数：
```
@arr=`ls`;
foreach (@arr){
    chomp;
    if (-s -f $_ < 512){     # 这里的结果会出乎意料
        print "${_}'s size < 512 bytes\n";
    }
}
```

上面的if条件子句等价于`(-f $_ and -s _) < 512`，它会输出小于512字节的普通文件，以及所有非普通文件。因为and是短路的，如果测试的目标`$_`不是一个普通文件，而是一个目录，`-f $_`就会返回假，并结束测试，然后这部分表达式和512做数值比较，假对应的数值是0，它永远会返回真。

所有，对于返回非真/假值的测试符，应该避免测试符连写：
```
@arr=`ls`;
foreach (@arr){
    chomp;
    if (-f $_ and -s _ < 512){
        print "${_}'s size < 512 bytes\n";
    }
}
```

## stat函数和lstat函数

虽然文件测试符有很多，且测试的属性来源都是stat函数，但stat函数返回的信息(共13项属性)比支持的文件测试符还要多。

注：Unix操作系统里有一个stat命令，它也是返回文件的属性信息，和perl的内置stat函数基本类似。

它返回13项属性先后顺序分别是：

```
my ($dev,$ino,$mode,$nlink,$uid,$gid,$rdev,$size,
    $atime,$mtime,$ctime,$blksize,$blocks)
    = stat($filename);

属性       意义
---------------------------------------------------
dev     ：文件所属文件系统的设备ID
inode   ：文件inode号码
mode    ：文件类型和文件权限(两者都是数值表示)
nlink   ：文件硬链接数
uid     ：文件所有者的uid
gid     ：文件所属组的gid
rdev    ：文件的设备ID(只对特殊文件有效，即设备文件)
size    ：文件大小，单位字节
atime   ：文件atime的时间戳(从1970-01-01开始计算的秒数)
mtime   ：文件mtime的时间戳(从1970-01-01开始计算的秒数)
ctime   ：文件ctime的时间戳(从1970-01-01开始计算的秒数)
blksize ：文件所属文件系统的block大小
blocks  ：文件占用block数量(一般是512字节的块大小，可通过unix的stat -c "%B"获取块的字节)
```
需要注意的是，$mode返回的是文件类型和文件权限的结合体，且文件权限并非直接的8进制权限值，要计算出Unix系统中直观的权限数值(如0755、0644)，需要和0777做位运算。

```
use 5.010;
$filename=$ARGV[0];

my @arr = ($dev,$ino,$mode,$nlink,$uid,$gid,$rdev,$size,
           $atime,$mtime,$ctime,$blksize,$blocks)
        = stat($filename);

say '$dev     :',$arr[0];
say '$inode   :',$arr[1];
say '$mode    :',$arr[2];
say '$nlink   :',$arr[3];
say '$uid     :',$arr[4];
say '$gid     :',$arr[5];
say '$rdev    :',$arr[6];
say '$size    :',$arr[7];
say '$atime   :',$arr[8];
say '$mtime   :',$arr[9];
say '$ctime   :',$arr[10];
say '$blksize :',$arr[11];
say '$blocks  :',$arr[12];
```
常用的位移是2、7、8、9，分别用来获取权限、大小、atime和mtime属性。这几项可以记忆下来，或`perldoc -f stat`查看。

返回结果：
```
$dev     :2050
$inode   :67326520
$mode    :33188
$nlink   :1
$uid     :0
$gid     :0
$rdev    :0
$size    :12
$atime   :1533544992
$mtime   :1533426824
$ctime   :1533426824
$blksize :4096
$blocks  :8
```

如果要计算文件权限，则：
```
printf "perm     :%04o\n",$mode & 0777;  # 将返回0644、0755类型的权限值
```

也可以直接一点取出某项属性的值：
```
my $mode = (stat($filename))[2];
```

以下是stat函数的其它一些注意事项：
- stat函数返回布尔值表示是否成功stat：
    - 如果stat成功，则立即设置这些属性变量，并缓存到特殊文件句柄`_`
    - 如果stat失败，则返回空列表
- 如果stat函数测试的是特殊文件句柄`_`，它将不会重新测试缓存文件，而是直接返回缓存的属性信息
- 如果省略stat函数的参数，则默认测试`$_`

对于软链接文件，stat会追踪到链接的目标。如果不想追踪，则使用lstat函数替代stat。lstat如果测试的目标不是软链接，则返回空列表。
