---
title: Perl的上下文
p: perl/perl_context.md
date: 2019-07-06 17:37:47
tags: Perl
categories: Perl
---

# perl中的上下文


在perl中，很多地方会切换上下文。所谓上下文，它的**重点在于同一个表达式出现在不同地方，得到的结果不同**。换句话说，同一个表达式，它表达的值不是固定的。这就像是同一个单词，在不同语境下的意思不同。

例如，运算操作符决定数值是一个数字还是一个字符串。
```
2 * 3
2 x 3
```
`2 * 3`中的2和3都是数值，因为操作符`*`是算术运算符，它要求两边都是数字。而`2 x 3`中的2是字符串，3是数字，因为操作符`x`是这样要求的。

还有，对数组`@arr`的两种操作：
```
@arr=qw{perl,python,shell};
print @arr,"\n";    # 返回：perlpythonshell
print @arr."\n";    # 返回：3
```
使用逗号分隔`@arr`和`\n`是产生一个列表，这时的`@arr`会替换为该数组中的元素值。使用点号连接`@arr`和`\n`，这时点号要求两边的都是字符串，数组在这种环境下(标量上下文)返回的是它的元素个数，所以`@arr`返回一个数值(但其实是字符串)。

在perl解析表达式的时候，你要么希望它返回一个标量，要么希望它返回一个列表(其实还有很多种上下文，但至今无人知晓有多少种上下文，perl长老团也不知道)。所以perl中常见的两种上下文是：**标量上下文和列表上下文**，除此之外还有一个很常见的上下文类型：空上下文(void context)。  
1. 标量上下文：表达式在标量上下文中返回一个标量  
2. 列表上下文：期待表达式返回一个列表  
3. 空上下文：不使用表达式的返回结果，即返回结果被丢弃  

上下文不仅决定了表达式的返回结果类型，还限制了某些环境下只能使用某些上下文。比如，需要传递一个列表的时候却传递了一个标量，这可能会报错误。

例如：
```
42 + something;     # 这里的something必须是标量
sort something;     # 这里的something必须是列表
```

数组`@arr`为例：
```
@arr=qw(perl shell python);
@sorted=sort @arr;           # 列表上下文：返回(perl python shell)
@num=@arr + 42;              # 标量上下文：返回45
```

即便是赋值这种操作，都有不同的上下文：
```
@arr1=@arr;      # 列表上下文，赋值@arr给另一个数组@arr1
$arr1=@arr;      # 标量上下文，赋值@arr的元素个数给变量arr1
```

比较悲剧的是，无法总结一个通用的规则来解释什么时候用什么上下文，只能通过一些经验来感受它。

## 列表操作切换到标量上下文

那些操作列表的表达式(如sort)用在标量上下文会如何？没人知道会如何。不同的操作，返回的内容没有规律可言。

例如，对列表排序的sort操作放在标量上下文只会返回undef，reverse操作放在标量上下文则是返回字符的逆排序(先将所有元素按照字符串格式连接起来，再对整体进行反转)。
```
@arr=qw(perl python shell);
$sorted=sort @arr;         # 返回undef
$reversed=reverse @arr;    # perlpythonshell-->llehsnohtyplrep
print $sorted,"\n";
print $reversed,"\n";
```

以下是常见的上下文：
```
$var = something;             # 标量上下文
@arr = something;             # 列表上下文
($var1,$var2) = something;    # 列表上下文
($var1) = something;          # 列表上下文
```

以下是常见的标量上下文：
```
$var = something;
$arr[3] = something;
123 + something;
if (something) {...}
wihle(something) {...}
$var[something] = something;
```
以下是常见的列表上下文：
```
@arr = something;
($var1,$var2) = something;
($var1) = something;
push @arr,something;
foreach $var (something){...}
sort something;
reverse something;
print something;
```

