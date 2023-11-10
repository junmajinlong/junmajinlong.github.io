## 文件和目录的复制、移动

### 文件拷贝

Perl中使用`File::Copy`模块的copy函数或cp函数进行文件的拷贝，它不支持目录的拷贝。

该模块中的copy和cp的用法相同，但行为有所不同：copy不会保留源文件的属性，而是遵循目标位置的默认属性，cp会尝试保留源文件属性。

```perl
use File::Copy qw(copy cp);
copy '/tmp/a.pl', '/mnt/g/桌面/a.pl' or warn 'copy failed: $!';
cp '/tmp/a.pl', '/mnt/g/桌面/a.pl' or warn 'copy failed: $!';
```

copy和cp拷贝时，有以下几个注意事项：  

- copy和cp拷贝时，是打开源文件获取其中内容，然后在目标位置创建文件并填充内容。因此，copy和cp可以跨文件系统拷贝文件  
- 如果目标文件已存在，则会直接覆盖目标文件，不会给出提示信息，除非它是只读文件  
- 如果目标位置不存在，比如父目录不存在，则报错  
- 如果给定的目标位置是目录，则文件将被拷贝到该目录下  

如果要拷贝目录，使用`File::Copy::Recursive`模块，稍后会给出介绍。

### 文件移动

`File::Path`模块还提供了move和mv函数(别名关系)用来实现文件的移动，同样，它们不支持目录的移动。

```perl
use File::Copy qw(move mv);
move("/dev1/sourcefile", "/dev2/destinationfile");
mv("/dev1/sourcefile", "/dev2/destinationfile");
mv("/dev1/sourcefile" => "/dev2/destinationfile");
```

move或mv移动文件时，有以下几个注意事项： 

- 只能移动文件，不能移动目录  
- 如果目标位置和源文件位置在同一个文件系统，则只是简单的重命名，速度非常快  
- 如果不在同一个文件系统，则是拷贝操作，并在拷贝完成后删除源文件，如果拷贝的过程中被中断，源文件和不完整的目标文件都将被保留  
- 如果目标位置是一个目录，则文件被移动到这个目录，如果目标位置是一个文件，则移动后将使用该名称作为文件名  
- 如果目标位置父目录不存在，则报错   
- 它们都会尝试保留源文件的属性   

### 文件重命名

Perl内置了一个`rename`函数用来文件重命名，它功能比较有限，无法做复杂的重命名，无法跨文件系统重命名，但可以移动文件和目录。

```perl 
rename '/tmp/empty', '/home/abc/not_empty' 
  or warn 'rename failed: $!';
```

虽然自带的rename函数功能受限，但可以使用`File::Rename`提供的重命名功能：

```perl
use File::Rename qw(rename);   # hide CORE::rename

# rename( FILES, CODE [, VERBOSE])
rename \@ARGV, sub { s/\.pl\z/.pm/ }, 1;
rename \@ARGV, '$_ = lc';
```

`File::Rename::rename`的参数：  

- 第一个参数必须是一个数组的引用   
- 第二个参数要么是字符串，要么是子程序引用，只要在这部分代码里修改`$_`的值，就意味着修改文件名称  
- 第三个可选参数表示是否输出详细信息  

`File::Rename`还提供了一个Perl版本的rename命令，重命名的功能非常强，可支持正则替换，支持多表达式。例如：

```shell
$ rename 's/\.bak$//' *.bak
$ rename 'y/A-Z/a-z/' *
$ rename 'y/A-Z/a-z/;s/^/my_new_dir\//' *.*
$ rename -E 'y/A-Z/a-z/' -E 's/^/my_new_dir\//' *.*
```

Perl版本的rename命令可能已经安装好了，可通过`man rename`来查看rename是不是perl版本的。如果没有安装，可手动安装：

```shell
$ cpan install File::Rename
```

## File::Copy::Recursive

`File::Copy`模块提供的copy和move函数，都无法针对目录进行操作，如有需要，可使用`File::Copy::Recursive`来拷贝。

```perl
use File::Copy::Recursive qw(rcopy rmove
                             rcopy_glob
                             rmove_glob);
 
# 可拷贝文件或目录
rcopy($orig, $new) or die $!;
rmove($orig, $new) or die $!;

# 支持通配的拷贝和移动，采用File::Glob::bsd_glob()的通配规则
rcopy_glob("orig/stuff-*", $trg) or die $!;
rmove_glob("orig/stuff-*", $trg) or die $!;
```

它们都会尝试保留源文件或源目录的属性。

