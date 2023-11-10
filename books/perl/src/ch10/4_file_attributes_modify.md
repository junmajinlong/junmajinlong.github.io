## 获取文件属性

使用`stat`函数获取文件属性，包括文件权限、所有者、所属组、atime/mtime/ctime、大小等，`stat`函数前面已经介绍过，可参考前面内容。

## 修改文件权限

使用内置函数`chmod`修改权限：  
- 可修改单个文件或文件列表内所有文件的权限  
- 修改失败时，会设置`$!`  
- 返回成功设置的权限的文件数量  
- 只接受8进制的权限值，不允许使用rwx字符串格式的权限字符  

例如，设置目录、文件的权限： 

```perl
chmod 0700, qw(/test/foo /test/foo1/a.log);
chmod 0700, '/test/foo', '/test/foo1/a.log';
```

## 修改文件owner、group

使用内置函数`chown`修改文件的所有者和所属组信息。

```perl
chown UID, GID, FILENAME
chown UID, GID, LIST
```

其中：

- chown可以同时修改单个文件或目录的所有者、所属组，也可以同时修改列表内所有文件或目录的所有者、所属组  
- 修改失败时，会设置`$!`  
- 返回成功修改的数量  
- chown只接受uid和gid作为参数，不接受username和groupname  
  - 可使用getpwnam和getgrnam函数将username/groupname映射为uid/gid  
- 在uid或gid的参数位置上指定特殊值`-1`，表示该位置的所有者或所属组属性保持不变  

例如：

```perl
chown 1001, 1001, glob '*.log';  # 第一个1001是user位，第二个1001是group位
chown -1, 1002, 'a.log';         # uid不变
```

如果想按照用户名、组名来设置，使用getpwnam和getgrnam函数：

```perl
my $uid = getpwnam 'longshuai' or die "bad user";
my $gid = getgrnam 'longshuai' or die 'bad group';
chown $uid, $gid, glob '*.log';
```

## 修改文件时间戳属性：atime/mtime

在Unix系统中，要求操作系统维护atime/mtime/ctime三种文件的时间戳属性：  

- atime：access time，文件最近一次被访问时的时间戳  
- mtime：modify time，文件内容最近一次被修改时的时间戳  
- ctime：inode change time，文件inode数据最近一次被修改的时间戳  

touch命令可以修改atime和mtime，ctime是操作系统自身维护的，无法通过上层命令工具直接修改。

Perl的`utime`函数也可以修改文件时间戳，语法如下：

```perl
utime(Atime, Mtime, FILE_LIST)
```

- 只能修改atime和mtime属性  
- 会直接发起系统调用，所以失败时会设置`$!`  
- atime和mtime可以同时定义为undef，表示修改为当前时间  
  - 但如果只有一个undef，则对应位置的时间戳会设置为1970年  
- 它的返回值是成功修改的文件数量  
- 如想保持某项时间戳不变，可用stat函数取出当前时间戳值保存下来  

例如，下面修改一堆文件的atime为当前时间，mtime为前一小时时间：  

```perl
my $atime=time;
my $mtime=$atime - 60*60;
utime $atime, $mtime, glob '*.log';
```

实现touch文件的等同功能：将文件时间戳设置为当前时间：

```perl
utime undef, undef, 'a.txt'
    or die "touch file failed: $!";
```

如果只想修改atime，不想修改mtime，则使用stat函数先将当前的mtime属性值取出保存下来：

```perl
my $mtime = (stat('a.txt'))[9];
utime time, $mtime, 'a.txt'
    or die "touch failed: $!";
```

其中`stat(FILE)[9]`对应的属性是mtime，`stat(FILE)[8]`对应的属性是atime。