## Find搜索文件

Perl的glob通配操作无法递归目录搜索文件，如果要递归搜索，使用`File::Find`模块或者更友好一些的`File::Find::Rule`模块。此处仅介绍`File::Find`模块的用法。

`File::Find`模块有两个函数`find()`和`finddepth()`，后者是前者的一种特殊用法。

```perl
use File::Find;
find({ wanted => \&proc, follow => 1 }, @dirs_to_search);

find(\&wanted, @dirs_to_search);
sub wanted { ... }
 
finddepth(\&wanted, @dirs_to_search);
sub wanted { ... }
```

`find()`有两种用法：

- (1).在第一个参数位置指定hash选项，并在hash中指定搜索行为  
- (2).在第一个参数位置指定子程序引用，这种方式表示采用默认行为  

在wanted子程序中，有三个特殊的变量：

- `$File::Find::name`：保存了当前搜索到的文件的完整路径(并非绝对路径，而是从初始搜索目录开始的路径)  
- `$File::Find::dir`：保存了当前搜索到的文件的目录部分  
- `$_`：保存了当前搜索到的文件的文件名部分  

在wanted子程序中，有几个注意事项：

- 可使用return来跳过当前文件不处理  
- 可设置`$File::Find::prune=1`来跳过整个子目录  
- 子程序中可能会多次测试文件的属性，可使用缓存的文件句柄`_`进行文件属性测试  

以find()第二种用法为例，演示find的常规用法。

示例一：搜索某目录内的txt文件。

```perl
use File::Find;

my @files;
my @dirpath=qw(/home/user1/);

# 搜索txt文件
find(sub {
           if (-f $File::Find::name and /\.txt$/){
             push @files, $File::Find::name;
           }
      }, @dirpath);

print join "\n", @files;

# 或者
sub wanted {
  push @files, $File::Find::name if (-f $File::Find::name and /\.txt$/);
}
find(\&wanted, @dirpath);

# 或者
sub wanted {
  return unless -f;  # 测试$_
  return unless /\.txt$/;
  push @files, $File::Find::name;
}
find(\&wanted, @dirpath);
```

示例二：搜索某目录内的子目录文件。

```perl
my @files;
my @dirpath=qw(/home/user1);
find(sub {
        push @files, $File::Find::name if (-d); 
      }, @dirpath);

print join "\n",@files;
```

示例三：搜索最近一天内修改过的文件。

```perl
sub wanted {
  return unless -f; 
  return unless int(-M _) < 1; # 测试对$_缓存下来的文件句柄
  push @files, $File::Find::name;
}
```

示例四：搜索最近30分钟内修改过的文件。

```perl
sub wanted {
  return unless -f;
  return unless (-M _) < 0.5/24;
  push @files, $File::Find::name;
}
```

示例五：跳过某子目录。

```perl
sub wanted {
  $File::Find::prune = 1 if /some_name/;
  push @files, $File::Find::name;
}
```

再回头来看find()的第一种用法。

采用find()第一种方法时，常见的hash选项如下：  

- no_chdir：如果搜索到的是子目录，决定是否要进入子目录
  - no_dir=1：表示不进入子目录，此时，`$File::Find::name`、`$File::Find::dir`、`$_`的值是从初始目录开始的完整路径  
  - no_dir=0：默认值，表示进入子目录，此时`$File::Find::name`是完整路径，`$File::Find::dir`是从初始目录开始到该子目录为止的路径、`$_`的值文件名(basename)  
- wanted：指定每次搜索到文件时执行的操作，是一个子程序的引用  
- bydepth：找到子目录时，表示先处理目录内的文件，最后才来处理该子目录自身。`finddepth`等价于指定了bydepth=1的find  
- preprocess：开始搜索每个初始目录前执行的操作，是一个子程序引用  
- postprocess：在搜索完每个初始目录之后执行的操作，是一个子程序引用，常用来统计报告所搜索目录的总文件大小  
- follow：是否要跟踪软链接，默认follow=0，即不跟踪  

关于no_chdir时三个变量的值，参考如下示例数据：

```perl
#               $File::Find::name  $File::Find::dir  $_
#  no_chdir=>0  /                  /                 .
#               /etc               /                 etc
#               /etc/x             /etc              x
#   
#  no_chdir=>1  /                  /                 /
#               /etc               /                 /etc
#               /etc/x             /etc              /etc/x
```