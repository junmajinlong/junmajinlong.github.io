---
title: Ruby正则表达式
p: ruby/ruby_regexp.md
date: 2020-05-14 12:37:22
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------


# Ruby正则表达式

Ruby的正则表达式采用Onigmo正则引擎，官方手册参考：<https://github.com/k-takata/Onigmo/blob/master/doc/RE>。

Ruby有一个名为Regexp的类，它是正则表达式类。

## 构建正则表达式对象

正则表达式对象的创建方式可以通过字面量方式创建，也可以通过Regexp类提供的方法new或compile来创建(它们别名关系)。

有两种字面量创建正则对象的方式：
```ruby
/hello/.class     #=> Regexp
%r(hello).class   #=> Regexp

/hello/im         #=> /hello/mi
%r(a.*b)i         #=> /a.*b/i
```

![](/img/ruby/1589465070256.png)

所以，如果想对某已存在的正则对象添加修饰符，可以先通过`source`方法将其转换成原始字符串模式，再通过`Regexp.new`来指定新的修饰符并编译成新的正则对象。

例如：
```ruby
Regexp.new("abc", Regexp::IGNORECASE)      #=> /abc/i
Regexp.new("abc", Regexp::MULTILINE)       #=> /abc/m
Regexp.new("abc", Regexp::EXTENDED)        #=> /abc/x
Regexp.new("abc", Regexp::IGNORECASE | Regexp::MULTILINE) #=> /abc/mi

reg = /abc/i  #=> /abc/i
reg.source    #=> "abc"
Regexp.new(reg.source, Regexp::MULTILINE)
#=> /abc/m
```

## 使用Ruby正则去匹配

Ruby正则是一个对象，它定义了几个方法可以用来匹配字符串和Symbol：`=~、match、~ ===`。但字符串和符号也能使用前两个方法，所以有以下几种匹配方式：
```
reg_obj =~ Str/Sym       # (1)
Str/Sym =~ reg_obj       # (2)
reg_obj.match(Str/Sym)   # (3)
Str.match(reg_obj)       # (4)
Sym.match(reg_obj)       # (5)
Str[/regexp/]            # (6)

~ reg_obj                # (7)，对$_做匹配，等价于(8)，还有更简写法，见下文
reg_obj =~ $_            # (8)

reg_obj === Str/Sym      # (9)
```

`=~`和`match`方法都是Regexp的方法，同样在String中进行了重写，也适用于Symbol对象。所以正则表达式和字符串的左右位置可以任意。

```ruby
str = "hello world"
reg = /hello/

# 等价
str =~ reg
reg =~ str

# 等价
str.match(reg)
reg.match(str)

# 匹配Symbol
:hello =~ /hello/  #=> 0
/hello/ =~ :hello  #=> 0
```

> 但使用字符串的match方法时，有一种情况必须避免：`str1.match(str2)`，这种情况下会将str2转换为正则表达式对象，然后再做匹配，这是一种低效率行为，因为它不会缓存正则表达式的编译结果，使得在循环每次都会重新编译，导致效率极大降低。据个人测试，至少比普通的正则匹配方式慢2倍（某次测试的结果分别是0.8秒和2.3秒）。

但是`reg =~ str`和`str =~ reg`是不等价的，当正则中使用了命名捕获，只有前者才会设置分组变量。再但是，`reg.match(str)`和`str.match(reg)`是等价的，因为两个match方法都返回MatchData对象，分组捕获的信息已经放进了该对象中。

例如：
```ruby
# 不等价
"hello world" =~ /(?<h>hello) (?<w>world)/
puts h,w   # NameError: undefined variable

/(?<h>hello) (?<w>world)/ =~ "hello world"
puts h,w   #=> "hello" "world"

# 等价
md1 = "hello world".match(/(?<h>hello) (?<w>world)/)
md2 = /(?<h>hello) (?<w>world)/.match("hello world")
md1["h"]    #=> hello
md2["h"]    #=> hello
```

而`~ reg_obj`的`~`方法是专门用于匹配`$_`这个特殊变量的，它等价于`$_ =~ reg_obj`或`reg_obj =~ $_`，这使得写ruby一行式命令非常方便。不仅如此，当正则表达式隐含地处于匹配上下文中时，还能够省略`~`。例如：

```bash
>> $_="hello world"
>> /(w.*d)/      # 定义并返回正则，然后丢弃返回值
>> ~/(w.*d)/    # 匹配$_
>> puts $1 if ~/(w.*d)/  # 匹配$_，输出world
>> puts $1 if /(w.*d)/   # 匹配$_，输出world

# 下面三个ruby命令都输出world
ruby -e '$_="hello world";~/(w.*d)/;puts $1'
ruby -e '$_="hello world";puts $1 if ~/(w.*d)/'
ruby -e '$_="hello world";puts $1 if /(w.*d)/'

# 但是下面的正则未处在匹配上下文，而是处于正则定义上下文
# 因此没有匹配动作，而是定义正则
ruby -e '$_="hello world";/(w.*d)/;puts $1' # 定义并返回正则然后丢弃
ruby -e '$_="hello world";puts /(w.*d)/'  # 定义并返回正则(?-mix:(w.*d))，然后输出
```

`===`则是Regexp实现的智能匹配符号，主要用于case语句和grep筛选中：

```ruby
str = "hello abc"
case str
when /hello/; puts "hello"
when /world/; puts "world"
end

%w(abc ABC def DEF).grep(/^[A-Z]+/)
#=> ["ABC", "DEF"]
```

match方法还可以指定第二个参数表示从哪个字符开始匹配，不仅如此，还能在匹配成功时使用语句块，它会向语句块中传递match所返回的MatchData对象，这个对象包含了匹配成功的状态信息，后文会详细介绍，现在只需要知道可以通过该对象来获取获取正则匹配到的每一个分组内容即可。
```
match(str,pos=0) → matchdata or nil
match(str,pos=0) {|matchdata| block}
```

例如：
```ruby
>> /(.)(.)(.)/.match("abc")
#=> #<MatchData "abc" 1:"a" 2:"b" 3:"c">

>> /(.)(.)/.match("abc", 1)
#=> #<MatchData "bc" 1:"b" 2:"c">

"hello world".match(/(\w+) (\w+)/) do |m|
  puts m[0]
  puts m[1]
  puts m[2]
end
=begin
hello world
hello
world
=end
```

Ruby的正则匹配操作在匹配成功时，返回的并不是所匹配到的内容，对于`=~`匹配方式而言，返回的是匹配到字符的索引位置，对于`match`方法而言，返回的是MatchData对象。但它们在匹配失败时都返回nil。

例如：
```ruby
str = "hello world"
str =~ /lo/      #=> 3
str =~ /world/   #=> 6
str =~ /HELLO/   #=> nil
```

如果想要取得所匹配到的内容，可以使用`$&`变量或去处理返回的MatchData对象，它封装了匹配成功后的状态，详细说明见后文。

如果想要取得所匹配到的内容，可以使用`$&`变量或去处理返回的MatchData对象，它封装了匹配成功后的状态，详细说明见后文。

或者更简单的，直接使用`Str[/regex/]`匹配方式：
```ruby
"hello world"[/w.*d/]
#=> "world"
```

## Ruby正则的修饰符

Ruby的正则支持四种常见的修饰符，这四种修饰符可以使用单个字符表示，可以使用数字代号表示，也可以使用Regexp的常量表示。但是它们用的场景不一样。

还支持其它一些修饰符，主要和编码有关，所以不做讨论。

```
/pat/i - (数值1，Regexp::IGNORECASE)忽略大小写
/pat/m - (数值4，Regexp::MULTILINE)元字符`.`可以匹配换行符
/pat/x - (数值2，Regexp::EXTENDED)可在正则中使用空白和#号注释
/pat/o - (没有对应的常量和数值代号)只执行一次#{}内插操作

/pat/u - UTF-8
/pat/e - EUC-JP
/pat/s - Windows-31J
/pat/n - ASCII-8BIT
```

注意，ruby没有全局修饰符`g`，如果需要全局匹配，则使用字符串的scan方法：`str.scan(reg)`。

先介绍下修饰符的用法，再单独解释`x`和`o`修饰符。

### 修饰符的使用方法

第一种使用方式是放在字面量的后面，这种方式的修饰符是对整个正则表达式生效的。

例如：
```ruby
/pat/i
/pat/im
%r(pat)i
%r(pat)imo
```

第二种使用方式是部分生效，在正则表达式的内部通过`(?on-off)`来使的后续正则部分应用对应的修饰符。例如：
```ruby
/a(?i-mx)a.a/
```

`(?i-mx)`表示该括号后面的正则部分将开启不区分大小写的匹配，且关掉m和x的功能，即点不能匹配换行符。这表示只有第一个字母a的匹配是区分大小写的，而后面3个字符是不区分大小写的，而且`.`不能匹配换行符。

```ruby
/a(?i-mx)a.a/ =~ "aA a" #=> 0
/a(?i-mx)a.a/ =~ "aa A" #=> 0
/a(?i-mx)a.a/ =~ "aa\nA"  #=> nil
```

第三种方式是`(?on-off:pat)`，它类似于第二种，它表示这个括号内的修饰符只对该括号内的pat有效，对括号外的不生效。

以下是一些示例分析。

例如，对于待匹配字符串『Hello world gaoxiaofang』，使用以下几种模式去匹配的话：

```ruby
str = "Hello world gaoxiaofang"
str1 = "Hello\nworld gaoxiaofang"

# 表示匹配hello时，可忽略大小写，但匹配world时仍然区分大小写。所以匹配成功
/(?i:hello) world/ =~ str  #=> 0

# 表示可以跨行匹配"hello\nworld"，也可以匹配单行的"hello world"，
# 且hello部分忽略大小写。所以匹配成功
/(?im:hello.)world/ =~ str  #=> 0
/(?im:hello.)world/ =~ str1 #=> 0

# hello部分会不区分大小写，world区分大小写，world后面的也不区分大小写
/(?i:hello (?-i:world) gaoxiaoFANG)/ =~ str #=> 0

# 和前面的类似，但是将"FANG"放到了括号外，意味着这部分要区分大小写。所以匹配失败
/(?i:hello (?-i:world) gaoxiao)FANG/ =~ str #=> nil

#和前面的类似，但是在最外面带上了i修饰符，意味着FANG这部分也可以不区分大小写。所以匹配成功
/(?i:hello (?-i:world) gaoxiao)FANG/i =~ str #=> 0

# hello和world部分都不区分大小写，且world括号中的点可以匹配换行符
/(?i)hello (?m:World.)/ =~ str   #=> 0
```

