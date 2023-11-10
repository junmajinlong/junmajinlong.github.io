## 特殊的文件句柄

Perl支持一些特殊的、已经预定义的文件句柄，包括：STDIN、STDOUT、STDERR、ARGV和DATA。

其中：

- STDIN、STDOUT和STDERR分别代表标准输入、标准输出以及标准错误  
- ARGV代表由perl命令行参数列表各文件组成的文件句柄  
- DATA代表perl脚本程序文件内位于`__END__`或`__DATA__`之后的内容组成的文件句柄  

特殊地，Perl还支持菱形操作符内不指定任何文件句柄的读取操作`<>`，这表示空文件句柄。

### STDIN

STDIN是预定义的标准输入文件句柄，STDIN是只读文件句柄，不能向STDIN写入数据。

当使用`<STDIN>`时，表示从标准输入中读取数据。

```perl
 while(<STDIN>){
   chomp;
   say $_;
 }
```

标准输入的数据默认来自于用户在终端的输入，也可能来自于shell中的管道、shell的标准输入重定向，等等。例如，对于名为a.pl的Perl脚本，代码内容如上，运行它：

```bash
# 等待用户在终端输入
$ perl a.pl

# 来自于管道的标准输入
$ echo "hello world" | perl a.pl

# 来自于shell标准输入重定向
$ perl a.pl <a.log
```

### STDOUT

STDOUT是预定义的标准输出文件句柄。print/printf/say的输出默认都是写入STDOUT。

```perl
# 写向标准输出，两者等价
print "hello world\n";
print STDOUT "hello world\n";
```

默认情况下，标准输出的输出目标是终端屏幕，但可以在shell中改变标准输出的输出目标。例如，输出给管道、输出到文件等等。

```bash
# 标准输出的目标是管道
$ perl a.pl | cat

# 标准输出的目标是文件
$ perl a.pl >/tmp/a.log
```

### ARGV

ARGV代表当前正在读取的来自命令行参数的文件对应的文件句柄，它是只读文件句柄。

例如，`perl a.pl a.log b.log`，如果在a.pl中使用了ARGV文件句柄，那么在处理a.log文件时，它对应的是a.log文件对应的文件句柄，当处理b.log时，它对应的是b.log文件对应的文件句柄。

通常，`<ARGV>`会简写成不带任何字符的`<>`，但注意，它们不等价，参考前文对`<>`的解释。

例如，从命令行指定的参数文件中读取数据：

```perl
#!/usr/bin/env perl

while(<ARGV>){
  print "line number: $., content: $_";
}
```

执行：

```bash
$ seq 3 5 >a.log
$ seq 6 10 >b.log
$ perl a.pl a.log b.log
line number: 1, content: 3
line number: 2, content: 4
line number: 3, content: 5
line number: 4, content: 6
line number: 5, content: 7
line number: 6, content: 8
line number: 7, content: 9
line number: 8, content: 10
```

从结果可以看出，`<>`或`<ARGV>`在切换文件时，是不会自动切换行号的。如果想要在切换文件时重置行号，官方文档给出了一个方案：

```perl
while(<>){
  print "$.\t$_";
} continue {
  close ARGV if eof;
}
```

也可以很容易地自己实现，遍历`@ARGV`即可，因为它保存了所有的命令行参数：

```perl
for(@ARGV){
  open my $file, $_;
  $. = 1;
  while(<$file>){
    print "$.\t$_";
  }
}
```

注意，要区分几种ARGV表达的不同含义：

- `@ARGV`：保存了所有的命令行参数，是一个数组  
- `$ARGV[n]`：表示访问`@ARGV`数组的第n+1个元素  
- `ARGV`：当前正在处理的命令行参数文件(即`@ARGV`中的元素)对应的文件句柄  
- `$ARGV`：当前正在处理的命令行参数文件(即`@ARGV`中的元素)对应的文件句柄对应的文件名  

