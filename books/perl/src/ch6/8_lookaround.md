## 环视锚定(断言)

环视(lookaround)是一种锚定匹配，即匹配的是位置，而不是字符。
- `(?=...)`：表示从左向右的顺序环视。如`(?=\d)`表示当前字符的右边是一个数字时就满足条件  
- `(?!...)`：表示顺序环视的否定。如`(?!\d)`表示当前字符的右边不是一个数字时就满足条件  
- `(?<=...)`：表示从右向左的逆序环视。如`(?<=\d)`表示当前字符的左边是一个数字时就满足条件  
- `(?<!)...`：表示逆序环视的取反。如`(?<!\d)`表示当前字符的左边不是一个数字时就满足条件  

一定要主要，环视锚定匹配的是位置，而不是字符，它不会消耗任何字符。

例如`"your name is longshuai MA"`和`"your name is longfei MA"`，使用`(?=longshuai)`将能锚定第一个句子中单词longshuai前面的位置(即空格和字母l中间的位置)，所以`(?=longshuai)long`才能long这几个字符。仅对于此处的两个句子，`long(?=shuai)`和`(?=longshuai)long`是等价的。

一般为了方便理解，在顺序环视的时候会将匹配内容放在锚定括号的左边(如`long(?=longshuai)`)，在逆序环视的时候会将匹配的内容放在锚定括号的右边(如`(?<=long)shuai`)。

例如：
```perl
my $str="abc123abcc12c34";

# 顺序环视
$str =~ /a.*c(?=\d)/;     # abc123abcc12c
say "$&";

# 顺序否定环视
$str =~ /a.*c(?!\d)/;     # abc123abc
say "$&";

# 逆序环视
$str =~ /a.*(?<=\d)c/;    # abc123abcc12c
say "$&";

# 逆序否定环视
$str =~ /a.*(?<!\d)c/;    # abc123abcc
say "$&";
```

逆序环视的表达式必须只能表示固定长度的字符串。例如`(?<=word)`或`(?<=word|word)`可以，但`(?<=word?)`不可以，因为`?`匹配0或1长度，长度不定，它无法对左边是word还是wordx做正确判断。
```perl
my $str="hello worlds Gaoxiaofang";
$str =~ /he.*(?<=worlds?) Gao/;         # 报错
$str =~ /he.*(?<=worlds|world) Gao/;    # 报错
```

在PCRE中，这种变长的逆序环视锚定可重写为`(?<=word|words)`，但Perl中不允许，因为Perl严格要求长度必须固定。

通过`\K`可以在一定程度上解决这样的问题：

```perl
my $str="hello worlds Gaoxiaofang";
$str =~ /he.*worlds?\K Gao/;
$str =~ /he.*(worlds|world)\K Gao/;
```