### x修饰符

`x`修饰符是可以在编写正则表达式的字符串里出现空格和#号表示的注释。例如：

```ruby
str = "cat sheep tiger"
str =~ /(\w) *(\w) *(\w)/
str =~ /(\w)\s*   (\w)\s*   (\w)/x
str =~ /
        (\w)\s*      # 可以加上本行注释：匹配第一个单词
        (\w)\s*      # 可以加上本行注释：匹配第二个单词
        (\w)         # 可以加上本行注释：匹配第三个单词
        /x
```

由于`x`修饰符使得正则字符串中的空白符号和#符号代表特殊的意思，所以要想表示匹配空白，需要使用`\s`符号或`\`转义，要表示匹配#则要使用`\#`转义该符号。

### o修饰符

对于Ruby来说，`o`修饰符直接关系到多次匹配时的性能问题。

`o`修饰符表示只执行一次`#{}`内插操作，不理解`o`修饰符可能很难理解这句话的意思。

事实上，任何一个正则表达式在真正能用于匹配之前，都需要经过两个基本的过程：  
- 处理正则源字符串文本  
- 编译正则表达式字符串  

很多时候，正则的源字符串是静态不变的，这类源字符串编译后的结果会缓存下来。而如果正则源字符串中使用了内插表达式`#{}`，则这个表达式的源字符串是动态的，不是固定不变的，比如内插的变量可能会发生变化，这类动态源字符串的编译结果默认不会缓存。

例如，在循环中需要多次构建字符串相同的正则，由于它们的源字符串是静态不变的，为了避免多次构建编译，它会缓存上一次的源字符串的编译结果。

```ruby
# 一直在变化
/xxx/.object_id   #=> 70368620934660
/xxx/.object_id   #=> 70368621014860
/xxx/.object_id   #=> 70368621097520

# 循环内，不变
("a".."d").each {|x| puts /xxx/.object_id}
70368620838300
70368620838300
70368620838300
70368620838300

# 函数内，不变
def f()
  /abc/.__id__
end

puts f     #=> 70368338800900
puts f     #=> 70368338800900
puts f     #=> 70368338800900
```

从结果上看到多次循环时，正则对象并没有重复构建，而是使用了上一次已经构建好的结果。

而在循环外，则每一次都重新构建，这是因为Ruby采用的是词法作用域规则，文本的定义位置决定其所见范围，每定义一次就解释一次。所以循环外多次定义会多次解释生成多个对象，而定义在循环内，或定义在方法内等一次定义多次执行的位置，不需要多次解释。

但是，如果循环中的正则使用了表达式内插，由于是动态的，所以不会缓存它的编译结果，每次循环中的正则对象都将重新编译：
```ruby
var="xxx"

("a".."d").each {|x| puts /#{var}/.object_id}
70368621566940
70368621566740
70368621566540
70368621566340
```

而如果使用`o`修饰符，这表示只执行一次内插操作，也就是认为正则源字符串是静态不变的，是可以缓存的，这意味着强制缓存编译结果。
```ruby
var="xxx"

("a".."d").each {|x| puts /#{var}/o.object_id }
70368621742640
70368621742640
70368621742640
70368621742640
```

前面说正则表达式的构建包含两个过程：处理正则源字符串文本和编译。其实`o`修饰符产生效果的位置在第一步：处理正则源字符串文本，它在这一步将其处理成静态的源字符串文本，使其可以被缓存下来。所以，使用了`o`和不使用`o`的正则匹配过程性能差距非常大，因为正则的编译过程相比缓存的查找过程是慢很多的。

上面的例子虽然适合理解`o`修饰符，但是毕竟没什么实际用处。但下面的例子是非常常见的：循环匹配每一行数据
```ruby
reg="root"

File.new("/etc/passwd").each do |line|
  puts line if /#{reg}/ =~ line
end
#=> root:x:0:0:root:/root:/bin/bash
```

上面每次循环都在构建新的正则对象，所以，加上`o`修饰符就正适合：
```ruby
reg="root"

File.new("/etc/passwd").each do |line|
  puts line if /#{reg}/o =~ line
end
```

但是使用`o`修饰符的时候必须注意，因为它只执行一次内插表达式，所以即使内插的结果变化了也不会改变正则表达式。这在大多数时候是没问题的，但是如果是一个服务型程序，它可能需要长时间处于运行状态，如果需要更换正则对象，则不能使用`o`修饰符。

例如：
```ruby
def match_rand_weekday
  day = %w(Mon Tue Wed Thu Fri Sat Sun)[rand(7)]

  DATA.each do |line|
    puts line if /^#{day}/o =~ line
  end
  DATA.rewind    # 重置文件指针
end

5.times { match_rand_weekday }

__END__
Monday
Tuesday
Wednesday
Thursday
Friday
Saturday
Sunday
```

上面的变量day是随机变化的，将其内插在正则源字符串中，如果使用了`o`修饰符，将失去day变量的随机性，所以上面调用5次该方法的结果均相同。
```bash
$ ruby abc.rb
Thursday
Thursday
Thursday
Thursday
Thursday
```

而如果把`o`修饰符去掉，则能保证匹配的随机性：
```bash
$ ruby abc.rb
Monday
Friday
Monday
Tuesday
Sunday
```

## Ruby支持的正则语法

### 字符类

```
[anystr]
[^anystr]
[[:digit:]]  - [0-9]
[[:lower:]]  - [a-z]
[[:upper:]]  - [A-Z]
[[:alnum:]]  - [0-9a-zA-Z]
[[:alpha:]]  - [a-zA-Z]
[[:xdigit:]] - hexadecimal number (i.e., 0-9a-fA-F)
[[:blank:]]  - Space or tab
[[:space:]]  - Whitespace character ([:blank:], newline, carriage return, etc.)
[[:cntrl:]]  - Control character
[[:graph:]]  - Non-blank character
[[:print:]]  - Like [:graph:], but includes the space character
[[:punct:]]  - Punctuation character
[[:word:]]   - [[:alnum:]_]

\w - 单词字符，等价于([a-zA-Z0-9_])
\W - 非单词字符，等价于([^a-zA-Z0-9_])
\d - 数字字符，等价于([0-9])
\D - 非数字字符，等价于([^0-9])
\h - 16进制数字，等价于([0-9a-fA-F])
\H - 非16进制数字，等价于([^0-9a-fA-F])
\s - 空白字符，等价于([ \t\r\n\f\v])
\S - 非空白字符，等价于([^ \t\r\n\f\v])
```

默认情况下，`.`无法匹配换行符，当开启`m`修饰符的时候才可以匹配换行符。但是，这些反斜线序列提供了另外一种技巧来匹配换行符，也就是可以匹配任意字符的方式：使用`[\d\D]`或`[\w\W]`或`[\h\H]`或`[\s\S]`代替元字符`.`。

例如，匹配任意长度的任意字符(包括换行符)：
```ruby
/[\d\D]*/.match("hello\n\nworld\n woniu")
#=> #<MatchData "hello\n\nworld\n woniu">

/[\w\W]*/.match("hello\n\nworld\n woniu")
#=> #<MatchData "hello\n\nworld\n woniu">

/[\h\H]*/.match("hello\n\nworld\n woniu")
#=> #<MatchData "hello\n\nworld\n woniu">

/[\s\S]*/.match("hello\n\nworld\n woniu")
#=> #<MatchData "hello\n\nworld\n woniu">
```

Ruby的正则引擎Onigmo还支持在字符类中使用`&&`做交集操作，他的优先级非常低，只比`[^]`高。例如：
```ruby
/[a-z&&[^c-x]y]/   #=> 等价于[abz]
```

### 量词：重复次数

```
* - Zero or more times
+ - One or more times
? - Zero or one times (optional)
{n} - Exactly n times
{n,} - n or more times
{,m} - m or less times
{n,m} - At least n and at most m times
```

默认是贪婪匹配(也就是匹配优先)，在这些量词后面加上符号可以改变匹配模式：
```
?：在量词后加上?改变成lazy匹配模式，即非贪婪模式，也叫忽略优先
+：在量词后加上+改变成占有优先匹配模式，即在贪婪的基础上不进行回溯
```

所以：
```
                (量词后加上?)      (量词后加上+)
  贪婪匹配量词    非贪婪匹配量词    占有优先匹配量词
----------------------------------------------
      *             *?               *+
      ?             ??               ?+
      +             +?               ++
      {M,}          {M,}?            {M,}+
      {,N}          {,N}?            {,N}+
      {M,N}         {M,N}?           {M,N}+
      {N}           {N}?             {N}+
```

例如：
```ruby
"hello world".match(/\w*/)
#=> #<MatchData "hello">

"hello world".match(/\w*?/)
#=> #<MatchData "">

"hello world".match(/\w+?/)
#=> #<MatchData "h">

"hello world".match(/\w{3,}?/)
#=> #<MatchData "hel">
```

### 分组、捕获

