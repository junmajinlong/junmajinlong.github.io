## 全局匹配

Perl在默认情况下，匹配一次成功后就会立即退出匹配。例如，对于下面的正则匹配来说：

```perl
"abcabc" =~ /ab/
```

匹配到第一个ab后就结束正则匹配。

如果想要继续匹配第二个ab，就需要进行全局匹配。使用`g`修饰符可以让正则进入全局匹配工作模式：匹配完第一个ab，继续向后匹配，直到匹配完成功或匹配结束。

当使用了全局修饰符g时，在标量上下文中，正则匹配返回表示匹配成功与否的值，在列表上下文中，如果匹配成功，则返回全局匹配成功的内容列表或小括号分组捕获的内容。

因此，想要查看全局匹配的匹配结果，将其放在列表上下文即可：

```perl
my @arr = "abcabc" =~ /ab/g;  # 匹配返回qw(ab ab)
say "@arr";  # ab ab
```

或者，使用循环匹配的方式来验证全局匹配中的每次匹配结果：

```perl
my $name="aAbBcCaBc";
while($name =~ m/ab/gi){
  say "pre match : $`";
  say "match     : $&";
  say "post match: $'";
}
```
执行它，将输出如下内容：
```
pre match : a
match     : Ab
post match: BcCaBc
pre match : aAbBcC
match     : aB
post match: c
```

在开启了g全局匹配后，perl会在每次匹配成功后记下匹配的偏移位置，以便下次匹配时可以从该位移处继续向后匹配。每次匹配成功后的位移值，都可以通过pos()函数获取。偏移值从0开始算，0位移代表的是第一个字符左边的位置。如果本次匹配导致位移指针重置，pos将返回undef。
```perl
my $name="123ab456";
$name =~ m/\d\d/g;     # 第一次匹配，匹配成功后记下位移
say "matched: $&, pos: ",pos $name;
$name =~ m/\d\d/g;     # 第二次匹配，匹配成功后记下位移
say "matched: $&, pos: ",pos $name;
```
执行它，将输出如下内容：
```
matched: 12, pos: 2
matched: 45, pos: 7
```
匹配失败的时候，正则匹配操作会返回假，所以可以作为if或while等的条件语句来终止全局匹配。例如：
```perl
my $name="123ab456";
while($name =~ m/\d\d/g){
  say "matched: $&, pos: ",pos $name;
}
```

### c修饰符

默认全局匹配情况下，当本次匹配失败，便宜指针将重置到起始位置0处，也就是说，下次匹配将从头开始匹配。例如：

```perl
my $txt="1234a56";
$txt =~ /\d\d/g;      # 匹配成功：12，位移向后移两位
say "matched $&: ",pos $txt;
$txt =~ /\d\d/g;      # 匹配成功：34，位移向后移两位
say "matched $&: ",pos $txt;
$txt =~ /\d\d\d/g;    # 匹配失败，位移指针回到0处，pos()返回undef
say "matched $&: ",pos $txt;
$txt =~ /\d/g;        # 匹配成功：1，位移向后移1位
say "matched $&: ",pos $txt;
```

执行上述程序，将输出：
```
matched 12: 2
matched 34: 4
matched 34:   #<-- warning: use undef value
matched 1: 1
```

如果`g`修饰符下同时使用`c`修饰符，也就是`gc`，它表示全局匹配失败的时候不重置位移指针。也就是说，本次匹配失败后，位移指针会卡在原处不动，下次匹配将从这个位置处开始匹配。
```perl
my $txt="1234a56";
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\d\d\d/gc;   # 匹配失败，$&和pos()保留上一次匹配成功的内容
say "matched $&: ",pos $txt;
$txt =~ /\d/g;        # 匹配成功：5
say "matched $&: ",pos $txt;
$txt =~ /\d/g;        # 匹配成功：6
say "matched $&: ",pos $txt;
$txt =~ /\d/gc;        # 匹配失败
say "matched $&: ",pos $txt;
```

执行上述程序，将输出：
```
matched 12: 2
matched 34: 4
matched 34: 4
matched 5: 6
matched 6: 7
matched 6: 7
```

### \G反斜线序列

如果上面第三个匹配语句不是`/\d\d\d/gc`，而是`/\d/g`，它匹配字母a的时候也失败，但是最终它会匹配成功。因为，它会继续先后匹配，直到匹配成功或匹配结束。

```perl
my $txt="1234ab56";
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\d/g;   # 字母a匹配失败，后移一位，字母b匹配失败，后移一位，数值5匹配成功
say "matched $&: ",pos $txt;
$txt =~ /\d/g;   # 数值6匹配成功
say "matched $&: ",pos $txt;
```

执行上述程序，将输出：
```
matched 12: 2
matched 34: 4
matched 5: 7
matched 6: 8
```

实际上，全局匹配时，默认情况下，如果当前字符匹配失败，将会后移继续去匹配，直到匹配成功或匹配结束。

可以指定`\G`，使得本次匹配强制从位移处进行匹配，不允许跳过任何匹配失败的字符。

- 如果本次`\G`全局匹配成功，位移指针后移到匹配终止的位置  
- 如果本次`\G`全局匹配失败，且没有加上c修饰符，那么位移指针将重置  
- 如果本次`\G`全局匹配失败，且加上了c修饰符，那么位移指针将卡着不动  

例如：
```perl
my $txt="1234ab56";
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\G\d/g;   # 强制从位移4开始匹配，无法匹配字母a，但又不允许跳过
                   # 所以本次\G全局匹配失败，由于没有修饰符c，指针重置
