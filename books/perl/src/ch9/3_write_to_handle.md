## 打开文件句柄：写入数据

以可写方式(覆盖写方式或追加写方式)打开文件后，可使用print、printf或say来向指定的文件句柄中写入数据。

```perl
# 以追加写方式打开文件
open my $fh, ">>", "/tmp/a.log";

# 通过文件句柄向文件尾部追加一行数据，不带换行符
print $fh "hello world 1";

# 再次使用printf向文件中追加一行数据
printf $fh "%s\n", "hello world 2";

# 使用say追加一行数据
say $fh "hello world 3"
```

使用非变量文件句柄写入数据也是一样的：

```perl
open FH, ">", "/tmp/b.log";

# 覆盖写入
print FH "hello world 1";
printf FH "%s\n", "hello world 2"; # 追加到行尾
say FH "hello world 3"   # 追加到行尾
```

需要注意，以覆盖写方式打开文件时，只会在打开的时候清空文件，之后的每次写入都根据文件偏移指针的位置来写入，而不是每次写入都会覆盖之前已经写入完成的数据。

此外，使用标量变量方式的文件句柄进行写入时，容易产生歧义。例如，下面的print操作是向终端输出`$fh`的值还是向`$fh`文件句柄中写入空字符串数据呢？

```perl
print $fh;
```

为了解决这种歧义，Perl允许使用大括号包围print/printf/say后的文件句柄：

```perl
print {$fh} "hello world 1";
say {FH} "hello world 2";
```

### 默认的输出文件句柄

如果print/printf/say没有指定要向哪个文件句柄写入数据，默认是向STDOUT输出。即，下面是等价的：

```perl
print "hello world";
print STDOUT "hello world";
```

STDOUT是perl中一个预定义的文件句柄，它代表标准输出。默认的标准输出的目标是终端屏幕，因此，print/printf/say默认会将要输出的数据打印在屏幕之上。

可使用perl提供的`select`函数来选择其他文件句柄作为默认输出的文件句柄：

```perl
# 选择$fh作为默认的输出文件句柄
select $fh;
# 向$fh写入数据
print "hello world";
```

select在选择默认文件句柄时，会返回当前的默认文件句柄。因此，下面的代码表示设置默认的文件句柄为`$fh`，同时备份当前默认的文件句柄：

```perl
my $oldfh = select($fh);
```

修改默认文件句柄后，在一些操作之后，通常会恢复原来的默认文件句柄。惯例的操作为：

```perl
my $oldfh = select($fh); ... ; select($oldfh);
```