- 小括号`(pat)`可以进行分组捕获。例如`"hello world" =~ /(hel).*(world)/`
- 在正则内部，可以使用`\N`这种方式进行反向引用对应的捕获。例如`"abcDabc" =~ /(abc)D\1/`
- 在正则外部，可以使用`$N`这种方式引用捕获到的分组。例如`"abcDabc" =~ /(abc)D\1/`匹配后，之后可以使用`$1`来引用括号匹配的内容，即『abc』  
- 可以进行命名捕获，使用`(?<NAME>pat)`替代普通的分组括号`(pat)`即可。例如`"hello world" =~ /(?<h>hel).*(?<w>world)/`，这里两个命名分组捕获，名称分别是『h』和『w』  
- 在正则内部，可以通过`\k<NAME>`的方式引用对应的命名捕获。例如`"abcDabc" =~ /(?<x>abc)D\k<x>/`
- 在正则外部，可以通过MatchData对象的hash索引方式获取到命名捕获的分组，或者通过`$N`也可以获取到命名捕获的分组，参见后文
- 可以只分组不捕获，使用`(?:pat)`替代小括号`(pat)`即可，因为是只分组不捕获，所以它不会设置相关变量，无法通过`\N`或`$N`的方式来引用该括号对应的分组
- 可以固化分组，使用`(?>pat)`替代小括号`(pat)`即可。固化分组和占有优先功能上是等价的，都是占有了就不再回溯

特别要注意的是，在Ruby中，**命名捕获和普通的小括号分组捕获功能不能共用**(其它语言可以)。当正则中使用了命名捕获，小括号将只分组不捕获，所以无法通过`\N`或`$N`来引用小括号的分组，因为根本就没有捕获，不会设置对应的变量。

### 锚定

锚定表示匹配位置而不匹配字符本身，也就是说，锚定的作用是定位。
```
^  - 匹配行首
$  - 匹配行尾
\A - 匹配字符串开头，不包括换行符后的位置
\Z - 匹配字符串结尾。可以是换行符前的位置，也可以是绝对结尾
\z - 匹配字符串结尾，允许包括换行符，即字符串的绝对结尾
\b - 匹配word边界
\B - 匹配非word边界
\G - 强制从位移指针处进行匹配，全局匹配时有效，参见全局匹配相关内容

# 环视、断言
(?=pat) - 正向环视断言: 右边的字符必须能匹配pat
(?!pat) - 正向环视断言取反: 右边的字符必须不能匹配
(?<=pat) - 逆向环视断言: 左边的字符必须能匹配pat
(?<!pat) - 逆向环视断言取反: 左边的字符必须不能匹配pat
```

**逆向环视的表达式必须只能表示固定长度的字符串**，例如`(?<=word)或(?<=word|words)`可以，但`(?<=word?)`或`(?<=word*)`不可以，因为长度不定。在Ruby中，长度不定的逆向环视表达式可重写为二选一模式`(?<=word|words)`。此时可使用`\K`来间接实现变长字符的逆向环视，见下文。

关于匹配行首、行尾、字符串首、字符串尾的几种方式：

![](/img/ruby/1565455836813.png)

```ruby
# ^匹配任意行首
/^w/m.match("hello\nworld\n")
#=> #<MatchData "w">
/^h/m.match("hello\nworld\n")
#=> #<MatchData "h">

# \A匹配字符串绝对开头
/\A./.match("hello\nworld\n")
#=> #<MatchData "h">

# $匹配任意行尾
/o$/m.match("hello\nworld\n")
#=> #<MatchData "o">
/d$/m.match("hello\nworld\n")
#=> #<MatchData "d">

# \z匹配字符串绝对结尾
/.\z/m.match("hello\nworld\n")
#=> #<MatchData "\n">

# \Z匹配字符串绝对结尾或换行符前的位置
/\n\Z/m.match("hello\nworld\n")
#=> #<MatchData "\n">
/.\Z/m.match("hello\nworld\n")
#=> #<MatchData "d">
```

### 条件分组

```
(?(cond)yes-subexp), (?(cond)yes-subexp|no-subexp)
```

当cond匹配成功时，接下来再匹配yes-subexp，当cond匹配失败时，接下来再匹配no-subexp，如果没有给定no-subexp，则跳过。

`cond`是一个分组引用，例如`(1)`表示第一个分组是否匹配成功，`(<name>)`表示命名分组name是否匹配成功。

例如：
```ruby
words = %w(<hello> bye bad> <good> 42 <3)

# 等价于\A(?:<\w+>|\w+)\z
words.grep(/\A(<)?\w+(?(1)>)\z/)
#=> ["<hello>", "bye", "<good>", "42"]

words = ['(hi)','good-bye','bad','(42)','-oh','i-j','(-)']

# 等价于/\A(?:\(\w+\)|\w+-\w+)\z/ 或 /\A(?:\((\w+)\)|\g<1>-\g<1>)\z/
words.grep(/\A(?:(\()?\w+(?(1)\)|-\w+))\z/)
#=> ["(hi)", "good-bye", "(42)", "i-j"]
```

### \K丢弃已匹配内容

使用`\K`可丢弃已匹配的内容。`\K`传达的含义是：你必须得有，但是我不要。

例如：
```ruby
/hello \K\w+/ =~ "hello world"
```

上面的正则表示：必须得匹配`hello `，但匹配的这部分内容被丢弃掉，因此最终匹配保存的字符是`world`。

因此，通过`\K`可以间接实现变长的逆向环视锚定。
```ruby
/h.*o \Kworld/ =~ "hello world"
```

这表示world的左边必须能匹配`h.*o `。

### 否定分组

否定分组是一种位置锚定技巧，而不是一种正则语法。它的用法为`((?!xxx).)`，用于取代匹配任意字符的`.`，其外层括号的主要目的是将`(?!xxx)`和`.`组合在一起。

例如`(?:(?!xxx).)*`表示右边不能是xxx，然后再匹配任意单个字符。其效果等价于**匹配任意单个字符，直到右边是xxx的字符，且不要求xxx存在**，即`.*(?!xxx).`。

例如`/(?:(?!cat).)*/ =~ "abcat"`，它的匹配过程如下：
- 首先锚定起始位置的右边不是cat，于是吞掉一个字符a  
- 进行下一轮匹配，锚定a右边的不是cat，于是吞掉字符b  
- 再锚定b右边的不是cat，但锚定失败，于是本轮匹配失败，但注意，b在上轮匹配中已经被吞掉  
- 所以最终匹配的结果是ab，因此`/(?:(?!cat).)/`相当于匹配任意单个字符，直到右边是cat字符  

否定分组常常需要结合位置锚定一起使用才能有比较好的效果。

再分析一下下面的匹配结果：
```ruby
# 匹配cat前的所有字符
'fox,cat,dog,parrot'[/\A((?!cat).)*/]
#=> "fox,"

# 匹配parrot前的所有字符
'fox,cat,dog,parrot'[/\A((?!parrot).)*/]
#=> "fox,cat,dog,"

# 匹配pig前的所有字符
'fox,cat,dog,parrot'[/\A((?!pig).)*/]
#=> "fox,cat,dog,parrot"

# 匹配连续重复字符前的所有字符
'fox,cat,dog,parrot'[/\A(?:(?!(.)\1).)*/]
#=> "fox,cat,dog,pa"

# cat((?!do).)*匹配从cat开始直到do前面的一个字符，即"cat,"
# 然后还要紧跟着匹配par
'fox,cat,dog,parrot'[/cat((?!do).)*par/]
#=> nil
```

除了上述技巧，否定分组还常用于如下匹配需求：  
- (1).左边任意位置(相邻或不相邻)不能有某字符(串)  
- (2).右边任意位置不能有某字符(串)  

即类似于环视锚定的需求。需求(2)完全可以使用正向环视锚定来匹配，而需求(1)不一定能通过逆向环视锚定来匹配，因为逆向环视锚定要求字符数量是固定的。`\K`能实现变长字符的逆向环视锚定，但`\K`会丢弃匹配结果。

因此，否定分组一般只用于实现需求(1)，且不丢弃匹配结果。

例如，想要匹配dog左边没有cat的字符串：
```ruby
# 匹配失败
# \A((?!cat).)*匹配cat前任意长度的字符，然后紧跟着要匹配dog
/\A((?!cat).)*dog/ =~ 'fox,cat,dog,parrot'

# 匹配成功
# 只有当pig不在dog左边时，才能匹配成功
/\A((?!pig).)*dog/ =~ 'fox,cat,dog,parrot'
```

### absence operator

Ruby支持一种称为absence operator的正则语法`(?~xxx)`。它是Ruby引擎Onigmo独有的正则语法，Perl正则不支持该功能。

`(?~xxx)`是作用类似于否定分组，但`(?~xxx)`可同时测试左右两边：表示匹配不包含xxx的字符串。

理解否定分组后，下面示例将很容易理解清楚。
```ruby
# 匹配失败
# 等价于/at((?!do).)*par/
# 即匹配那些at和par中间没有do的字符串
'fox,cat,dog,parrot'.match?(/at(?~do)par/)

# 匹配成功：匹配at和par中间没有go的字符串
'fox,cat,dog,parrot'.match?(/at(?~go)par/)

'fox,cat,dog,parrot'[/at(?~go)par/]
#=> "at,dog,par"

# 匹配/**/方式的注释内容
/ \/\*  (?~\*\/)  \*\/ /x
```

上面示例中，`(?~xxx)`的左右两边都有限定，但当它的左边或右边没有其他限定时，很容易理解错误：
```ruby
# 匹配不包含abc的字符串，可匹配"","ab","abd","acd"等
/(?~abc)/

# 匹配成功
# (?~hello)表示匹配不包含hello的字符，例如"","ello","hllo","abc"
# (?~hello)左边没有限定，右边有限定匹配" and world"
# 因此，整个正则可匹配" and world","abc and world","ello and world"
# 还可以匹配"hello and world"，因为其左边没有限定
"hello and world"[/(?~hello) and world/]
#=> ello and world

# 匹配成功
# (?~world)可匹配"","wor","worl","orld"
# 但右边没有做限定，因此可匹配world的"worl"部分
"hello and world"[/hello and (?~world)/]
#=> "hello and worl"
```

因此，当正则表达式里面使用了`(?~xxx)`且左右两边没有都做限定匹配时，应当将该部分当作一个(正则元素)整体进行分析。


## 正则匹配的返回结果

当使用`=~`进行匹配时，如果匹配成功，则返回匹配到的第一个字符的索引位置，如果匹配失败则返回nil。