say "matched $&: ",pos $txt;
$txt =~ /\G\d/g;   # 指针回到0，强制从0处开始匹配，数值1能匹配成功
say "matched $&: ",pos $txt;
```
以下是输出内容：
```
matched 12: 2
matched 34: 4
matched 34:    #<-- warning: use undef value
matched 1: 1
```

如果将上面第三个匹配语句加上修饰符c，甚至后面的语句也都加上`\G`和c修饰符，那么位移指针将卡在那个位置：
```perl
my $txt="1234ab56";
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\d\d/g;
say "matched $&: ",pos $txt;
$txt =~ /\G\d/gc;        # 匹配失败，指针卡在原地
say "matched $&: ",pos $txt;
$txt =~ /\G\d/gc;        # 匹配失败，指针继续卡在原地
say "matched $&: ",pos $txt;
$txt =~ /\Gab/gc;        # 匹配成功，指针后移两位
say "matched $&: ",pos $txt;
```

以下是输出结果：
```
matched 12: 2
matched 34: 4
matched 34: 4
matched 34: 4
matched ab: 6
```

一般来说，全局匹配都会用循环去多次迭代，和上面一次一次列出匹配表达式不一样。所以，下面使用while循环的例子来对`\G`和c修饰符稍作解释，其实理解了上面的内容，在循环中使用`\G`和c修饰符也一样很容易理解。
```perl
my $txt="1234ab56";
while($txt =~ m/\G\d\d/gc){
  say "matched: $&, ",pos $txt;
}
```
执行结果：
```
matched: 12, 2
matched: 34, 4
```
当第三轮循环匹配到a字母的时候，由于使用了`\G`，导致匹配失败，结束循环。

上面使用c与否是无关紧要的，但如果这个while循环的后面后还有对`$txt`的匹配，那么使用c修饰符与否就有关系了。例如下面两段程序，返回结果不一样：
```perl
## 第一段匹配
## ----------------
my $txt1="1234ab56";
# 使用c修饰符，第二轮匹配完成后位移指针卡住，pos=4
while($txt1 =~ m/\G\d\d/gc){
  say "matched: $&, ",pos $txt1;
}
# 从卡住的地方继续匹配，匹配失败，继续卡住，pos=4
$txt1 =~ m/\d\d/gc;
say "matched: $&, ",pos $txt1;

## 第二段匹配
## ----------------
# 不使用c修饰符，第二轮匹配完成后位移指针重置，pos=0
my $txt2="1234ab56";
while($txt2 =~ m/\G\d\d/g){   # 不使用c修饰符
  say "matched: $&, ",pos $txt2;
}
# 从头开匹配配，匹配成功，pos=2
$txt2 =~ m/\G\d\d/gc;
say "matched: $&, ",pos $txt2;
```