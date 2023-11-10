## 正则引用： qr构建正则对象

可以在正则模式中使用变量替换，所以可以将正则中的一部分表达式先保存在变量中，然后插入到正则表达式中。例如：
```perl
my $str="hello worlds malong";
my $pattern="w.*d";
$str =~ /$pattern/;
say "$&";
```

但这样很容易出现问题：保存正则表达式的变量中存放特殊字符时要进行转义处理。例如，当使用`m//`的方式做匹配分隔符时，不能在变量中保存`/`，它需要先转义。

Perl提供了`qr/pattern/`的功能，它把pattern部分构建成一个正则表达式对象(也称为正则引用)，这使得构建复杂的正则表达式变得非常方便：  
- 在正则表达式中可以直接引用这个对象  
- 可以将这个对象保存到变量中，通过引用变量的方式来引用这个已保存好的正则对象  
- 将正则对象变量插入到其它正则模式中来构建更复杂的正则表达式  

其中：  
- `qr//`的斜线可以替换为其它符号，例如`qr() qr{} qr<> qr[] qr%% qr## qr!! qr$$ qr"" qr''`等    
- 使用单引号时比较特殊(即`qr''`)，它会使用单引号的语义去解析pattern部分。例如变量`$var`无法解析为变量的值，但这样可以使得正则表达式的元字符仍然起作用，例如`$`仍然表示行尾  

```perl
my $str="hello worlds malong";

# 直接作为正则表达式
$str =~ qr/w.*d/;
say "$&";

# 保存为变量，再作为正则表达式
my $pattern=qr/w.*d/;
$str =~ $pattern;    # (1)
$str =~ /$pattern/;  # (2)
say "$&";

# 保存为变量，作为正则表达式的一部分
$pattern=qr/w.*d/;
$str =~ /hel.* $pattern/;
say "$&";
```

还允许为这个正则对象设置修饰符，比如忽略大小写的匹配修饰符为i，这样在真正匹配的时候，就只有这一部分正则对象会忽略大小写，其余部分仍然区分大小写。
```perl
my $str="HELLO wORLDs malong";

my $pattern=qr/w.*d/i;      # 忽略大小写

$str =~ /HEL.* $pattern/;   # 匹配成功，$pattern部分忽略大小写
$str =~ /hel.* $pattern/;   # 匹配失败
$str =~ /hel.* $pattern/i;  # 匹配成功，所有都忽略大小写
```

### qr如何构建正则对象

可以输出查看qr构建的正则引用，从而了解qr构建的正则对象是怎样的结构：
```perl
my $patt1=qr/w.*d/;
say "$patt1";

my $patt2=qr/w.*d/i;    # 加上修饰符i
say "$patt2";

my $patt3=qr/w.*d/img;  # 加上修饰符img
say "$patt3";
```
输出结果：
```
(?^:w.*d)
(?^i:w.*d)
(?^mi:w.*d)
```
qr的作用实际上就是在给定的正则pattern基础上加上`(?^:)`并带上一些修饰符，得到的结果总是`(?^FLAGS:pattern)`。

但是上面patt3的修饰符g不见了。要知道其原因，需要了解`(?^:)`的作用：非捕获分组，并重置修饰符。重置为哪些修饰符？对于`(?^FLAGS:)`来说，只有这些修饰符`alupimsx`是可用的，即`(?^alupimsx:)`：  
- 如果给定的修饰符不在这些修饰符内，则不被识别，有时候会报错  
- 如果给定的修饰符属于这几个修饰符，那么没有给定的修饰符部分将采用默认值(不同版本可能默认是否开启的值不同)  

所以上面的g修饰符被丢弃了，甚至在进一步操作这个正则引用时，会报错。

由于qr构建正则表达式对象的时候给pattern部分加上了`(?^:)`，使得它们插入到其它正则中的时候，能保证这一段正则子表达式是独立的，不受全局修饰符影响。

```perl
my $patt1=qr/w.*d/im;
my $patt2=qr/hel.*d $patt1/i;
say "$patt2";     # 输出：(?^i:hel.*d (?^mi:w.*d))
```

### 正则对象作为标量的用法

`qr//`创建的正则对象引用是一个标量，可以使用标量的地方，就可以使用正则对象。例如，放进hash结构、放进数组结构、作为参数传递给函数、作为函数返回值被返回，等等。

例如，放进数组中形成一个正则表达式列表，然后给定一个待匹配目标，依次用列表中的这些模式去匹配。
```perl
use v5.10.1;
my @patterns = (
  qr/(?:Willie )?Gilligan/,
  qr/Mary Ann/,
  qr/Ginger/,
  qr/(?:The )?Professor/,
  qr/Skipper/,
  qr/Mrs?. Howell/,
);

my $name = 'Ginger';
foreach my $pattern ( @patterns ) {
  if( $name =~ /$pattern/ ) {
    say "Match!";
    print "$pattern";
    last;
  }
}
```

还可以将这些正则对象放进hash中，为每个pattern都使用key来标识，例如：
```perl
use v5.10.1;
my %patterns = (
  Gilligan => qr/(?:Willie )?Gilligan/,
  'Mary Ann' => qr/Mary Ann/,
  Ginger => qr/Ginger/,
  Professor => qr/(?:The )?Professor/,
  Skipper => qr/Skipper/,
  'A Howell' => qr/Mrs?. Howell/,
);
my $name = 'Ginger';
my( $match ) = grep { $name =~ $patterns{$_} } keys %patterns;
say "Matched $match" if $match;
```

### 构建复杂的正则表达式

