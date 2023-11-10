---
title: Ruby字符串类型
p: ruby/ruby_string.md
date: 2020-05-11 09:37:33
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby字符串类型

字符串由String类提供，除了直接使用单双引号或其它字面量创建字符串，也可以使用String.new()方法来创建。

```ruby
a = "hello"
b = String.new("world")
```

Ruby中的**字符串是可变对象**。

## 字符串的连接

直接连接即可，空格会被忽略：

```ruby
>> "a""b"
=> "ab"

>> "a" "b"
=> "ab"

>> "a"    "b"
=> "ab"
```

也可以使用`+`号进行串联：

```ruby
"aaa"+"bbb"
=> "aaabbb"
```

## 单双引号

这和Perl一样，和Shell也类似。单引号是强引用，双引号是弱引用。

![](/img/ruby/1589465205967.png)

## 格式化字符串内插

Ruby显然也是支持printf、sprintf的，printf是直接输出格式化后的字符串，sprintf是返回(而不是输出，比如赋值给变量)格式化后的字符串。

此外，Ruby除了表达式或变量内插，还支持`%`的格式化字符串内插，等价于sprintf的功能。

```ruby
sprintf "pi is about %.4f",Math::PI
=> "pi is about 3.1416"

"pi is about %.4f" % Math::PI # 单个格式化字符串
=> "pi is about 3.1416"

"%s: %f" % ["pi", Math::PI]  # 多个格式化字符串
=> "pi: 3.141593"

"xiaomage = %{age}" % {:age => "23"}
=> "xiaomage = 23"
```

正如上面的示例，需要进行格式化的字符使用`%`标识，并使用`%`连接字符串和待替换的值。如果要内插多个字符串，则值部分使用中括号包围(即放进数组)或放进hash。

## %q和%Q和%

这和Perl里的`q() qq()`是一样的，也是分别充当单引号、双引号的角色。`%q()`被解析成单引号，单个`%`或`%Q`被解析成双引号。

`%q %Q`后面的`()`是引号的起始、终止定界符，定界符可以替换成其他成对或相同的符号。例如，下面是等价的：

```ruby
# 以下等价，内部的单引号不需要反斜线转义
%q(hello'world)
%q[hello'world]
%q{hello'world}
%q!hello'world!
%q#hello'world#

# 以下等价
%Q(hello'world)
%Q[hello'world]
%{hello'world}    # 单个%是一样的
%!hello'world!
%#hello'world#
```

如果使用的是成对的定界符，那么在定界符内只要合理的配对，就可以包含定界符字面符号。例如：

```ruby
%Q(hello(hello world)world)
%<<book>Ruby</book>>
%((1+(2*3)) = #{(1+(2*3))})
%(A mismatched paren \( must be escaped) # 不配对的括号需要转义
```

## 单字符

使用**一个问号**作为下一个字符的前缀，这个字符将成为字符字面量。例如：

```ruby
?A   # 代表一个大写字母A
??   # 代表一个问号
?"   # 代表一个双引号
?ab  # 错误，?只能修饰单个字符
```

在Ruby1.8及之前的版本，这是一个单字符序列，会转换成ASCII码存放，在Ruby 1.9之后，**单字符等价于只有一个字符的字符串**。

```ruby
>> ?A == 'A'
=> true
```

## Ruby字符串的可变性

对于Ruby来说，字符串是可变的。所以，无法使用单个对象来引用内容相同的两个字符串，如果能引用的话，其中一个修改了就表示另一个字符串也会修改，但这已经表示同一个对象了。

所以，只要Ruby遇到一个字符串字面量，都会新创建一个字符串对象。这意味着，如果在一个循环中使用了字符串常量，那么**这个常量字符串对象也会在每次循环过程中新创建**，而这是可以避免的消耗性能的一种方式：**尽量不要在循环内使用字符串常量**。

```ruby
>> 10.times {puts "test".object_id}
12046480
12046340
12046280
12046220
12046160
12046080
12046000
12045880
12045780
12045680
```

