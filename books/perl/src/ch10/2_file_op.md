## 文件路径处理

Perl没有内置函数可以获取当前工作目录，需要使用其他模块提供的功能。例如`Cwd`模块(current work directory)提供了cwd函数和getcwd函数可获取当前工作目录的绝对路径，可随便选择使用哪个：

```perl 
use Cwd;
print cwd();
print getcwd();
```

`Cwd`模块还提供了路径处理的功能：`abs_path`或`realpath`可获取绝对路径，它们是别名关系。要使用这两个功能，需要手动导入：

```perl
use Cwd qw(:abs_path);
print abs_path('../code');
```

使用`chdir()`内置函数可以切换当前工作目录：

```perl
chdir("/tmp/test/");
```

但是`chdir`内置函数不能将`~`扩展为家目录，并且chdir的切换是全局的，意味着在某处切换会直接影响全局。

`File::chdir`提供了可局部切换工作路径的方式，建议使用。

先安装：

```shell
$ cpan install File::chdir
```

使用`File::chdir`的示例：

```perl
use File::chdir;

# 全局切换到/var/log/apt目录
$CWD = "/var/log/apt";
say getcwd();
{
  # 局部作用域内切换到/tmp
  local $CWD = '/tmp';
  say getcwd;
}
# 外面仍然是/var/log/apt内

# 可直接修改变量来切换工作目录
# 切换到/tmp/log/apt
$CWD =~ s/var/tmp/;
say cwd;
```

## 创建和删除文件

没有直接创建文件的函数，但可以通过open以写模式打开文件再关闭文件的方式来创建文件。

```perl
sub touch {
  my $path = shift;
  return if -e $path;
  open my $fh, '>', $path
    or die "create file failed: $!";
  # close $fh;  退出函数，可省略close操作
}

touch "/tmp/a.log";
```

删除文件使用`unlink`函数：

```
unlink FILE  # 删除单个文件
unlink LIST  # 删除列表中的所有文件
unlink       # 删除$_对应的文件
```

unlink返回删除成功的数量，如果想要知道哪些文件删除失败，应该迭代遍历然后逐个删除并查看`$!`获取删除失败的原因。

```perl
unlink '/tmp/a.log';
unlink @filelists;
unlink glob('*');   # 删除当前目录下所有非隐藏文件
```

注意，unlink不能删除目录。

下面是两个示例：使用Perl创建大量小文件，和删除目录中大量小文件。

在当前目录下创建大量随机大小的小文件：

```perl
# 第一个参数是要创建的小文件数量
# 第二个参数是小文件的随机大小(kb)
sub create_many_small_files {
  my $n=shift;
  my $max_size=1024 * shift;
  for(1..$n){
    open my $f, ">", "$_.txt" or die "open failed: $!";
    print {$f} "0" x int(rand($max_size));
    close $f or die "close failed: $!";
  }
}
# 例如，创建50W个随机0-8K大小的小文件
create_many_small_files 500000, 8
```

删除大量小文件：

```shell
$ perl -e 'unlink <"*.log">'
```

## 创建和删除目录

使用`mkdir`函数来创建目录，创建目录时可指定权限属性。如果创建失败，将返回false并设置`$!`变量。

```perl
mkdir "/tmp/test1";
mkdir;                # 等价于mkdir "$_"
mkdir "/tmp/test2", 0755   # 权限不能加引号包围，它是8进制数值
    or die "Can't create directory: $!";
```

mkdir不会递归创建缺失的目录，如果要同时创建缺失的目录，使用`File::Path`模块中的mkpath函数。稍后再介绍该模块的用法。

删除目录使用`rmdir`函数，但是rmdir函数只能删除空目录：

```perl
rmdir "/tmp/test";
```

`File::Path`提供了递归创建缺失目录和删除非空目录的方式：  

- mkpath和别名make_path：递归创建缺失的上级目录  
- rmtree和别名remove_tree：删除目录树，即删除非空目录  

对于mkpath，用法如下：

```perl
make_path(dir1,dir2,...,{opts})

#opts可以是以下几种：
mask  => NUM       # mask和mode是同义词，NUM指定八进制权限值，
mode  => NUM       # 这种方式指定权限值受umask影响，若目录已存在，则不修改

chmod => NUM       # 直接赋予一定权限值，不受umask影响，若目录已存在，则不修改

verbose => $bool   # 是否输出详细信息，默认不输出

error => \$err     

owner => $owner    # 这3条都表示为创建的目录设置所有者，如果已存在，则不设置
user  => $user     # 可以使用username，也可以使用uid，但如果username无法
uid   => $uid      # 映射为uid，或者uid不存在，或者无权限的时候，将报错

group => $group   # 设置所属组，处理方式和上面所有者的处理方式一样
```

例如：

```perl
use File::Path qw(make_path);
make_path "/test/foo/bar";    # 一次性创建3级目录
make_path "/test/foo1/bar1",{
  chmod => 0777,
  verbose => 1
}
```

对于rmtree，用法如下：

```perl
rmtree($dir1, $dir2,..., \%opt)

#opts可以是以下几种：
verbose => $bool     # 是否显示删除信息，默认不显示
safe    => $bool     # 删除时跳过无法删除的目标。例如/proc下很多无法删除
keep_root => $bool   # 是否保留顶级目录，即只删除子文件和子目录，目录自身不删除
result => \$res
error => \$err
```

例如：

```perl
use File::Path qw(rmtree);

# foo1整个被删除
rmtree '/test/foo1', {verbose => 1};

# foo2的子目录和子文件被删除，foo2被保留
rmtree '/test/foo2', {verbose => 1, keep_root => 1};
```