当使用`match()`进行匹配时，如果匹配成功，则返回一个MatchData对象，如果匹配失败，则返回nil。

```ruby
"hello world" =~ /hello/     #=> 0
/hello/ =~ "hello world"     #=> 0
/HELLO/ =~ "hello world"     #=> nil

/hello/.match("hello world")
#=> #<MatchData "hello">
"hello world".match(/hello/)
#=> #<MatchData "hello">
"hello world".match(/HELLO/)
#=> nil
```

其实使用`=~`也可以获取到MatchDate对象，只不过它将MatchData赋值给一个全局变量`$~`。由于是一个代表MatchData的全局变量，所以匹配成功和匹配失败都会设置该变量，而且匹配失败时会将其设置为nil。
```ruby
"hello" =~ /hello/  #=> 0
$~       #=> #<MatchData "hello">

"HELLO" =~ /hello/  #=> nil
$~                  #=> nil
```

此外，无论是`=~`还是match方法匹配，在匹配之后都可以通过`Regexp.last_match`这个类方法来获取最近匹配时设置的`MatchData`，如果匹配成功last_match返回刚才的MatchData对象，如果匹配失败就返回nil。
```ruby
"hello world" =~ /(?<h>hel.*) (?<w>world)/
Regexp.last_match
#=> #<MatchData "hello world" h:"hello" w:"world">

"hello world" =~ /(hel.*) (world)/
Regexp.last_match
#=> #<MatchData "hello world" 1:"hello" 2:"world">
```

那么MatchDate这个对象到底是什么东西呢？

它是匹配成功时设置的，如果匹配失败，任何想要获取该对象的方式会返回nil。这个对象中包含了匹配成功后的一些状态信息，比如匹配了具体哪些字符，从哪个位置开始匹配的，到哪里匹配介绍等等。所以，可以操作这个对象来获取一些匹配成功后的内容。

下面单独介绍和MatchData相关的内容，包括如何操作正则匹配结果。

## MatchDate对象

首先，如何获取MatchData对象？根据前面的介绍，简单总结下：
1. 任何正则匹配操作后，通过`Regexp.last_match`方法获取
2. 任何正则匹配操作后，通过`$~`全局变量来获取
3. str.match()或reg.match()匹配成功后返回

MatchData对象包含了很多匹配相关的状态，看看下面这个MatchData对象的输出结果：
```ruby
"hello world" =~ /(?<h>hel.*) (?<w>world)/
Regexp.last_match
#=> #<MatchData "hello world" h:"hello" w:"world">
```

其中`#<MatchData "hello world"`的『hello world』是正则匹配成功的原始字符串部分。`h`和`w`是命名分组捕获对应的名称。可以通过这个名称来获取各命名分组对应的匹配结果，因为MatchData使用hash结构设置各分组对应的匹配结果，分组名称就是hash结构中的key：
```ruby
"hello world" =~ /(?<h>hel.*) (?<w>world)/
Regexp.last_match["h"]   #=> "hello"
Regexp.last_match["w"]   #=> "world"
```

但同时也能将MatchData看作是数组结构，比如按一些数组的方式去索引相关分组数据。例如：
```ruby
"hello world" =~ /(?<h>hel.*) (?<w>world)/
Regexp.last_match[0]  #=> "hello world"
Regexp.last_match[1]  #=> "hello"
Regexp.last_match[2]  #=> "world"

# last_match方法也支持直接使用参数来获取对应的分组
"hello world" =~ /(?<h>hel.*) (?<w>world)/
Regexp.last_match(0)  #=> "hello world"
Regexp.last_match(1)  #=> "hello"
Regexp.last_match(2)  #=> "world"
```

1号索引保存的是第一个分组捕获的内容，2号索引保存的是第二个分组捕获的内容。0号索引保存的是所匹配到的所有字符串，也就是`#<MatchData "hello world"`的『hello world』部分。

当使用数值去获取MatchData对象中的分组捕获内容时，甚至还支持负数索引、范围、指定长度的方式：
```ruby
md = "hello".match(/(\w)(\w)(\w)(\w)(\w)/)
md[1..-1]   #=> ["h", "e", "l", "l", "o"]
md[1,3]     #=> ["h", "e", "l"]
```

从1号索引开始，也可以使用`$N`来获取对应的分组内容。例如：
```ruby
"hello world" =~ /(?<h>hel.*) (?<w>world)/
$&  #=> "hello world"
$1  #=> "hello"
$2  #=> "world"
```

但不能使用`$0`来表示0号索引的分组内容，因为Ruby中的`$0`代表的是ruby程序的名称。但Ruby为此提供了`$&`来获取0号分组的内容。

前面说了，`$~`也对应MatchData对象，所以也可以通过该变量来获取相关信息：
```ruby
# 等价的$~也表示MatchData
$~["h"]
$~["w"]
$~[0]
$~[1]
$~[2]
```

从MatchData中还支持获取匹配前的内容(pre_match)、匹配后的内容(post_match)：
```ruby
"good hello world bye".match(/hello world/)
#=> #<MatchData "hello world">
$~.pre_match
#=> "good "
$~.post_match
#=> " bye"
```

此外，也能使用一些全局变量和其它一些方法获取匹配到的各部分内容。这里做个归纳总结：

```
# 下面的md表示MatchData对象
获取MatchData对象      ：match()、$~、last_match()
获取0号分组捕获的内容    ：md[0]、$&、last_match(0)
获取N号(N>=1)捕获的内容 ：md[N]、$N、last_match(N)
获取最后一个捕获的内容   ：md[-1]、$+
获取本次匹配内容前面的内容：md.pre_match()、$`
获取本次匹配内容后面的内容：md.post_match()、$'
```

所以，这里还可以对各种全局变量做个总结：
```
$~ : 获取MatchData对象
$& : 0号分组捕获内容
$N : N号分组捕获的内容
$+ : 最后一个分组捕获的内容
$` : 匹配内容前的内容
$' : 匹配结果后的内容
```

### 从MatchData获取更多信息

**1.begin方法获取每个捕获部分在原始字符串中开始匹配的索引位置**
**2.end方法获取每个捕获部分在原始字符串中结束匹配的索引位置**
**3.offset方法获取每个捕获部分在原始字符串中开始和结束匹配的索引位置，以数组方式返回**

```ruby
md = "good hello world bye".match(/(hello) (world)/)
#=> #<MatchData "hello world" 1:"hello" 2:"world">
md.begin(0)    #=> 5
md.begin(1)    #=> 5
md.begin(2)    #=> 11

md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
#=> #<MatchData "hello world" h:"hello" w:"world">
md.begin(0)    #=> 5
md.begin(0)    #=> 5
md.begin(1)    #=> 5
md.begin(2)    #=> 11
md.begin("h")  #=> 5
md.begin("w")  #=> 11

md.end(0)      #=> 16
md.end(1)      #=> 10
md.end(2)      #=> 16
md.end('h')    #=> 10
md.end('w')    #=> 16

md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
#=> #<MatchData "hello world" h:"hello" w:"world">
md.offset(0)   #=> [5, 16]
md.offset(1)   #=> [5, 10]
md.offset(2)   #=> [11, 16]
md.offset("h") #=> [5, 10]
md.offset("w") #=> [11, 16]
```

**4.named_captures方法获取所有命名捕获及其捕获到的分组内容，以hash方式返回**
**5.names方法获取所有命名捕获分组的名称，以数组方式返回**
**6.captures方法获取所有分组捕获(可以是命名捕获的分组，也可以是普通捕获的分组)的内容，以数组方式返回**

```ruby
# named_catpures
md
#=> #<MatchData "hello world" h:"hello" w:"world">
md.named_captures
#=> {"h"=>"hello", "w"=>"world"}

# names
md
#=> #<MatchData "hello world" h:"hello" w:"world">
md.names   #=> ["h", "w"]

# captures
md = "good hello world bye".match(/(hello) (world)/)
md.captures  #=> ["hello", "world"]

md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
md.captures  #=> ["hello", "world"]
```

**7.values_at方法获取指定索引位置处的捕获内容，可以是数值索引也可以是字符串索引**
```ruby
md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
#=> #<MatchData "hello world" h:"hello" w:"world">
md.values_at(1,2,1)
#=> ["hello", "world", "hello"]
md.values_at('h','w','h')
#=> ["hello", "world", "hello"]
```

**8.length方法和size方法(它们是别名)获取所有捕获的分组数量，即MatchData保存捕获分组的数组的长度**

```ruby
md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
#=> #<MatchData "hello world" h:"hello" w:"world">
md.length  #=> 3
```

**9.regexp方法返回原始的正则表达式对象**

```ruby
md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
md.regexp
#=> /(?<h>hello) (?<w>world)/
```

**10.string方法返回正则所匹配的源字符串，即对哪个字符串做匹配操作，它的返回结果是frozen状态的**

```ruby
md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
md.string  #=> "good hello world bye"
```

**11.to_s方法输出0号索引的内容**
```ruby
md = "good hello world bye".match(/(?<h>hello) (?<w>world)/)
md.to_s  #=> "hello world"
```


## 判断是否匹配成功

由于`=~`匹配成功返回索引位置，匹配失败返回nil，match()匹配成功返回MatchData对象，匹配失败返回nil。

也就是说，只要匹配失败就返回nil，只要匹配成功就返回一个非nil值，于是可以通过这一点来判断正则是否匹配成功。

```ruby
str = "hello world"
puts str if str =~ /hello/
puts str if str.match(/hello/)
```

其实String和Regexp都提供了一个名为`match?`的方法，直接在匹配时返回true或false。
```
match?(str) → true or false
match?(str,pos) → true or false
```

例如：
```ruby
str = "hello world"
puts str if str.match?(/hello/)
```

既然有了`match`还要`match?`干嘛？其实match匹配成功后会做一大堆的后续操作，比如构建一个MatchData对象，设置可能存在的分组等，而`match?`则只做一件事，那就是匹配，匹配成功后不会做后续的一大堆操作。

所以如果只是判断是否匹配成功的话，那么`match?`的效率比`match`要更高。