需要注意的几点，将**数组赋值给标量变量，得到的是数组的长度(元素个数)，将列表赋值给标量变量，得到的是最后一个元素**，除了最后一个元素外，其它元素都被丢弃，也就是放进了void context。
```
@arr = qw(a b c d);
$x = @arr;           # 结果：$x=4

$y = qw(a b c d);    # 结果：$y=d，开启了warnings的情况下会警告
($y) = qw(a b c d);  # 结果：$y=d，不会警告
($a,$b,$c,$d) = qw(a b c d)  # 结果：$a=a,$b=b,$c=c,$d=d
```

## 标量操作切换到列表上下文

这种情况很简单，如果某个操作的返回结果是标量值，但却在列表上下文中，则直接生成一个包含此返回值的列表。

```
@arr = 6 * 7;    # 结果：@arr=(42)
@arr = "hello".' '.'world';  # 结果：@arr=("hello world")
```

但关于undef和空列表有一个陷阱：
```
@arr1 = undef;
@arr2 = ();
```

上面的undef是一个标量值，所以赋值后`@arr1=(undef)`，它不会清空数组；而`()`是空列表，它表示未定义的，所以赋值后`@arr2`被清空。


## 强制指定标量上下文

有时候如果想要强制指定标量上下文，可以使用伪函数scalar进行强制切换，它会告诉perl这里要切换到标量上下文。
```
@arr=(perl python shell);
print "How many subject do you learn?\n";
print "I learn ",@arr," subjects!\n";         # 错误，这里会输出课程名称
print "I learn ",scalar @arr," subjects!\n";  # 正确，这里输出课程数量
```

另一种切换为标量上下文的方式是使用`~~`，这是两个比特位取反操作，因为比特位操作环境是标量上下文，两次位取反相当于不做任何操作，所以将环境变成了标量上下文。这在写精简程序时可能会用上，而且因为`~~`是符号，可以和函数之间省略空格分隔符：
```
my @arr = qw(Shell Perl Python);
print ~~@arr;
print~~@arr;
```

还可以使用下面的技巧从列表上下文切换成标量上下文：
```
$var = () = expr
```

Perl中赋值操作总是先评估右边，所以上面等价于`$var = (() = expr)`。`() = expr`表示转换成列表上下文，使得expr以列表上下文的环境工作。最后的赋值操作，由于左边是`$var`，会将列表转换成标量上下文。

(如果不理解，暂时跳过)这种技巧比较常见的是`$num = () = <>`，因为`<>`在不同上下文环境下工作的方式是不一样的，这个表达式表示以列表上下文环境读取所有行，然后赋值给标量，所以赋值给标量的是列表的元素个数，也就是文件的行数。

它等价于`$num = @tmp = <>`。而且完全可以使用scalar()替代`scalar(@tmp = <>)`。

只有强制切换到标量上下文的伪函数scalar，没有切换到列表上下文的函数。因为根本用不到。

## 列表上下文中的STDIN

`<STDIN>`放在列表上下文时，会 **一次性** 读取所有输入(文件/键盘等)。一次性意味着大文件需要大量内存，一般400M的文件，perl可能会花上1G内存，因为perl会事先分配好富裕的空间避免事后问题。

例如：

```
@lines=<STDIN>;
foreach (@lines){
    print $_;
}
```

在目前来说，这正是我们所需的方式。可以读取每行，并对每行进行操作。但因为是一次性读取，对于大文件来说，这种方法不可取，应该想其它方法。

`<STDIN>`放在列表上下文时，它会一行一行读取，直到读取到文件结尾(EOF)。但是，如果是读取键盘输入，如何给出EOF？在Linux中，按下CTRL+D(Windows下是CTRL+Z)即可，默认它会发送EOF给操作系统，操作系统会通知perl到了文件结尾。

因为`<STDIN>`读取的每一行中默认就带有换行符，在列表上下文中，同样可以使用chomp()函数来去除每一行的换行符。
```
@lines=<STDIN>;
chomp(@lines);
```

或者采用更简洁的方式：
```
chomp(@lines=<STDIN>);
```