(另注：Python中字符串是不可变的，所以`for i in range(10):print(id("test"))`会返回10次相同的结果)


## +-可变和不可变(frozen)字符串

虽然字符串默认是可变的，但是也可以改变字符串的可变性。

```ruby
+str → str (mutable)
-str → str (frozen)
freeze()
```

`+str`表示返回一个可变的字符串对象：  
- 如果原始字符串是frozen的，则拷贝一份并返回它的可变对象  
- 如果原始字符串本身就是可变的(字符串默认就是可变的)，则返回自身，不拷贝字符串对象  

`-str`表示返回一个不可变(frozen)的字符串对象：  
- 如果原始字符串是可变的，则拷贝一份  
- 如果原始字符串本身不可变，则返回自身  

`freeze()`也表示返回不可变字符串对象，它总是在原处修改的。

所以，`[+ -]str`可能会创建新对象，而`freeze`则总是使得原始字符串不可变。

```ruby
>> a="world"     #=> "world"
>> a.object_id   #=> 15976580

>> b = +a        #=> "world"
>> b.object_id   #=> 15976580  # a本身可变，所以返回自身

>> a="world"     #=> "world"
>> a.object_id   #=> 8911880

>> b=-a          #=> "world"
>> b.object_id   #=> 8897840   # a可变，所以拷贝返回新的不可变对象b
>> b[1]="OO"     # b不可变，RuntimeError: can't modify frozen String

>> a[1]="OO"     #=> "OO"   # a仍然可变
>> a             #=> "wOOrld"

>> b.object_id   #=> 8854280
>> c = -b        #=> "world"  # b不可变，-b返回自身
>> c.object_id   #=> 8854280

>> d = +b        #=> "world"  # b不可变，所以+b创建新对象
>> d.object_id   #=> 11837980

>> x="hello"     #=> "hello"
>> x.object_id   #=> 11676080

>> y=x.freeze    #=> "hello"  # x和y是同一对象，都不可变
>> y.object_id   #=> 11676080
?> x[1]="E"      # RuntimeError: can't modify frozen String
>> y[1]="E"      # RuntimeError: can't modify frozen String
```

## 扩充字符串

有几种方式：
- （1）两字符串或多字符串直接相连`"abc""def"`或空格相连`"abc"  "def"`
- （2）使用加符号`+`连接两个或多个字符串`"abc"+"def"`
- （3）使用`*`号重复一个字符串n次`"ab" * 3`
- （4）使用`<<`将一或多个字符串(能通过to_str转换成字符串的对象均可)追加在原字符串尾部`"a" <<"b" <<"b"`
- （5）使用`concat()`将一或多个字符串追加在原字符串尾部`"a".concat("b","c")`
- （6）使用`prepend()`将一或多个字符串插入在原字符串的前面`"c".prepend("a","b")`

其中(1)到(3)的扩充方式都会返回新的字符串对象，而(4)到(6)的方式都是直接在原处修改字符串对象的。


注意，(1)和(2)扩充方式不会自动将其它类型转换成字符串类型，需要手动调用to_s方法来转换。

```ruby
>> "A""B"      #=> "AB"
>> "A"+"B"     #=> "AB"
>> "A"+2.to_s  #=> "A2"

>> "A"2    # SyntaxError:
>> "A"+2   # TypeError
```

可使用`<<`将多个字符串**追加**到某个字符串的尾部，它同样不会自动转换成字符串。这时候字符串就像是一个字符数组一样，但需要知道，**Ruby字符串不是字符数组，只是实现了一些好用的操作字符串的方法**：

```ruby
>> "abc" << "hello" <<"world"
=> "abchelloworld"
```

`<<`可以直接追加整数，整数被当作ASCII或其它编码字符进行转换，使得追加到字符串里的是字符。

```ruby
>> "xyz" << 65
=> "xyzA"
```