当然，对于`$_`的匹配来说，使用`if ~ /reg/`来判断即可，在Ruby一行式的命令行中用的比较多。


## 如何做全局匹配

默认情况下，Ruby的正则匹配都只匹配一次就退出：
```ruby
"abc ABC" =~ /abc/i
```

上面的字符串中，『abc』和『ABC』都能被正则表达式`/abc/i`匹配成功，但是它只会匹配abc，而不会匹配ABC。

在Ruby中并没有提供g修饰符，所以Ruby的正则表达式没办法直接做全局匹配。但是，String提供了一个名为scan的方法，它可以做全局正则匹配。

```ruby
scan(pattern) → array
scan(pattern) {|match, ...| block } → str
```

按照正则表达式匹配字符串，从前向后每次匹配到的结果放进数组或传递到代码块。

如果没有使用分组捕获，则从前向后每次匹配到的内容都作为数组的元素或直接传递给代码块。

如果使用了分组捕获，则正则每次匹配的分组放进子数组中。

```ruby
a = "cruel world"
a.scan(/\w+/)        #=> ["cruel", "world"]
a.scan(/.l/)         #=> ["el", "rl"]
a.scan(/.../)        #=> ["cru", "el ", "wor"]
a.scan(/(...)/)      #=> [["cru"], ["el "], ["wor"]]
a.scan(/(..)(..)/)   #=> [["cr", "ue"], ["l ", "wo"]]

a.scan(/\w+/) {|w| print "<<#{w}>> " }
   #=> <<cruel>> <<world>>
a.scan(/(.)(.)/) {|x,y| print y, x }
   #=> rceu lowlr
```

## 替换和全局替换

String类中定义了四个方法：`gsub`、`gsub!`、`sub`、`sub!`，它们可以用来做字符串的替换，就像sed命令一样。其中gsub是全局替换。

```ruby
sub(pattern, replacement) → new_str
sub(pattern, hash) → new_str
sub(pattern) {|match| block } → new_str

sub!(pattern, replacement) → str or nil
sub!(pattern) {|match| block } → str or nil

gsub(pattern, replacement) → new_str
gsub(pattern, hash) → new_str
gsub(pattern) {|match| block } → new_str
gsub(pattern) → enumerator

gsub!(pattern, replacement) → str or nil
gsub!(pattern, hash) → str or nil
gsub!(pattern) {|match| block } → str or nil
gsub!(pattern) → an_enumerator
```

gsub用来做全局字符串替换。sub只做一次替换。

pattern部分是正则表达式对象，但也可以是双引号包围的正则字符串，但不建议。所以，应该遵从使用`/pattern/`的方式作为pattern参数的格式。

replacement表示要替换被pattern所匹配的内容。在replacement中，可以使用反向引用`\N`、分组捕获的分组引用`\k<NAME>`。replacement部分由单引号或双引号包围，如果双引号包围，那么其中的反斜线要多加一个前缀`\`转义。

使用hash参数时，表示pattern匹配的内容是hash中的某个key，那么将根据hash中的key来对应替换。

使用语句块时，将传递所匹配的内容到代码块中，这时会自动设置好`$1`, `$2`, `` $` ``, `$&`, `$'`等变量。

对于`gsub!`，如果没有做任何替换，则返回nil。

```ruby
# gsub
"hello".gsub(/[aeiou]/, '*')                  #=> "h*ll*"
"hello".gsub(/([aeiou])/, '<\1>')             #=> "h<e>ll<o>"
"hello".gsub(/./) {|s| s.ord.to_s + ' '}      #=> "104 101 108 108 111 "
"hello".gsub(/(?<foo>[aeiou])/, '{\k<foo>}')  #=> "h{e}ll{o}"
'hello'.gsub(/[eo]/, 'e' => 3, 'o' => '*')    #=> "h3ll*"
```

```ruby
# sub
"hello".sub(/[aeiou]/, '*')                  #=> "h*llo"
"hello".sub(/([aeiou])/, '<\1>')             #=> "h<e>llo"
"hello".sub(/./) {|s| s.ord.to_s + ' ' }     #=> "104 ello"
"hello".sub(/(?<foo>[aeiou])/, '*\k<foo>*')  #=> "h*e*llo"
'Is SHELL your preferred shell?'.sub(/[[:upper:]]{2,}/, ENV)
    #=> "Is /bin/bash your preferred shell?"
```

必须注意的是，如果sub或gsub没有使用语句块，那么特殊全局变量`$X`不能在replacement中使用。如果在replacement中使用这些全局特殊变量，将被替换成上次匹配遗留下来的值，或直接替换成空(之前没有进行过匹配操作)。

```ruby
"abcABC".match(/(abc)(ABC)/)
#=> #<MatchData "abcABC" 1:"abc" 2:"ABC">
$1    #=> "abc"
$2    #=> "ABC"

"0xy0".sub(/(x)(y)/,"#{$2}#{$1}")
#=> "0ABCabc0"
$1    #=> "x"
$2    #=> "y"
```

这是因为replacement也是这几个方法的参数，和第一个参数pattern是同级别的存在。而参数的评估先于方法的执行，所以不会先执行匹配操作再评估replacement部分。所以评估replacement的时候，如果发现了特殊全局变量，将直接进行变量的替换(要么替换成上次匹配遗留下来的值，要么替换成空)，因此此时pattern部分设置的全局变量将跟此次replace操作无关。

使用语句块则没有该问题，因为执行语句块的时候，匹配操作已经完成了，sub或gsub将匹配到的字符串(即`$0`的值)传递给语句块变量，而且到执行语句块的时候，其它特殊全局变量也都已经设置好了，所以能直接使用这些特殊的全局变量。
```ruby
$1  #=> nil
$2  #=> nil
"0xAy0".gsub(/(x)A(y)/) {|m| $2+"A"+$1 }
#=> "0yAx0"
```


## \\G的用法

`\G`锚定符号表示只要某次匹配失败，就立即停止，不再继续向后匹配。

**默认在非全局匹配环境下，匹配会从第一个字符位置处开始一直向后匹配，直到匹配成功**。

```ruby
"hello world".match("o")      #=> #<MatchData "o">
"hello world".match("o", 3)   #=> #<MatchData "o">
"hello world".match(/\Go/, 3) #=> nil
"hello world".match(/\Go/, 4) #=> #<MatchData "o">
=> #<MatchData "o">
```

上面前两条匹配语句从第1个或第3个字符开始匹配时，都匹配失败，但它们都继续向后匹配，在此过程中，正则引擎的匹配指针一直在向后移动，直到匹配成功，才停止。

但是使用了`\G`之后，由于第三个字符是`l`，无法匹配`\Go`中的`o`，所以匹配失败，`\G`直接终止本次匹配，而不会向后移动正则引擎的匹配指针，所以返回nil。

**默认在全局匹配环境下，匹配也会在匹配失败的情况下不断向后移动匹配指针，直到到达字符串结尾**。

```ruby
"    a b c".gsub(/ /, '_')    #=> "____a_b_c"
"    a b c".gsub(/\G /, '_')  #=> "____a b c"
# 删除行首和行尾空格
"    a b c   ".gsub(/\G /, '')  #=> "a b c"
```

上面第一条gsub语句会替换所有的空格，即使在匹配字符`a`的时候发现本次匹配失败，也会继续移动匹配指针向后继续匹配。

但使用了`\G`后，将会在匹配失败的时候立即停止，而不会继续向后移动指针。上面第二条gsub语句只替换了前面4个空格，就是因为在匹配字母a的时候发现已经失败了，`\G`直接导致它停止匹配。

<a name="recur_regexp"></a>

## \\g递归匹配

一般来说，递归的正则表达式用来匹配任意嵌套层次的结构或左右对称的结构。例如匹配：
```
((((()))))
(hello (world) good (boy) bye)
<p>hello world <strong>hello world</strong> </p>
abc.def.ghij...stu.vwx.yz
abcdcba
123454321
```

递归正则在正则表达式里算是比较灵活的部分，换句话说就是可能会比较难。下面这个正则表达式是在网上流传的非常广泛的递归正则的示例，它用来匹配嵌套任意次数的括号，括号内可以有其它字符，比如可以匹配`(a(bc)de)`、`(abc(bc(def)c)de)`。

```ruby
# 使用了x修饰符，忽略正则表达式内的空白符号
/\( ( (?>[^()]+) | (\g<0>) )* \)/x
```

这似乎看不怎么懂？其实即使知道了正则递归的方式，也还是很难看懂(至少，我分析了很久)。

难懂的原因大概是因为这里使用的固化分组在多选分支`|`中属于一个技巧性的写法，而且分组外还使用了量词`*`，这些结合起来就太难懂了。

正因为网上到处流传这个例子，曾使我多次对递归正则的学习望而却步。这里我也不去解释这个递归正则的含义，因为『太学术化』或者说『太装xxx逼』，**而一般递归正则完全可以写的很简单但却能实现目标**。

如何写出简单易懂版本的递归正则并且理解递归正则的匹配方式，正是本文的目标。在后文，我介绍了一个更加简单、更加容易理解的版本，同样能实现这个递归匹配的需求。

为了解释清楚递归正则，本文会以循序渐进的方式逐步深入到递归正则的方方面面。所以，篇幅可能稍大，其中大量篇幅都用在了解释分析递归正则是如何递归匹配上。

> 注：
> 本文以Ruby的正则表达式来介绍递归正则，但对其它支持递归正则的语言也是能通用的。例如Perl、PHP、Python(自带的re不提供，但第三方库regex提供递归正则)等。

### 理解反向引用\\N和\\g

首先通过正则表达式的反向引用的用法来逐步引入递归正则表达式的用法。

正则表达式`(abc|def) and \1xyz`可以匹配字符串『abc and abcxyz』或『def and defxyz』，但是不能匹配『abc and defxyz』或『def and abcxyz』。这是因为，反向引用在引用的时候，只能引用之前分组捕获成功后的那个结果。

```ruby
reg = /(abc|def) and \1xyz/

reg =~ "abc and abcxyz"  #=>0
reg =~ "def and defxyz"  #=>0
reg =~ "def and abcxyz"  #=>nil
reg =~ "abc and defxyz"  #=>nil
```