```perl
push(@ARGV, 'a.log');
push(@ARGV, 'b.log');
while(<>){    # 将自动打开a.log，读完a.log后自动打开b.log
  if(eof){
    say "$.: $ARGV";  # 分别输出a.log和b.log
    close ARGV;
  }
}
```

### 空文件句柄

默认情况下，空文件句柄读取操作`<>`表示读取标准输入，此时`<>`等价于`<STDIN>`。

例如，读取标准输入中的所有数据并输出行号和内容：

```perl
while(<>){
  print "$.\t$_";
}
```

但`<>`不总是等价于`<STDIN>`。实际上，如果给定了命令行参数，空文件句柄读取操作`<>`将优先打开命令行参数代表的各个文件，此时`<>`等价于`<ARGV>`。

```bash
$ perl a.pl a.log b.log
```

也就是说，如果指定了命令行参数时(严格来说是`@ARGV`数组不为空时)，`<>`将等价于`<ARGV>`，表示从命令行参数对应的文件中读取数据，即使此时向标准输入中传递了数据，也不会读取标准输入。

如果指定了命令行参数的同时，也想要读取标准输入，此时应该在命令行参数合理的位置上使用`-`来代表标准输入的文件名。

```bash
# <>将先读取标准输入，再读取a.log，再读取b.log
$ perl a.pl - a.log b.log

# <>将先读取a.log，再读取标准输入，再读取b.log
$ perl a.pl a.log - b.log
```

### DATA

DATA是一个伪文件句柄。

Perl允许直接在当前源代码文件的最尾部定义数据，这些数据要么是用来测试，要么是临时数据。定义的方式如下：

```perl
... some perl code ...

# 从__DATA__或__END__开始的数据都将被DATA文件句柄读取，直到文件结尾
__DATA__
...一些待读取数据...
```

当perl编译器遇到`__DATA__`或`__END__`了，就知道这个源文件的代码部分到此结束，下面的数据都将作为当前文件内有效的DATA文件句柄的数据流。

例如：
```perl
#!/usr/bin/perl

while(<DATA>){
  chomp;
  print "read from DATA: $_\n";
}

__DATA__
first line in DATA
second line in DATA
third line in DATA
last line in DATA
```


### Inline::Files

DATA伪文件句柄的一个缺点是从遇到`__DATA__`或`__END__`起直到文件的尾部，都属于DATA文件句柄的内容，也就是说在源代码文件中只能定义一个伪文件句柄。

在CPAN上有一个`Inline::Files`模块，它可以在同一个源代码文件中定义多个伪文件句柄。需要先安装：
```
cpan install Inline::Files
```

例如：
```perl
use Inline::Files;
use 5.010;

say "@main::FILE1";
say "@main::FILE2";

while(<FILE1>){
  say "$_";
}

while(<FILE2>){
  say "$_";
}

__FILE1__
first line in FILE1
second line in FILE1
third line in FILE1
__FILE2__
first line in FILE2
second line in FILE2
third line in FILE2
```

它像ARGV一样，在运行程序初始阶段就打开这些虚拟文件句柄，并将每个虚拟文件句柄保存到`@<PACKNAME>::<HANDLE>`中。例如，上面的是示例是在main包中定义了FILE1和FILE2两个文件句柄，那么这两个文件句柄将保存到`@main::FILE1`和`@main::FILE2`中，并在处理某个文件句柄的时候，将其保存到标量`$main::FILE1`或`$main::FILE2`中。

可以同时定义多个名称相同的虚拟文件系统。例如：
```
__FILE1__
...
__FILE2__
...
__FILE1__
...
```

这时在`@<packname>::FILE1`数组中就保存了两个元素，当处理第二个FILE1的时候，将自动重新打开这个文件句柄。

一般来说，这些就够了，更多详细的用法请参见官方手册：[Inline::Files](https://metacpan.org/pod/Inline::Files)。