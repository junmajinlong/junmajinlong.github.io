---
title: Perl文件、目录常用操作
p: perl/perl_file_dir_op.md
date: 2019-07-06 17:37:58
tags: Perl
categories: Perl
---

# Perl文件、目录常用操作

注意，这些操作的对象是文件名(相对路径/绝对路径)，而非文件/目录句柄，句柄只是perl和文件系统中文件的关联通道，而非实体对象。

## 创建文件

在unix类操作系统中有一个touch命令可以非常方便的创建文件，还能批量创建一些名称规律的文件。但实际上touch的主要介绍中却是【修改文件时间戳】，创建文件只不过是它的辅助能力。如果没有touch命令，如何在shell环境下创建文件？最佳方式是通过重定向的方式。

在perl中没有touch类似的功能，所以原始地只能通过open打开输出类的文件句柄和输出操作来创建文件(同时也可以写入数据)。
```
open FH,">>/tmp/cc.log";
print FH;       # 创建空文件
close FH;
```

如果想批量创建命名规范的文件，可以将创建操作放进循环中进行迭代。例如创建`cc{1..10}.log`这10个文件：
```
foreach (1..10){
    open FH,">cc${_}.log";
    print FH;
    close FH;
}
```

当然，perl可以通过system或反引号或exec来和shell进行交互。例如：
```
`touch dd{1..10}.log`;
```
但是如果想要批量创建的文件不能用一个touch(或少数几个)来创建的话，还是建议采用perl的创建方式，因为每次和shell交互touch都fork一次perl进程并解析执行shell语句，文件数量多时，这样的效率很一般。不过毕竟不太可能用到这种情形。

## 删除文件

在unix系统里，使用rm删除文件/目录，但它内部调用的是unlink函数。

在perl中删除文件使用perl自带的unlink函数，它也会调用操作系统的unlink函数来删除文件。它可以接一个文件元素，也可以接一个列表，或者说它的参数上下文就是列表。

但是注意，**unlink无法删除目录**，要删除目录，见下文。

```
unlink @lotfiles;            # 删除数组中的文件
unlink 'cc.log';             # 删除单个文件
unlink 'cc1.log','cc2.log';  # 删除文件列表
unlink qw(cc3.log cc4.log);  # 删除文件列表
unlink glob dd*.log;         # 通配文件名删除
```
关于文件名通配详细内容，见后文。

需要注意，unlink有返回值，返回的是成功删除的文件数量。

所以，unlink删除3个文件时，如果它的返回值为3，表示全删除成功了，如果返回值为0表示一个都没删除，但如果返回的是1或者2，我们就无法判断哪些文件删除成功，哪些文件删除失败。这时需要在循环中一个一个文件地迭代删除操作，并给出错误提示。

```
`touch dd{1..10}.log`;
foreach (1..10){
    unlink "dd${_}.log"
        and ++$count       # 注意，++放在变量的前面
        or  warn "Can't remove file dd${_}.log: $!";
}
print "removed $count files\n";
```

注意上面的and语句中，自增`++$count`的自增符号放在变量的前面，如果放在后面，会因为初始化$count为0，`$count++`表达式返回的值为0(但$count加完后返回1)而执行or语句。也就是说，删除第一个文件dd1.log时也会报告警告信息。


## 创建目录

- 创建目录可以使用mkdir，它会发起系统调用并返回布尔值：成功/失败，失败时会设置`$!`
- mkdir时可以同时指定文件权限，权限需要指定4位的8进制。如果使用变量传递这个权限位，应当使用oct()函数保护它不被当作十进制数
- mkdir不能递归创建目录(unix系统下`mkdir -p`的功能)，可以使用File\::Path里的make_path函数或mkpath函数，他们等价

例如，创建一个目录
```
mkdir "/tmp/test1";
mkdir;                # 等价于mkdir "$_"
mkdir "/tmp/test2",0755   # 权限不能加引号包围，它是8进制数值
    or die "Can't create directory: $!";
```
如果是使用变量传递权限位，应当使用oct()函数来保护它作为8进制数。
```
$perm="0755";
mkdir "/tmp/test3",oct($perm);
```

mkdir函数无法递归创建目录。也就是说，当要创建的目录的上级目录不存在时，mkdir函数将失败。如果想递归创建目录，可使用File\::Path里的make_path函数或mkpath函数，他们是等价的。

这个函数的语法是：
```
make_path(dir1,dir2,...,{opts})

opts可以是以下几种：
mask  => NUM       # mask和mode是同义词，NUM指定八进制权限值，
mode  => NUM       # 这种方式指定权限值受umask影响，若目录已存在，则不修改

chmod => NUM       # 直接赋予一定权限值，不受umask影响，若目录已存在，则不修改

verbose => $bool   # 是否输出详细信息，默认啥也不输出

error => \$err     

owner => $owner    # 这3条都表示为创建的目录设置所有者，如果已存在，则不设置
user  => $user     # 可以使用username，也可以使用uid，但如果username无法
uid   => $uid      # 映射为uid，或者uid不存在，或者无权限的时候，将报错

group => $group   # 设置所属组，处理方式和上面所有者的处理方式一样
```

```
use File::Path qw(make_path);
make_path "/test/foo/bar";    # 一次性创建3级目录
make_path "/test/foo1/bar1",{
    chmod => 0777,
    verbose => 1
}
```

当然，和shell交互来创建目录也非常方便：
```
`mkdir -p /test/a/b/c`;
```

## 删除目录

- 删除空目录用rmdir，且只能删除空目录
- 要删除非空目录，可以使用File\::Path里的rmtree函数或remove_tree函数，它们等价
- 删除目录时，可随意对待尾随斜线问题，因为perl会自动删除尾随斜线，以满足多种平台对尾随斜线的定义标准