但是，如果使用`\g<1>`来代替`\1`，那么就能匹配这四种情形的字符串(Perl中使用`(?1)`对应这里的`\g<1>`)：
```ruby
reg = /(abc|def) and \g<1>xyz/

reg =~ "abc and abcxyz"  #=>0
reg =~ "def and defxyz"  #=>0
reg =~ "def and abcxyz"  #=>0
reg =~ "abc and defxyz"  #=>0
```

`\g<1>`和`\1`的区别在于：`\1`在反向引用的时候，引用的是该分组捕获到的结果值，`\g<1>`则不是反向引用，而是直接将索引号为1的分组捕获重新执行捕获分组的匹配操作。相当于是`/(abc|def) and (abc|def)xyz/`。

所以，`\1`相当于是在引用的位置插入索引号为1的分组捕获的结果，`\g<1>`相当于是在此处插入索引号为1的分组捕获表达式，让其能再次进行分组表达式这部分的匹配操作。

如果把分组捕获表达式看作是函数的定义，那么开始匹配时表示调用该函数进行分组捕获。而反向引用`\N`则是在引用位置处插入该函数的返回值，`\g<name>`则表示在此处再次调用该函数进行匹配。

`\g<name>`的name可以是数值型的分组索引号，也可以是命名捕获的名称索引，还可以是0表示整个正则表达式自身。

```ruby
/(abc|def) and \g<1>xyz/
/(?<var>abc|def) and \g<var>xyz/
/(abc|def) and \g<0>xyz/  # 错误正则，稍后分析

=begin
# Perl、Python(regex，非re)、PHP与之对应的方式：
\g<0>    -> (?R)或(?0)
\g<N>    -> (?N)
\g<name> -> (?P>name)或(?&name)
=end
```

前面两种好理解，第三种使用`\g<0>`就不太能理解了，继续向下看。

### 初探递归正则：递归正则匹配什么

`\g<0>`表示正则表达式自身，所以这相当于是递归正则表达式，假如进行第一轮正则表达式替换的话，相当于：
```
/(abc|def) and (abc|def) and \g<0>xyzxyz/
```

当然，这里只是为了帮助理解才将`\g<0>`替换成正则表达式，但它不会真的直接替换正则表达式的定义。就像函数调用时，不会在调用函数的地方替换成函数定义里的代码再去执行，函数定义了就能多次复用。

不管怎样，不难发现这里已经出现了无限递归的可能性，因为替换一轮后的正则表达式中再次包含了`\g<0>`，它可以再次进行第二轮替换、第三轮替换......

那么，对于`/(abc|def) and \g<0>xyz/`这个递归的正则表达式来说，它能匹配什么样的字符串呢？这才是理解正则递归时最需要关心的。

可以将上面的`\g<0>`看作是一个占位符，首先它可以匹配`abc and _xyz`或者`def and _xyz`这种格式的字符串，这里我用了`_`表示`\g<0>`占位符。递归一轮的话，它可以匹配`abc and def and _xyzxyz`，这里又会继续递归下去，将没完没了。所以这里先将该正则匹配什么字符串的问题保留，稍后再回头分析。

事实上，`/(abc|def) and \g<0>xyz/`是错误的正则表达式，它会提示我们，递归没有终点：
```ruby
/(abc|def) and \g<0>xyz/
#=>SyntaxError: never ending recursion
```

所以，**使用递归正则必须要保证递归能够有终点**。

### 保证正则递归的终点

怎么保证递归正则的终点呢？只要给`\g<>`这部分做一个量词的限定即可，比如：

```ruby
\g<0>+        # 错误正则
\g<0>{3}      # 错误正则
\g<0>{,3}     # 错误正则

\g<0>*        # 正确正则
\g<0>?        # 正确正则
\g<0>{0}      # 正确正则
pat|\g<0>     # 正确正则
(\g<0>)*      # 正确正则
(\g<0>)?      # 正确正则
...
```

`\g<0>+`表示递归至少1轮，但是这里已经错了，因为递归多次的时候，`\g<0>`这个占位符及其量词`+`将始终保留在最后一轮的结果中，于是导致无限递归。同理`\g<0>{3}`这种表示严格递归三次的方式也是错误的，因为递归第三次后仍然保留了`\g<0>{3}`占位符及其量词`{3}`，这也将无限递归。

所以，**只有`\g<0>*`和`\g<0>?`和`\g<0>{0}`和`pat|\g<0>`等这种能在量词数量选择意义上表示递归0次的方式才是正确的正则表达式语法**，因为无论递归多少次，最后一次的占位符的量词都可以是0次，从而达到递归的终点，即停止递归。

所以，修改前面的正则表达式，假如使用`?`量词修饰`\g<>`：
```ruby
/(abc|def) and \g<0>?xyz/
```

### 再探递归正则：递归正则匹配什么

回到之前遗留的问题，现在这个正确的递归正则表达式`/(abc|def) and \g<0>?xyz/`能匹配什么样的字符串呢？

按照之前的分析，它能匹配的字符串的模式类似于`abc and _?xyz`或者`def and _?xyz`。

如果量词`?`取0次，那么该递归正则匹配的是『abc and xyz』或『def and xyz』：
```ruby
reg = /(abc|def) and \g<0>?xyz/
reg =~ "abc and xyz"  #=> 0
reg =~ "def and xyz"  #=> 0
```

如果量词`?`取1次，那么该递归一轮后的正则模式为`abc and abc and _?xyzxyz`，其中任何一个『abc』替换成『def』都是满足条件的。那么这里又有了`\g<>`量词的次数选择问题。

假如这里量词`?`取0次，也就是从开始到现在总体递归了一轮。那么该递归正则匹配到是：
```ruby
reg = /(abc|def) and \g<0>?xyz/
reg =~ "abc and abc and xyzxyz"  #=> 0
reg =~ "abc and def and xyzxyz"  #=> 0
reg =~ "def and def and xyzxyz"  #=> 0
reg =~ "def and abc and xyzxyz"  #=> 0
```

如果递归一轮后的量词`?`继续取1次呢？那么下一轮递归仍将会有量词次数选择的问题。

至此，应该理解了递归正则的基本匹配方式。不过这里使用的`\g<0>`递归还很基础，下面将继续逐步深入。

### 深入递归(1)：括号分组内的\\g

前面的递归示例中是将能表示递归的表达式`\g<0>`部分放在分组的外面，这种情况下，只有`\g<0>`这种形式才能算是递归，如果是`\g<1>`或`\g<name>`，就算不上是递归，充其量也就是个表达式的调用。

但是，当需要使用递归正则来解决问题的时候，递归表达式往往是在分组内部而不是在分组外部的。所以，前面解释的递归方式其实非常少见。于是，要使用递归正则，还得继续深入探索。

首先看一个非常简单的组内递归正则表达式：
```ruby
/(abc\g<1>?xyz)+/
```

这个表达式中，进行了一个分组捕获，这个分组首先匹配`abc`字符，然后在分组捕获内使用了表达式`\g<1>?`(注意这个`?`是不能少的，当然`?`也可以换成其它的前面解释过的量词)，紧随其后的是匹配字符`xyz`。由于这里的`\g<1>?`放在1号索引对应的分组捕获的内部，所以就形成了一个递归的正则表达式。

问题是，这个正则表达式能匹配什么样的字符串呢？要学会递归正则表达式，必须会分析它能够匹配什么类型的字符串。

仍然，以占位符的方式来表示`\g<1>`，那么该递归正则表达式匹配的字符串模式为：`"abc_?xyz" * N`，这个`* N`表示重复N次，因为这种表达式的括号分组外面有一个`+`符号。

如果量词`?`选择为0次，也就是不进行递归，则匹配字符串`"abcxyz" * N`：
```ruby
/(abc\g<1>?xyz)+/ =~ "abcxyz"  #=> 0

/(abc\g<1>?xyz)+/ =~ "abcxyzabcxyz"
#=> 0

/(abc\g<1>?xyz)+/ =~ "abcxyzabcxyzabcxyz"
#=> 0

/(abc\g<1>?xyz)+/ =~ "abcxyz" * 10
#=> 0
```

如果量词`?`选择为1次，那么进行一轮递归后，匹配的字符串模式为：`"abcabc_?xyzxyz" * N`。再次进行`?`量词的次数选择，假如选0次，那么匹配的字符串是`"abcabcxyzxyz" * N`：
```ruby
/(abc\g<1>?xyz)+/ =~ "abcabcxyzxyz" #=> 0
/(abc\g<1>?xyz)+/ =~ "abcabcxyzxyzabcabcxyzxyz"
#=> 0
/(abc\g<1>?xyz)+/ =~ "abcabcxyzxyz" * 3
#=> 0
```

再继续分析一轮递归。假设这是`?`量词选择1次，那么进行第二轮的递归，匹配的字符串模式为：`"abcabcabc_?xyzxyzxyz" * N`。

至此，应该不难推测出递归正则表达式`/(abc\g<1>?xyz)+/`匹配的字符串的模式：
```ruby
"abcxyz" * N
"abcabcxyzxyz" * N
"abcabcabcxyzxyzxyz" * N
# 归纳后，即匹配如下通用模式：n和N均大于等于1
("abc" * n + "xyz" * n) * N
```

将目光集中于刚才的递归正则表达式`/(abc\g<1>?xyz)+/`，如何能通过这个正则表达式直接推测匹配何种类型字符串呢？

量词`+`或其它可能的量词先不看，先将焦点放在分组捕获。这个分组捕获匹配的是`abc_?xyz`，如果要进行递归N轮，那么每一轮都是`abc_?xyz`这种模式，直接将其替换到该正则中去观察：`abc(abc_?xyz)*xyz`，其中`(abc_?xyz)*`表示这部分重复0或N次。当然替换后的这部分不是标准的正则，只是为了有助于理解才将不同地方的概念混在一起，我想并不会对你的理解造成歧义。

