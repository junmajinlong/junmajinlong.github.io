## 字符串字面量

Perl中的字符串和Shell类似，可以使用双引号或单引号包围。它们的区别是：

- 双引号：双引号包围的是字符串字面量，但允许在双引号内使用变量内插(Interpolate)、表达式内插、反斜线转义、反斜线字符序列  
- 单引号：单引号包围的字符串字面量不允许使用变量内插、表达式内插、反斜线序列，也禁止几乎所有的反斜线转义，只允许对单引号自身和反斜线自身进行反斜线转义  

例如，下面是一些字符串字面量：

```perl
use v5.12;
use warnings;

my $name = "junma";
my $age = 23;

say "hello";   # hello，普通字符串
say 'hello';   # hello，普通字符串

# 双引号内可以使用反斜线转义，可以使用单引号
# 单引号内只能转义单引号和反斜线，可以使用双引号
say "hello\"world, hello'world";  # hello"world, hello'world
say 'hello\'world, hello"world';  # hello'world, hello"world
say "hello\\world";  # hello\world
say 'hello\\world';  # hello\world

# 双引号内可以变量内插，单引号内不能变量内插
say "hello $name";    # hello junma
say 'hello $name';    # hello $name

# 双引号内可以表达式内插，单引号内不能表达式内插
say "age: @{[$age+2]}";  # age: 25
say 'age: @{[$age+2]}';  # age: @{[$age+2]}

# 双引号内转义变量内插、转义表达式内插
say "hello \$name";      # hello $name
say "age: \@{[$age+2]}"; # age: @{[23+2]}

# 双引号内可以使用反斜线字符序列，单引号内不允许使用
say "hello\tworld";  # hello_TAB_world
say 'hello\tworld';  # hello\tworld
```

变量内插不仅可以内插标量标量，还可以内插数组变量，但不能内插hash变量。

```perl
my @arr = (11, 22, 33);
say "@arr";     # 输出：11 22 33
my %person = (name=>"junma", age=>23);
say "%person";  # 输出：%person，hash变量不内插
```

### 反斜线字符序列

Perl除了支持换行符`\n`、制表符`\t`等ASCII的特殊字符外，还支持几个具有特殊意义的反斜线序列。

```
\u 修改下一个字符为大写
\l 修改下一个字符小写 
\U 修改后面所有字符大写 
\L 修改后面所有字符小写 
\Q 使后面的所有字符都成为字面符号
\E 结束\U \L或\Q的效果
```

例如：

```perl
print "\uabc"; # 输出Abc
print "\Uabc"; # 输出ABC
print "ab\Ucxyz";   # 输出abCXYZ
print "ab\Ucx\Eyz"; # 输出abCXyz
```

这些反斜线序列有时候非常实用，特别是在使用函数修改数据不方便时。例如，在正则的s替换语法`s///`中，replacement部分可以使用这些反斜线字符序列。

```perl
# 将匹配的字符换成大写
my $name = "junmajinlong";
$name =~ s/([j-n])/\U\1\E/g;
say $name;    # 输出：JuNMaJiNLoNg
```

### 使用q()和qq()替代单双引号

如果一个字符串比较复杂，使用单双引号包围字符串可能会非常麻烦。

Perl中可以使用`q`实现单引号相同的引用功能，使用`qq`实现双引号相同的引用功能。

例如：

```perl
q(abc)            # 等价于'abc'
qq(abc)           # 等价于"abc"
qq(hello"world)   # 等价于"hello\"world"
```

注意上面`q()`和`qq()`的括号，它们是引用的起始符和终止符，它们可以被替换为其他成对的符号：要么是前后相同的单个标点字符，要么是对称的括号(大括号、小括号、尖括号、中括号都可以)。

```perl
say q!abc!;
say q<abc>;
say qq{abc};
```

甚至，还可以使用数值、字母作为起始符和终止符，但要求将起始符和q或qq使用空白分隔开。

```perl
say qq 1def1;   # 等价于qq(def)
say qq adefa;   # 等价于qq(def)
```

注意，如果字符串中出现了起始符或终止符，则需要对其反斜线转义。

```perl
say qq{abc\}def};  # 转义终止符
say qq{abc\{def};  # 转义起始符
say qq ad\aefa;    # 转义起始符a，输出daef
```

但是，在起始符和终止符之间，可以嵌套成对的起始符号和终止符号。

```perl
say qq{ab{cd}e};   # ab{cd}e
```

### bareword

虽然觉得很诡异，但不用任何引号包围的字符也被Perl当作字符串，这种字符串称为Bareword(裸字符串)。如果开启了warnings功能，使用Bareword时，perl会警告。

```perl
$s = abc;   ## bareword字符串abc
say $s;   # abc
```

注意，不要直接在say/print/printf的第一个参数处使用bareword，因为第一个bareword会被它们当作输出文件句柄。

```perl
# 将bareword字符串hello写入标准输出
# 这里的STDOUT也是bareword，但会被say解析为输出文件句柄
say STDOUT hello;
```

不建议使用bareword。

### v-str

Perl还支持一种称为v字符串的字面量，v字符串多用来表示版本号。例如`use v5.12`。