例如，删除一个目录
```
rmdir "/tmp/test";
```

因为rmdir无法删除非空目录，所以要删除非空目录，可以采用File\::Path模块的rmtree函数。

rmtree的语法：
```
rmtree($dir1, $dir2,..., \%opt)

opts可以是以下几种：

verbose => $bool     # 是否显示删除信息，默认啥也不显示
safe    => $bool     # 删除时，跳过无法删除的对象。例如虚拟文件系统(VMS,如/proc)下有很多是无法删除的
keep_root => $bool   # 设置为真值时，保留顶级目录，也就是说自删除目录内的文件和子目录，顶级目录自身不删除
result => \$res
error => \$err
```

```
use File::Path qw(rmtree);
rmtree '/test/foo1',{verbose => 1};
```

由于perl无法向shell一样可以直接使用`*`通配，所以如果想删除目录内的文件和子目录，而保留foo1目录自身，应该设置keep_root选项，而不是用`/test/foo1/*`的方式来删除：
```
rmtree '/test/foo1',{verbose => 1,keep_root => 1};
```

实在想通配删除，可以使用glob来通配，关于通配，见后文：
```
rmtree((glob '/test/foo1/*'),{verbose => 1});
```

另一种保留目录自身的删除方式是遍历顶级目录，然后迭代删除遍历出来的每个子项：
```
foreach (</test/foo1/*>){                # 括号内等价于 glob "/test/foo1/*"
    -d $_ ? rmtree $_ : unlink $_;
}
```

图方便的话，直接和shell交互也非常方便。
```
`rm -rf /test/foo1/*`;
```


## chdir切换目录

- chdir用于切换到指定目录，如果不给参数，则回到家目录
- chdir因为直接发起系统调用，所以报错时会设置`$!`
- chdir不能使用shell里的`~`来表示家目录

```
chdir /tmp or die "Can't change dir to /tmp: $!";
```


## 获取当前工作目录

perl没有内置函数可以直接获取当前工作目录，但`Cwd`模块和更通用的`File::Spec`模块都提供了相关工具获取当前路径。

例如，Cwd模块：
```
use Cwd(getcwd,cwd);
print getcwd();
print cwd();
```

cwd()比getcwd()要更可移植，它和pwd命令基本一致。注意，cwd()会移除尾随斜线。


## 修改权限

- chmod函数修改文件、文件列表的权限值，它会直接发起系统调用，所以错误的话会设置`$!`
- 只接受8进制的权限值，不允许使用rwx的权限字符
- 它返回成功设置权限的数量

例如，设置目录、文件的权限：
```
chmod 0700,qw(/test/foo /test/foo1/a.log);
chmod 0700,'/test/foo','/test/foo1/a.log';
```

## 修改所有者、所属组

- chown可以同时修改文件/目录、文件/目录列表的所有者、所属组，它会直接发起系统调用，所以报错时会设置`$!`
- chown只接受uid和gid作为参数，不接受username和groupname
    - 但在unix上显示的时候，还是会显示为username和groupname，这是操作系统的映射机制
    - 可以使用getpwnam和getgrnam函数将username/groupname映射为uid/gid
- 在uid或gid的参数位置上指定特殊值`-1`，表示该位置的所有者或所属组属性保持不变
- 它返回成功设置的文件数量

```
chown 1001,1001,glob '*.log';   # 第一个1001是user位，第二个1001是group位
chown -1,1002,'a.log';         # uid不变
```

如果想按照用户名、组名来设置，使用getpwnam和getgrnam函数：
```
defined(my $user = getpwnam 'longshuai') or die "bad user";
defined(my $group = getgrnam 'longshuai') or die 'bad group';

chown $user,$group,glob '*.log';
```

## 修改文件时间戳属性

关于文件的时间戳属性，这里简单说明下，如果需要详细了解，参看：http://www.cnblogs.com/f-ck-need-u/p/6995195.html#blog1.3

在unix系统中，要求操作系统维护atime/mtime/ctime三种文件的时间戳属性：
- atime：access time，文件最近一次被访问时的时间戳
- mtime：modify time，文件内容最近一次被修改时的时间戳
- ctime：inode change time，文件inode数据最近一次被修改的时间戳

unix的touch命令可以修改atime和mtime，ctime是操作系统自身维护的，无法通过上层命令工具直接修改。

Perl的utime函数也可以修改文件时间戳，语法如下：
```
utime(Atime,Mtime,FILE_LIST)
```

- 也只能修改atime和mtime属性
- 会直接发起系统调用，所以失败时会设置`$!`
- Atime和Mtime可以同时定义为undef，表示修改为当前时间
    - 但如果只有一个undef，则对应位置的时间戳会设置为1970年
- 它的返回值是成功修改的文件数量
- 如想保持某项时间戳不变，可用stat函数取出当前时间戳值保存下来

例如，下面修改一堆文件的atime为当前时间，mtime为前一小时时间：
```
$atime=time;
$mtime=$atime - 60*60;
utime $atime,$mtime,glob '*.log';
```

实现touch文件的等同功能：将文件时间戳设置为当前时间：
```
utime undef,undef,'a.txt'
    or die "touch file failed: $!";
```

如果只想修改atime，不想修改mtime，则使用stat函数先将当前的mtime属性值取出保存下来：
```
$mtime = (stat('a.txt'))[9];
utime time,$mtime,'a.txt'
    or die "touch failed: $!";
```

其中stat()[9]对应的属性是mtime，stat()[8]对应的属性是atime。