这样理解起来就不难了。当然这个递归正则比较简单，如果把上面的`\g<1>?`换成`\g<1>*`，看上去又会更复杂一点。那么它匹配什么样的字符串呢？

同样的分析方式，将`/(abc\g<1>*xyz)+/`看作是`"abc_*xyz" * N`的结构，然后对`*`取值，假设取值3次，所以递归后的结果看上去类似于：
```ruby
"abc(abc_*xyz)(abc_*xyz)(abc_*xyz)xyz" * N
```

上面的每个括号里都可以对量词`*`做选择，但要到达递归的终点，最后(可能是递归了好多轮后)每一个递归里的`*`都必须取值0次才能终结这个递归。

所以，假如现在这3个括号里的每个`*`都选择0次，那么匹配的字符串模式类似于：
```ruby
"abc(abcxyz)(abcxyz)(abcxyz)xyz" * N

# 即等价于：n和N均大于等于1
( "abc" + "abcxyz" * n + "xyz" ) * N
```

例如:
```ruby
/(abc\g<1>*xyz)+/ =~ ( "abc" + "abcxyz" * 1 + "xyz" ) * 1
#=> 0
/(abc\g<1>*xyz)+/ =~ ( "abc" + "abcxyz" * 1 + "xyz" ) * 2
#=> 0
/(abc\g<1>*xyz)+/ =~ ( "abc" + "abcxyz" * 4 + "xyz" ) * 2
#=> 0
```

假如上面三个括号里第一个括号里的`*`取值1次，后面两个括号里的`*`取值0次，那么再次递归后，匹配的字符串模式类似于：
```ruby
"abc(abc(abc_*xyz)xyz)(abcxyz)(abcxyz)xyz" * N
```

没错，又要做量词的次数选择。假如这次`*`取0次，那么将终结本次递归匹配，它匹配的字符串模式为：
```ruby
"abc(abc(abcxyz)xyz)(abcxyz)(abcxyz)xyz" * N
```

那么如果`*`不是按照上面的次数进行选择的，那么匹配的字符串模式是怎样的？

没有答案，唯一准确的答案就是回归这个正则表达式的含义：它匹配的字符串模式为`(abc\g<1>*xyz)+`。

### 深入递归(2)：写递归正则(入门)

前面一直都是根据给定的递归正则表达式去分析能匹配什么样的字符串，这对于理解递归正则有所帮助。但是我们更想要掌握的是如何根据字符串写出递归的正则表达式。

一般来说，要使用递归正则去匹配，往往是要匹配嵌套的一些东西，如果不是匹配嵌套内容，很可能不会想到要去用递归正则。这里，假设也要去匹配嵌套的东西。

先从简单的嵌套开始。比如，如何匹配无限嵌套的空括号`()`、`(())`、`((()))`，即`"(" * n + ")" * n`？

分析一下。如果不递归的话，那就是匹配一对小括号`()`，所以这两小括号字符必须要在分组内，即`(\(\))`。(如果使用`\g<0>`来递归的话，则可以不用在分组内，不过这里先不考虑这种情况。)

按照前文多次对递归正则表达式匹配何种字符串的分析，用占位符替代要递归的话，要匹配的嵌套括号的字符串模式大概是这样的：`(_)`。所以递归表达式`\g<1>`要在`\(`和`\)`的中间，即`(\(\g<1>\))`。

这里还少了个量词来保证递归的终点。那么使用什么样的量词呢？

使用`\g<1>*`肯定没问题，只要`*`号每次递归都只选择量词1次，并且最后一轮递归选择0次终结递归即可，那么匹配的模式是`((_*))`、`(((_*)))`等等，这正好符合嵌套匹配。

```ruby
/(\(\g<1>*\))/ =~ "(" * 1 + ")" * 1
#=> 0
/(\(\g<1>*\))/ =~ "(" * 3 + ")" * 3
#=> 0
/(\(\g<1>*\))/ =~ "(" * 10 + ")" * 10
#=> 0
```

看别人写的递归正则，往往会在分组后加上`*`号量词，即`(\(\g<1>*\))*`，针对于这种模式的嵌套，其实这个`*`是多余的，它要匹配成功，这个量词必须只能选0或1次。如果选择多于1次，那么匹配的字符串模式就变成了`"((_*))" * N`，更标准一点的表示方式是`( "(" * n + ")" * n ) * N`，当然，前面也说了，这还有无数种其他的匹配可能。

所以，在这里我不在分组的后面加`*`或`+`这样的量词。要继续刚才的讨论。

使用`\g<1>?`这种量词方式可以吗？当然可以，上面分析`\g<1>*`的时候，是说当每一轮递归时的`*`次数选择都是1次或0次，就能匹配无限嵌套的小括号。对于`\g<1>?`来说当然也可以，因为`?`也可以表示0或1次。

```ruby
/(\(\g<1>?\))/ =~ "(" * 1 + ")" * 1
#=> 0
/(\(\g<1>?\))/ =~ "(" * 3 + ")" * 3
#=> 0
/(\(\g<1>?\))/ =~ "(" * 10 + ")" * 10
#=> 0
```

这两种递归正则表达式，都是符合要求的，都能匹配无限嵌套的小括号。

下面是命名捕获版本的：
```ruby
/(?<var>\(\g<var>?\))/ =~ "(" * 3 + ")" * 3
#=> 0
```

也能直接使用`\g<0>`作为嵌套表达式，这时甚至可以去掉分组：
```ruby
/(?<var>\(\g<0>?\))/ =~ "(" * 3 + ")" * 3
#=> 0

# 去掉分组，直接递归这种本身
/\(\g<0>?\)/ =~ "(" * 3 + ")" * 3
#=> 0
```

这样看上去，写递归正则好像也不难。其实嵌套模式简单的递归正则确实不难，只要理解递归的含义基本上就能写出来。再看另一个示例。

### 深入递归(3)：写递归正则(进阶)

假设要匹配的字符串模式为：`(abc(d(xy)e)fgh)`，其中每个括号内的字符长度任意。这似乎正是本文开头所举的例子。

这一个递归写起来其实非常非常简单：
```ruby
# 为了可读性，使用了x修饰符忽略表达式内的空白符号
/\( [^()]* \g<0>* [^()]* \)/x

# 匹配：
reg = /\( [^()]* \g<0>* [^()]* \)/x
reg =~ "(abc(d(xy)e)fgh)"  #=> 0
reg =~ "(abc(d(xy)))"      #=> 0
reg =~ "((()e)fgh)"        #=> 0
reg =~ "((()))"            #=> 0
```

其中`\([^()]*`和`[^()]*\)`是头和尾，中间使用`\g<0>`来无限嵌套头和尾。逻辑其实很简单。

相比于网上流传的版本`/\( ( (?>[^()]+) | (\g<0>) )* \)/x`，此处所给出的写法应该容易理解的多。

再回头扩充刚才的递归匹配需求，如果需要匹配的字符串是`ab(abc(d(xy)e)fgh)df`这种模式呢？另一个问题，这种字符串模式和`(abc(d(xy)e)fgh)`有什么区别呢？

仔细比对一下，`(abc(d(xy)e)fgh)`按左右括号划分配对的话，它左右刚好能够成对数：`(abc (d (xy ) e) fgh)`(这里用一个空格分隔，从内向外互相成对)。但`ab(abc(d(xy)e)fgh)df`按左右括号划分配对的话，得到的是`ab( abc( d( xy )e )fgh )df`，显然，它中间多了一层无法成对的内容`xy`。

为了写出按照这种成对划分的递归表达式，先不考虑多出来无法成对的`xy`这一层。那么对应的递归正则表达式为：
```ruby
/[^()]* \( \g<0>* \) [^()]*/x
```

其中`[^()]*\(`是头部，`\)[^()]*`是尾部，中间用`\g<0>*`实现头尾成对的无限嵌套。

再来考虑中间多出来的无法成对的`xy`这部分。解决多余无法成对内容的更通用方法是使用二选一的分支结构，即`|`结合递归表达式一起使用，参见下一小节。

### 深入递归(4)：递归结合二选一分支

要处理上面多出的无法成对的数据，可以通过二选一结构`|`改写成如下更通用的方式：

```ruby
/[^()]* \( \g<0>* \) [^()]* |./x
```

进行匹配测试：
```ruby
reg = /[^()]* \( \g<0>* \) [^()]* |./x
reg =~ "ab(abc(d(xy)e)fgh)df"
#=> 0
```

当递归正则表达式结合了`|`提供的二选一分支功能时，`|`左边或右边(和`\g<>`相反的那一边)都可以用来提供这些『孤儿』数据。

例如，上面示例中，当递归进行到发现`xy`这部分是多余的时候将无法继续匹配，这时候将可以从二选一的另一个分支来匹配这个多余的数据。

但是这个二选一分支带来了一个新的问题：只要有无法匹配的，都可以去另一个分支匹配。假如右边的分支是个`.`，这就相当于多了一个万能箱，什么都可以从这里匹配。

但如果无法匹配的多余字符是右括号或左括号这个必须的字符呢？少了任何一个括号，都不再算是成对的嵌套结构，但却因为二选一分支而匹配成功。

如何解决这个问题？第一，需要保证另一分支不是万能的`.`；第二，需将整个结构做位置锚定。例如：
```ruby
/\A ( [^()]* \( \g<1>* \) [^()]* | [^()] ) \Z/x
```

注意，上面加了括号分组，所以`\g<0>`随之改变成`\g<1>`，因为递归的时候并不需要将锚定也包含进来。

当然，上面示例中二选一分支的另一个分支所使用的是单字符匹配`[^()]`，如果有多个连续的多余字符，这会导致多次选中该分支。为了减少匹配的测试次数，可以将其直接写成`[^()]*`。

```ruby
/\A ( [^()]* \( \g<1>* \) [^()]* | [^()]* ) \Z/x
```

<a name="lower_performence"></a>