只是需要注意的是，使用`+`或直接相连扩展字符串的方式时会**自动创建一个新的字符串对象**，原始字符串不变，也就是说在得到扩展的结果前拷贝了一些数据进行创建新字符串对象。而使用`<<`的方式，因为修改的是字符数组，所以是**原地修改的**。

```ruby
>> a="xyz"
#=> "xyz"

>> a + "XYZ"
#=> "xyzXYZ"
>> a      # a没有变
#=> "xyz"

>> a << "XYZ"
#=> "xyzXYZ"
>> a          # a已经变了
#=> "xyzXYZ"
```

`*`号重复字符串N次。于是，可以简单地写出等长的分割线。例如：

```ruby
>> "ab" * 3
=> "ababab"

>> '-' * 40
=> "----------------------------------------"
```

## Ruby字符串的索引属性

字符串可变、可索引子串、设置子串、插入子串、删除子串等等。

### 字符串\[\]搜索和赋值

通过`[]`可以对字符串进行搜索和赋值，**赋值时是原处修改字符串的**。索引方式有多种，且支持负数索引号。

```ruby
# 1.根据索引，搜索或赋值单元素
str[index] → new_str or nil
str[index] = new_str

# 2.根据索引和给定长度，搜索或赋值0或多个元素
str[start, length] → new_str or nil
str[start, length] = new_str

# 3.根据索引范围，搜索或赋值0或多个元素
str[range] → new_str or nil
str[range] = aString

# 4.根据正则模式(斜线包围正则表达式)，搜索或赋值匹配到的元素
# 返回或赋值最先匹配到的那个或那段字符串
str[regexp] → new_str or nil
str[regexp] = new_str

# 5.根据正则模式(包含分组匹配)，返回给定分组内容
# capture可以是分组名，也可以是分组索引号(即反向引用)
# 分组索引号为0表示regexp匹配的所有内容
# 如果是赋值操作，则替换给定分组的内容
str[regexp, capture] → new_str or nil
str[regexp, integer] = new_str
str[regexp, name] = new_str

# 6.根据给定字符串精确搜索或赋值
str[match_str] → new_str or nil
str[other_str] = new_str
```

可以说，Ruby对字符串的索引操作支持的是相当的丰富、完善。下面是一些例子：

```ruby
a = "hello there"

a[1]            #=> "e"
a[2, 3]         #=> "llo"
a[2..3]         #=> "ll"

a[-3, 2]        #=> "er"
a[7..-2]        #=> "her"
a[-4..-2]       #=> "her"
a[-2..-4]       #=> ""

a[11]           #=> nil，已越界
a[11, 0]        #=> ""
a[12, 0]        #=> nil
a[12..-1]       #=> nil

a[/[aeiou](.)\1/]      #=> "ell"
a[/[aeiou](.)\1/, 0]   #=> "ell" 等价于上面方式
a[/[aeiou](.)\1/, 1]   #=> "l"   第一个分组内容
a[/[aeiou](.)\1/, 2]   #=> nil   第二个分组

a[/(?<vowel>[aeiou])(?<non_vowel>[^aeiou])/, "non_vowel"] #=> "l"
a[/(?<vowel>[aeiou])(?<non_vowel>[^aeiou])/, "vowel"]     #=> "e"

a["lo"]         #=> "lo"
a["bye"]        #=> nil

s = "hello"
while(s["l"])   # 将所有的l替换成L
    s["l"] = "L"
end

while(s[/l/])   # 将所有的l替换成L
    s[/l/] = "L"
end
```

与索引搜索、索引赋值(包括替换、插入、删除元素)、索引检查元素是否存在等操作有一些相对应的String方法。如下。

### include?()

字符串中是否存在某子串。

```ruby
include? other_str → true or false
```

它和`STR["str"]`在true/false的布尔判断上是等价的。

例如：

```ruby
"hello".include? "lo"   #=> true
"hello".include? "ol"   #=> false
"hello".include? ?h     #=> true，是否包含单个字符h

"hello"["a"]  #=> nil，也表示false
"hello"["e"]  #=> e，也表示true
```

