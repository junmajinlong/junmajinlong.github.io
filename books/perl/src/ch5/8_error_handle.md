## Perl处理异常

Perl自带了`die`函数，用来报错并退出程序，相当于其他语言中的raise。还自带了`warn`函数，用法和die类似，它用来发出警告信息，但不会退出。

```perl
# 如果打开文件失败，报错退出
if ( ! open LOG "<" "/tmp/a.log" ){
  die "open file error: $!";
}
```

输出的错误信息如下：

```
open file error: No such file or directory at 1.pl line 4.
```

上面的特殊变量`$!`表示Perl调用系统调用出错时由操作系统收集并反馈给Perl的错误信息，即`No such file or directory`。并非所有的错误都会收集到`$!`变量中，只有涉及到系统调用且出错时，才会设置`$!`。例如：

```perl
if ( @ARGV < 2 ){
  die "wrong! help me!";
}
```

注意，die和warn默认会输出程序名称和行号，但如果在错误消息后面加上`\n`换行符，则不会报告程序名称和行号。

```perl
if ( @ARGV < 2 ){
  die "wrong! help me!\n";
}
```

### croak和carp

Perl自带的die和warn有时候并不友好，它们只会报告代码出错的位置，即哪里使用了die或warn，就报告这个地方有问题。

例如，文件第11行调用函数fff()，函数fff()里的第3行代码(位于文件第20行)错误，使用die和warn时将报告第20行出错，这不利于追踪是在何处调用代码导致的错误。

`Carp`模块提供的croak和carp函数提供了更细致的错误追踪功能，用法分别对应die和warn，区别仅在于它们会展示更具体的错误位置。

例如：

```perl
use Carp 'croak';
sub f{
  croak "error in f";
}

f;
```

报告的错误信息：

```
error in f at first_perl.pl line 73.
        main::f() called at first_perl.pl line 76
```

### eval错误处理

使用die或Carp的croak，都将报错退出程序，如果不想退出程序，可以使用eval来处理错误：当eval指定的代码出错时，它将返回undef而不是退出。

eval有两种用法，一种是字符串作为参数，另一种是语句块作为参数。无论是字符串参数还是语句块参数，参数部分都会被当作代码执行一次。例如：

```perl
eval 'say "hello world"';
eval {say "hello world"};
```

如果eval参数部分执行时没有产生错误，则eval的返回值是最后一条被执行语句的计算结果，并且特殊变量`$@`为空。

如果eval参数部分执行时产生了错误，则eval的返回值为undef或空列表，同时将错误信息设置到`$@`中。

因此，可以通过eval结合`$@`来判断程序是否出错：

```perl
eval {...};
if (my $err = $@){
  ...处理错误：handle_error...
}
```

之所以将`$@`保存起来，是因为handle_error可能也有eval，这时handle_error的eval将覆盖之前的`$@`。

更合理的eval使用方式是：

```perl
my $res;
my $ok = eval { $res = some_func(); 1};
if ($ok){
  ...代码正确...
} else {
  ...eval代码出错...
  my $err = $@;
  ...
}
```