但这有可能会在匹配失败的时候导致大量的回溯，从而性能暴降。例如，如下失败的匹配：
```ruby
reg = /\A([^()]* \( \g<1>* \) [^()]* | [^()]* )\Z/x

# 匹配失败性能暴降
(st=Time.now) ; (reg =~ "ab(abc(d(xy)e)fghdf") ; (Time.now - st)
#=> 1.7730072
(st=Time.now) ; (reg =~ "ab(abc(d(xy)e)fghdffds") ; (Time.now - st)
#=> 47.5858051

# 匹配成功则无影响
(st=Time.now) ; (reg =~ "ab(abc(d(xy)e)fgh)df") ; (Time.now - st)
#=> 5.9e-06
```

从结果发现，就这么短的字符串，第一个匹配失败竟需要花费1.8秒，第二个字符串更夸张，仅仅只是多了3个字符，耗费的时间飙升到47秒。

解决方法有很多种，这里提供两种：一种是将`*`号直接移到分组外，这虽然并不等价，但并不影响最终的匹配结果；另一种是将该多选分支使用固化分组或占有优先的模式。
```ruby
reg1 = /\A([^()]* \( \g<1>* \) [^()]* | [^()] )*\Z/x
reg2 = /\A([^()]* \( \g<1>* \) [^()]* | (?>[^()]*) )\Z/x

# 匹配成功
(st=Time.now) ; (reg1 =~ "ab(abc(d(xy)e)fgh)df") ; (Time.now - st)
#=> 6.1e-06
(st=Time.now) ; (reg2 =~ "ab(abc(d(xy)e)fgh)df") ; (Time.now - st)
#=> 5.8e-06

# 匹配失败
(st=Time.now) ; (reg1 =~ "ab(abc(d(xy)e)fghdf") ; (Time.now - st)
#=> 8.46e-05
(st=Time.now) ; (reg2 =~ "ab(abc(d(xy)e)fghdf") ; (Time.now - st)
#=> 0.0004223
```

### 深入递归(5)：小心递归中的分组捕获

在介绍示例之前，先验证一下结论。

在递归过程中，可能也会有分组捕获的表达式，所以，递归正则设置的相关变量值是最后一次分组捕获对应的状态。例如：
```ruby
reg = /(abc|def) and \g<0>?xyz/

# 只递归一轮
reg =~ "abc and def and xyzxyz"  #=> 0

# $~表示本次所匹配到的所有字符串
$~
#=> #<MatchData "abc and def and xyzxyz" 1:"def">

# $1表示第一个分组捕获所对应的内容
$1   #=> "def"
```

上面结果可以看出，在递归过程中，最后一轮的递归操作(此处示例即第一轮递归)设置了一些正则匹配时的变量，它会覆盖在它之前的递归设置的结果。

再来看一个示例。现在有个需求：匹配任何长度的回文字符串(palindrome)，比如1234321、abcba、好不好、abccba、好、好好、123321，该示例只能使用二选一的分支来实现。

这里简单分析一下，如何通过递归正则来实现该需求。

假设要匹配的这个字符串是`abcdcba`，先把多余的字符d去掉，那么要匹配的是`abccba`，这也是我们想要匹配的一种字符串模式。首先，左右配对的部分必须是完全一致的数据，这个递归正则其实很容易实现，用占位符来描述，大概模式为：`(.)_*\1`。将其替换成递归正则表达式：
```ruby
/(.) \g<0>* \1/x
```

再来考虑多余的那个字符，直接将其放在二选一分支的另一分支即可：因为二选一分支，所以这里的`\g<0>`就可以不用量词修饰来保证递归的终点
```ruby
/(.) \g<0> \1 |./x
```

最后，加上位置锚定。
```ruby
/\A ( (.) \g<1> \2|.) \Z/x
```
似乎已经没问题了，去测试匹配下：
```ruby
/\A ( (.) \g<1> \2|.) \Z/x =~ "abcba"
#=> nil
```

结果却并不如想象中那样成功。

不过，这个正则表达式的逻辑确实是没有问题的。例如，使用`grep -P`(使用PCRE)执行等价的正则去匹配回文字符串。
```bash
$ grep -P "^((.)(?1)\2|.)$" <<<"abcdcba"
abcdcba

# 下面的则失败
$ grep -P "^((.)(?1)\2|.)$" <<<"abcdcbad"
```

但是这个"正确的"正则表达式在Ruby中却无法达到目标。这是因为Ruby中的递归也会设置分组捕获，每个`\2`所反向引用的就不再是每轮递归中同层次的分组捕获`(.)`的内容了，而是真正的从左向右的第二个分组捕获括号所捕获的内容。

好在，Ruby提供了更加灵活的分组捕获的引用控制。除了`\N`这种方式的反向引用，也可以通过`\k<N>`或`\k<name>`来引用，灵活之处在于`\k<>`支持递归层次的偏移，例如`\k<name+0>`表示取当前递归层次里的name分组捕获，`\k<name+1>`和`\k<name-1>`分别表示取当前递归层的下一层和上一层里的name分组捕获。

所以，在Ruby中改一下这个正则表达式就能正常工作：
```ruby
/\A ( (.) \g<1> \k<2+0>|.) \Z/x =~ "abcba"
#=> 0
/\A ( (.) \g<1> \k<2+0>|.) \Z/x =~ "abcbaa"
#=> nil
```

当然，用命名捕获也是可以的：
```ruby
/\A (?<i> (?<j>.) \g<i> \k<j+0>|.) \Z/x
```

最后，可以将上面的正则表达式改动一番。上面正则中，多选分支的`.`一直都是放在尾部的(放头部也没问题)，但下面这种将多选分支和递归表达式嵌在一个分组内也是很常见的用法。下面这两种递归正则表达式是等价的。
```ruby
/\A (?<i> (?<j>.) \g<i>       \k<j+0>|.) \Z/x
/\A (?<i> (?<j>.) (?:\g<i>|.) \k<j+0> ) \Z/x
```

`(?:\g<i>|.)`进行了只分组不捕获，分组将它们两绑定在一个组内，如果不分组将会出错，因为`|`的优先级太低。


### 不要滥用递归正则

虽然递归正则确实能解决一些特殊需求，但是能不用尽量不用，因为递归正则要配合量词来修饰递归表达式，这本身不是问题，但是递归表达式很多时候在分组内，而分组本身可能也会用量词去修饰，这样两个量词一结合，一不小心可能就出现大量的回溯，导致匹配效率疯狂下降。

[前文](#lower_performence)已经演示过一个这样的现象，仅仅只是多了3个字符，匹配失败竟然需要多花费40多秒，而且随着字符的增多，匹配失败所需时间飙升的更快。这绝对是我们要去避免的。

所以，当写出来的递归正则表达式里又是分组、又是量词，看上去还『乱七八糟』的结合在一起，很可能会出现性能不佳的问题。这时候可能需要去调试优化，以便写出高性能的递归正则，但这可能会耗去大量的时间。

所以，尽量想其它方法来解决递归正则想要实现的匹配需求，或者只写看上去就很简单的递归正则。

## 其它一些有用的方法

---------

**1.escape或quote：它们是别名，用于将字符串中包含正则元字符的符号进行转义，返回安全的用于构建正则的源字符串**

```ruby
Regexp.escape("^hello\s(w.rld)$")
#=> "\\^hello\\ \\(w\\.rld\\)\\$"

Regexp.new(Regexp.escape("^hello\s(w.rld)$"))
#=> /\^hello\ \(w\.rld\)\$/
```

从结果中不难发现，其实是把元字符转义了，使得它们不再具有元字符的特殊意义。因此，`Regexp.new(Regexp.escape(str)) =~ str`总是能够完全匹配。

有时候转义是非常需要的功能，比如在正则源字符串中使用变量内插，但变量包含了元字符，比如一个点`.`，直接内插的话它就表示匹配任意单个字符，而期待的效果可能是仅只是匹配点字符，这时候就需要对这个特殊符号进行转义。
```ruby
var="hello.world"

Regexp.new(Regexp.escape(var))
#=> /hello\.world/

/#{Regexp.escape(var)}/
#=> /hello\.world/
```

----------

**2.inspect、to_s和source：将正则表达式转换成字符串格式**

```ruby
/hello.*$/i.to_s
#=> "(?i-mx:hello.*$)"

/hello.*$/i.inspect
#=> "/hello.*$/i"

/hello.*$/i.source
#=> "hello.*$"
```

从结果中可以看出，source转换时会完全移除修饰符，inspect转换的结果更加人性化，to_s总是以`(?on-off)`的方式转换。

**3.names和named_captures：获取正则表达式中的命名分组捕获相关的信息**

named_captures返回一个hash结构，将分组名与其索引号作为key/value对应保存起来。注意，不会设置非命名分组的信息，事实上Ruby中只要正则中有命名分组，普通分组就变成只分组不捕获的行为了，所以无法再引用该分组。
```ruby
/(?<foo>.)(?<bar>.)/.named_captures
#=> {"foo"=>[1], "bar"=>[2]}

/(?<foo>.)(?<foo>.)/.named_captures
#=> {"foo"=>[1, 2]}

/(?<h>hel.*) (world)/.named_captures
#=> {"h"=>[1]}
```

names返回一个数组结构，包含所有命名分组的分组名。
```ruby
/(?<foo>.)(?<bar>.)(?<baz>.)/.names
#=> ["foo", "bar", "baz"]

/(?<foo>.)(?<foo>.)/.names
#=> ["foo"]

/(.)(.)/.names
#=> []
```

**4.正则表达式的比较**

一般不会直接去比较两个正则表达式，但Regexp确实也提供了`==`和`eql?`这两个方法。

**5.union：可根据变量推导正则和二选一结构的正则**  

```ruby
Regexp.union("penzance")             #=> /penzance/
Regexp.union("a+b*c")                #=> /a\+b\*c/
Regexp.union("skiing", "sledding")   #=> /skiing|sledding/
Regexp.union(["skiing", "sledding"]) #=> /skiing|sledding/
Regexp.union(/dogs/, /cats/i)        #=> /(?-mix:dogs)|(?i-mx:cats)/

words = %w[cat dog fox]
Regexp.union(words)   #=> /cat|dog|fox/
```