有了qr，就可以将正则表达式细化成简单的子表达式，然后将它们组合起来形成复杂的正则表达式。例如：
```perl
my $howells = qr/Thurston|Mrs/;
my $tagalongs = qr/Ginger|Mary Ann/;
my $passengers = qr/$howells|$tagalongs/;
my $crew = qr/Gilligan|Skipper/;
my $everyone = qr/$crew|$passengers/;
```

就像RFC 1738中对URL各个部分的解剖，如果转换成Perl正则，大概是这样的：
```perl
# 可复用的基本符号类
my $alpha = qr/[a-z]/;
my $digit = qr/\d/;
my $alphadigit = qr/(?i:$alpha|$digit)/;
my $safe = qr/[\$_.+-]/;
my $extra = qr/[!*'\(\),]/;
my $national = qr/[{}|\\^~\[\]`]/;
my $reserved = qr|[;/?:@&=]|;
my $hex = qr/(?i:$digit|[A-F])/;
my $escape = qr/%$hex$hex/;
my $unreserved = qr/$alpha|$digit|$safe|$extra/;
my $uchar = qr/$unreserved|$escape/;
my $xchar = qr/$unreserved|$reserved|$escape/;
my $ucharplus = qr/(?:$uchar|[;?&=])*/;
my $digits = qr/(?:$digit){1,}/;

# 可复用的URL组成元素
my $hsegment = $ucharplus;
my $hpath = qr|$hsegment(?:/$hsegment)*|;
my $search = $ucharplus;
my $scheme = qr|(?i:https?://)|;
my $port = qr/$digits/;
my $password = $ucharplus;
my $user = $ucharplus;

my $toplevel = qr/$alpha|$alpha(?:$alphadigit|-)*$alphadigit/;
my $domainlabel = qr/$alphadigit|$alphadigit(?:$alphadigit|-)*$alphadigit/x;
my $hostname = qr/(?:$domainlabel\.)*$toplevel/;
my $hostnumber = qr/$digits\.$digits\.$digits\.$digits/;
my $host = qr/$hostname|$hostnumber/;
my $hostport = qr/$host(?::$port)?/;
my $login = qr/(?:$user(?::$password)\@)?/;

my $urlpath = qr/(?:(?:$xchar)*)/;
```

然后就可以用上面看上去无比复杂的正则表达式去匹配一个路径是否是合格的http url：
```perl
use v5.10.1;
my $httpurl = qr|$scheme$hostport(?:/$hpath(?:\?$search)?)?|;
say "yes" if /$httpurl/;
```

### 正则表达式模块

虽然qr为构建复杂正则表达式提供了比较友好的方式，但很多正则表达式本身就是复杂的。实际上，很多常用且复杂的正则表达式已经被别人造好了轮子，我们可以直接拿来用。例如，`Regexp::Common`模块，提供了很多种已经构建好的正则表达式。

首先安装这个模块：
```bash
sudo cpan -i Regexp::Common
```

以下是CPAN上提供的`Regexp::Common`已造好的轮子，可参考：<https://metacpan.org/release/Regexp-Common>。
```
Regexp::Common - Provide commonly requested regular expressions
Regexp::Common::CC - provide patterns for credit card numbers.
Regexp::Common::SEN - provide regexes for Social-Economical Numbers.
Regexp::Common::URI - provide patterns for URIs.
......
Regexp::Common::comment - provide regexes for comments.
Regexp::Common::delimited - provides a regex for delimited strings
Regexp::Common::lingua - provide regexes for language related stuff.
Regexp::Common::list - provide regexes for lists
Regexp::Common::net - provide regexes for IPv4, IPv6, and MAC addresses.
Regexp::Common::number - provide regexes for numbers
Regexp::Common::profanity - provide regexes for profanity
Regexp::Common::whitespace - provides a regex for leading or trailing whitescape
Regexp::Common::zip - provide regexes for postal codes.
```
这些正则表达式是通过hash进行嵌套的，最外层hash的名称为`%RE`。

以模块`Regexp::Common::URI::http`为例，它提供的是HTTP URI的正则表达式，它嵌套了两层，第一层的key为URI，这个key对应的值是第二层hash，第二层hash的key为HTTP，于是可以通过`$RE{URI}{HTTP}`的方式获取这个正则。

例如，匹配一个http url是否合理：
```perl
use Regexp::Common qw(URI);
say "yes" if /$RE{URI}{HTTP}/;
```

再例如，从`Regexp::Common::net`中可以获取匹配IPV4的正则表达式：
```perl
use Regexp::Common qw(net);
my $ipv4=$RE{net}{IPv4};
say $ipv4;
```
以下是结果(为美化排版，下面将一行结果分成了多行显示)：
```
(?:
(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})
[.]
(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})
[.]
(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})
[.]
(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})
)
```

需要注意的是，在真正匹配的时候，应该将获取到的正则引用配合锚定一起使用，否则对318.99.183.11进行匹配的时候也会返回true，因为18.99.183.11是符合匹配结果的。所以，对前后都加上锚定，例如：
```perl
$ipv4 =~ /^$RE{net}{IPv4}$/;
```

可以将上面的ipv4正则改造一下(去掉非捕获分组的功能)，让它适用于shell工具中普遍支持的扩展正则(为了美化排版，将一行结果分成多行显示)：
```
(25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})
(\.
(25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})
){3}
```

默认情况下，`Regexp::Common`的各个模块没有开启捕获功能。如果要使用`$1 $N`这种变量，需要使用`{-keep}`选项，至于每个分组捕获的是什么内容，需要参考帮助文档的说明。

例如：
```perl
use Regexp::Common qw(number);
say $1 if /$RE{num}{int}{ -base => 16 }{-keep}/;
```