### index()

搜索某子串或匹配的子串的索引位置。

```ruby
index(substr[, offset]) → int or nil
index(regexp[, offset]) → int or nil
```

offset指定从哪个索引位置处开始搜索。

例如：

```ruby
"hello".index('e')             #=> 1
"hello".index('lo')            #=> 3
"hello".index('a')             #=> nil
"hello".index(?e)              #=> 1
"hello".index(/[aeiou]/, -3)   #=> 4
```

### replace()

替换字符串为另一个字符串，它会替换掉全部内容并返回替换后的新字符串。**它会原处修改字符串**。

```ruby
replace(other_str) → str
```

```ruby
s = "hello"         #=> "hello"
s.replace "world"   #=> "world"
s                   #=> "world"

# 字符串字面量也可以替换并返回新字符串值
"hello".replace("world") #=> "world"
```

### insert()

在字符串中某个位置处开始插入另一个字符串并返回插入后的字符串。**它会原处修改字符串**。
```ruby
insert(index, other_str) → str
```

例如：
```ruby
"abcd".insert(0, 'X')    #=> "Xabcd"
"abcd".insert(3, 'X')    #=> "abcXd"
"abcd".insert(4, 'X')    #=> "abcdX"
"abcd".insert(-3, 'X')   #=> "abXcd"
"abcd".insert(-1, 'X')   #=> "abcdX"
```

<a name="blogeachxxx"></a>

## 字符串的迭代

因为字符串支持索引操作，所以可以直接通过索引的方式进行迭代。

```ruby
a="hello"

i=0
while(i<a.length) do
  puts a[i]
  i += 1
end

0.upto(a.size - 1 ) do |x|
  print a[x]
end
```

可以将字符串split成数组，然后通过each迭代。注意，Ruby 1.9里字符串不能直接通过each迭代了，它不再max-in Enumerable中的each。

```ruby
a="hello"

a.split("").each do |x|
  puts x
end

a.split("").each_with_index do |char,idx|
  puts "#{idx}: #{char}"
end
```

Ruby的字符串类中还定义了4个迭代方式：`each_byte, each_char, each_line, each_codepoint`。最常用的，显然是`each_char`。

```ruby
a = "hello"

a.each_char do |x|
  puts x
end
```

`each_byte`、`each_codepoint`迭代时，得到的是各字符的ASCII码或Unicode代码点。

```ruby
"我".each_byte.to_a  #=> [230, 136, 145]

"我是单身狗".each_byte.to_a
    #=> [230, 136, 145, 230, 152, 175, 229, 141,
    #    149, 232, 186, 171, 231, 139, 151]

"我".each_codepoint.to_a  #=> [25105]

"我是单身狗".each_codepoint.to_a
    #=> [25105, 26159, 21333, 36523, 29399]
```

按块读取数据(或按段落读取)时，很可能会用上`each_line`来迭代缓冲中的一大段数据的每一行。

```ruby
each_line(separator=$/ [, getline_args]) {|substr| block } → str
```

每次迭代，都将该行传递到代码块中。

其中：

`separator`指定记录(行)分隔符，默认的是`\n`、`\r`或`\r\n`。

`getline_args`指定读取到每一行时的所作的操作，目前支持的是`:chomp`选项，表示将每行尾部的换行符移除掉。这是非常方便的功能，这样在迭代每一行的时候，不需要手动在代码块中使用chomp()了。

```ruby
str="a line\nb line\nc line\n"
str.each_line {|x| puts "#{x}"}
# 输出：
=begin
a line
b line
c line
=end
```

下面是使用`chomp`参数的功能。

```ruby
str="a line\nb line\nc line\n"
str.each_line(chomp: true) {|x| puts "#{x}: #{x.length}"}
# 输出：
=begin
a line: 6
b line: 6
c line: 6
=end

# 等价于
str.each_line {|x| x.chomp!();puts "#{x}: #{x.length}" }
# 输出：
=begin
a line: 6
b line: 6
c line: 6
=end
```

