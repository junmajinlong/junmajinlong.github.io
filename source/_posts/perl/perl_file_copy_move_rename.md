---
title: Perl复制、移动、重命名文件/目录
p: perl/perl_file_copy_move_rename.md
date: 2019-07-06 17:37:59
tags: Perl
categories: Perl
---

# Perl复制、移动、重命名文件/目录

## File\::Copy复制文件

File\::Copy模块提供了copy函数和cp函数来复制文件，它们参数上完全一致，但行为上稍有区别。

用法大致如下：
```
use File::Copy qw(copy cp);
	copy("sourcefile","destinationfile") or die "Copy failed: $!";
	copy("Copy.pm",\*STDOUT);
```

- 两个参数都可以是文件或文件句柄或者文件句柄通配，第一个参数指定源，第二个参数指定目标
    - 如果第一个参数是文件句柄，那么将直接从文件句柄来读取数据，如果这个参数是文件，那么将打开这个文件来读取数据
    - 第二个参数是数据的写入目标
- 如果目标文件不存在，但父目录存在，则创建该目标文件，但如果父目录也不存在，则报错
- 如果目标文件存在，则覆盖该目标文件，不会给出任何提示
- 如果目标是一个已存在的目录，且源不是一个文件句柄，则拷贝到目标目录中，如果源是一个文件句柄，将报错
- 源和目标不能是同一文件
- 因为是拷贝操作，所以可以跨文件系统拷贝
- 第三个可选参数用于设置拷贝时的缓冲大小。对于文件来说，一般缓冲大小是整个文件(但最大2MB)，对于不关联实体文件的文件句柄(如套接字文件句柄)，默认为1K大小
- cp可以替换copy。它们的参数模式完全一致，但cp会保留源文件的属性，而copy则是采用目标文件的默认属性。此外，cp在遇到权限错误的时候返回0，而不管文件是否成功拷贝
- **强烈建议**：如果可以，都使用文件名而不是文件句柄。如果要使用文件句柄，则采用binmode模式的文件句柄，以免丢失某些数据
- File\::Copy模块无法操作目录，所以copy无法复制目录

例如，现在/mnt/g下创建一个文件t1.py。然后执行如下内容的perl程序：它将拷贝root下的t.py，然后再拷贝覆盖到已存在的t1.py。
```
use File::Copy qw(copy cp);
copy "/root/t.py","/mnt/g/" or die "Can't copy file1: $!";
cp qw(/mnt/g/t.py /mnt/g/t1.py) or die "Can't copy file2: $!";
```


## 重命名/移动文件

rename函数可以重命名文件，也可以移动文件到其它目录。功能类似于unix下的mv命令。

```
rename old_name,new_name;
rename old_name => new_name;   # 列表环境下，逗号可用胖箭头替换
```

但需要注意，rename函数无法跨文件系统移动文件，因为它的底层仅仅只是重命名，修改文件inode中的数据。跨文件系统移动文件，实际上是复制文件再删除源文件，它会导致inode号码改变，rename的本质是基于inode的，无法实现这样的功能。
```
rename "test2.log","test222.log"
    or die "Can't rename file1: $!";

rename "test222.log","/tmp/test223.log"
    or die "Can't rename file2: $!";

rename "/tmp/test223.log","/boot/test223.log"    # 本行将报错
    or die "Can't rename file3: $!";
```

File\::Copy模块提供了move函数，它可以跨文件系统移动文件。用法大致如下：
```
use File::Copy qw(move mv);
move("/dev1/sourcefile","/dev2/destinationfile");
mv("/dev1/sourcefile","/dev2/destinationfile");
mv("/dev1/sourcefile" => "/dev2/destinationfile");
```

- File\::Copy模块无法操作目录，所以move无法重命名或移动目录
- move有两个参数，第一个是源，第二个是目标
- 如果目标是个已存在的目录，而源是个非目录，则源将被移动到目标目录内
- 如果可以，move在文件系统底层只是简单地重命名文件。否则(例如跨文件系统)，将copy源文件到目标，然后删除源文件。如果这个copy+delete的过程中失败，则在目标路径下会遗留一个可能还未拷贝完成的副本
- move可以使用mv替代

其实，可以采用shell交互的方式来取巧重命名：
```
rename(old,new) or system("mv",old,new);
```


## 递归复制/移动File\::Copy\::Recursive

具体内容暂缺，可看官方手册：[File::Copy::Recursive](https://metacpan.org/pod/File::Copy::Recursive)