v字符串有时候也能用在其他方面，例如保存点分十进制的IP地址，这里不深究v字符串，除了用在版本号上，多数时候也用不到它。

### here doc

所谓heredoc，即表示此处内容是文档，即将文档内容当作字符串来处理。

既然是文档，就需要有文档起始符和文档结束符，分别标识文档从哪里起始，到哪里结束。一般来说，所有支持heredoc的语言，文档起始符和文档结束符必须相同(一般使用EOF或eof作为起始符和结束符)，且结束符必须单独占行且顶格书写。

Perl中支持的heredoc格式如下，以print为例： 

```perl
print <<EOF;
  line1
  line2
  line3
EOF

# 输出结果：
#  line1
#  line2
#  line3
```

这里以EOF作为文档起始符和结束符，起始符EOF后面加上分号结尾表示print语句结束。结束符EOF单独占用一行，且顶格书写。起始符和结束符中间是怎样的数据，输出时就是怎样的数据。

只要需要字符串的地方，都可以使用heredoc来表示这部分字符串。例如，将heredoc字符串赋值给变量、作为函数字符串实参，等等。

```perl
# heredoc作为字符串赋值给变量
$msg = <<EOF;
  HELLO
  WORLD
EOF
print $msg;
```

还**可以为heredoc的起始符加上单引号、双引号以及反引号**。它们的效果和普通的单、双、反引号的效果一样：   
- 单引号内只允许`\\ \'`这两种转义  
- 双引号内允许变量内插、表达式内插、反斜线转义、反斜线字符序列  
- 加反引号`` ` ` ``，表示将字符串放进Shell环境执行(和Shell的命令替换效果差不多)  
- 不加引号等价于加双引号，加反斜线前缀`\EOF`等价于加单引号

加单双引号： 

```perl
$name="malongshuai";
print <<'EOF';
  haha
  \$name # 反斜线转义功能失效
  
  $name # 变量无法替换
EOF

print <<"EOF";
  haha
  \$name # 反斜线成功转义
  
  $name # 变量成功替换
EOF
```

加反引号： 

```perl
print <<`EOF`;
  date +"%F %T"
EOF
```

另外，从Perl v5.26开始，**可以在heredoc的起始符EOF或被引用的EOF前加上波浪号**：

- `<<~EOF   <<~"EOF"`  
- `<<~'EOF' <<~\EOF` 
- ``<<~`EOF` ``  

加上前缀波浪号，使得heredoc允许终止符被缩进，且会将起始符和终止符之间的heredoc内容的空白前缀进行修剪，使之与终止符的缩进进行对齐。看示例理解。

```perl
print <<~EOF;
  line1
    line2
  line3
  EOF
  
print <<~\eof;
    LINE1
  LINE2
    LINE3
  eof
```

输出：

```
line1
  line2
line3
  LINE1
LINE2  
  LINE3
```

使用波浪号前缀时，要求heredoc的内容必须不能出现在终止符之前，否则报错。例如下面代码会报错。

```perl
print <<~\eof;
    LINE1
LINE2
    LINE3
  eof
```

**可以同时在一个语句中使用多个heredoc**，它们互不影响。例如：

```perl
print <<~EOF, <<~\eof;
  line1
    line2
  line3
  EOF
    LINE1
      LINE2
    LINE3
    eof
```

输出：

```
line1  
  line2
line3  
LINE1  
  LINE2
LINE3 
```

由于heredoc的内容带有尾部换行符，如果想去掉这个尾部换行符，可：

```perl
chomp(my $str = <<STR);
line1
line2
STR

print $str;
```

### 字符串串联和重复

Perl使用点`.`来串联字符串。

例如：

```perl
my $name = "junmajinlong";
say "www.".$name.".com";  # 输出：www.junmajinlong.com
```

Perl使用`x`(字母xyz的x)来重复字符串指定次数：

```perl
say '-' x 5;   # 重复5次，输出：-----
```

需注意，如果`x`重复次数是小数，则截断为整数，如果x是0，则清空字符串。

```perl
say '-' x 5.6;  # 输出-----
say '-' x 0;    # 输出空字符串
```

### 数值和字符串的类型自动转换

在前一面介绍数值字面量的小节中曾提到过，当使用运算符`+ - * / % -(负号) ++ -- abs`以及`< <= > >= == !=`时，都会将操作数强制转换为数值。

对于字符串来说，当使用`.`串联字符串或使用`x`来重复字符串时，会将操作对象强制转换为字符串。

```perl
# 字符串转换为数值
"0333" + 22  # 返回355
"12abc" * 3  # 36
"abc12" * 4  # 0
" 12abc" * 3 # 36

# 数值转换为字符串
"033".22    # 返回03322
033.22      # 返回2722，033表示8进制，转换为十进制为27(3*8+3)
```

注意，当字符串操作符和数值操作符混用时，注意它们的优先级。如果不确定优先级，使用括号强制改变优先级：

```perl
"abc".5*3 # 返回abc15，乘法先运算
"abc".5 + 3 # 返回3，"."先运算
"abc".(5+3) # 返回abc8
```