下面是指定行分隔符的用法。

```ruby
str="a line\nb line\nc line\n"
str.each_line("e") {|x| puts "-#{x}-"}
# 输出：
=begin
-a line-
-
b line-
-
c line-
-
-
=end

# 指定分隔符和chomp参数的时候
str.each_line("e", chomp:true) {|x| puts "-#{x}-"}
-a lin-
-
b lin-
-
c lin-
-
-
```

## Ruby字符串的比较

Ruby中典型的几种方法：`<=>、eql?、equal?、==、===`。

```ruby
string <=> other_string → -1, 0, +1, or nil
str == obj → true or false
str != obj → true or false
str === obj → true or false
eql?(other) → true or false
equal? → true or false
```

用`<=>`比较字符串大小时：
- 左边小于右边，则返回-1
- 左边等于右边(使用`==`做等值比较)，则返回0
- 左边大于右边，则返回1
- 两者不可比较(比如一方不是字符串)，则返回nil

有了`<=>`之后，如果再mix-in Comparable模块，那么就额外具备了`<、<=、> 、>=`和`between?`方法(买一送N)。

其中，对于纯字符串对象，`==、===、eql?`等价，都用于判断字符串内容是否相同。`==`是判断对象的内容是否完全一致(或者说是相等)；`===`是智能判断，对于字符串而言，等价于`==`；`eql?`是根据计算对象hash值进行比较的，对于字符串的hash()方法来说，它是根据字符串长度、内容、编码三个属性来生成hash值的。

`equal?`则最严格，只有双方引用的是同一对象时，才返回true。

说明下`eql?`比较。下面是`eql?()`在不同编码时的比较过程。

```ruby
>> "我".encode("utf-8").eql?( "我".encode("utf-16") )
=> false

>> "hello".encode("utf-8").eql?( "hello".encode("utf-16") )
=> false

>> "hello".encode("utf-8").eql?( "hello".encode("gbk") )
=> true
```

此外，String类还实现了`casecmp`和`casecmp?`。前者是无视大小写的`<=>`，后者是无视大小写的等值比较。编码不同或一方不是字符串时，返回nil。

```ruby
casecmp(other_str) → -1, 0, +1, or nil
casecmp?(other_str) → true, false, or nil
```

例如：

```ruby
"1234abc".casecmp("1234ABC")  #=> 0

"我".casecmp("我") #=> 0

"\u{c4 d6 dc}" #=> "ÄÖÜ"
"\u{e4 f6 fc}" #=> "äöü"

"\u{c4 d6 dc}".casecmp("\u{e4 f6 fc}")  #=> -1
"\u{c4 d6 dc}".casecmp?("\u{e4 f6 fc}") #=> true
```

## =~正则匹配字符串

```ruby
str =~ obj → integer or nil
```

- 如果obj是一个正则表达式，则用此正则去匹配str，匹配成功则返回匹配到的第一个字符的索引位置，否则返回nil
- 如果obj不是正则表达式，则调用`obj.=~(str)`，即调用obj的`=~`方法，然后以str作为参数

注：在Ruby中，**str =~ reg 和 reg =~ str是不等价的**，如果正则中包含了命名捕获时，只有后者才会设置分组变量，所以此时应该将reg放在符号`=~`的左边。为了统一代码规范，一直都如此书写比较好。而**在Perl中，reg的位置是在=~符号右边的**，这和Ruby正好相反。

```ruby
"hello" =~ /(?<x>e)/
#=> 1

puts x    # NameError: undefined `x'

/(?<x>e)/ =~ "hello"
#=> 1
```

## Ruby中字符串内存问题

- [Seeing double: how Ruby shares string values](http://patshaughnessy.net/2012/1/18/seeing-double-how-ruby-shares-string-values)
- [Never create Ruby strings longer than 23 characters](http://patshaughnessy.net/2012/1/4/never-create-ruby-strings-longer-than-23-characters)

