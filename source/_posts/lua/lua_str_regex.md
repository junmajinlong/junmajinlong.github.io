---
title: Lua字符串和Lua正则
p: lua/lua_str_regex.md
date: 2020-05-02 10:40:39
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# 字符串

在Lua中，字符串是不可变的，所以相同内容的字符串在lua进程的内存中只有一份。

```lua
a,b,c = "hello","hello","hello"   -- (a,b,c) --> "hello"
```

因为字符串不可变，所以任何带有修改性质的字符串操作，都会创建一个新的字符串。

# 字符串字面量

字符串可使用单引号、双引号包围，它们在效果上等价。

```lua
"a\tb"      --> a       b
'a\tb'      --> a       b
```

支持`\ddd`格式的字符。例如：

```lua
"a\039b"   --> a'b
"a\034b"   --> a"b
```

还可以使用`[[]]`包围字符串，类似于here doc，它允许换行，允许出现单双引号。

```lua
a = [[
  hello"world
  HELLO'WORLD ]]

a
--[[
  hello"world
  HELLO'WORLD
--]]
```

但是不能出现`]]`，为了允许长字符串里面也能出现`]]`，可加上一个或多个`=`号，结尾处也对应加上同样数量的等号。

```lua
a = [=[
  hello]]world
  HELLO'WORLD ]=]
```

但这时又不能出现`]=]`，为了允许出现这个符号，多加几个等号：

```lua
a = [==[
  hello]=]world
  HELLO]]WORLD ]==]
```

# 字符串基本操作


使用`..`运算符可以连接两个字符串：

```lua
a="hello "
b="world"
a..b  --> "hello world"  -->创建了新的字符串hello world
```

使用长度运算符`#`可以计算字符串的长度(也用于计算数组长度)，它返回的是字节数量，而不是字符数量：

```lua
a="hello"
b="我是"

#a      --> 5
#b      --> 6
```

如果要返回字符数量，可使用utf8标准库的len函数，例如`utf8.len("我是")`返回2。

使用`tostring(arg1)`会尝试将arg1转换成字符串类型。

```lua
tostring(3.3)    --> 3.3
tostring("3.3")  --> 3.3
```

# 字符串标准库string

最常用的几个函数：

```
string.byte(), string.char()
string.sub()
string.find(), string.match()
string.gsub(), string.gmatch()
string.format()
```

上面这些函数也可以写成面向对象语法。比如`string.sub(s,1,-1)`的等价写法是`s:sub(1,-1)`。

## string库的基本操作

`string.lower()`和`string.upper()`改变字符串大小写。

`string.rep(str,N)`重复str字符串N次。

```lua
string.rep("a", 3)    --> aaa
```

`string.sub(str,i,j)`表示提取str中从索引i开始到索引j结束的子串，lua中索引都从1开始，而不是0，索引可以是负数索引，-1表示最后一个字符。字符串是不可变的，所以string.sub()返回的是新字符串，不会原处修改。

```lua
s = "junmajinlong"
s = string.sub(s, 1, 5)   --> 截取前5个字符
s = string.sub(s, 5, -1)  --> 截取从第5个字符开始的所有字符
s = string.sub(s, #s-5, -1)  --> 截取后5个字符
s = string.sub("[mysql]", 2, -2)  --> 截取去头去尾后的字符mysql
```

## char()和byte()

`string.char(n1,n2,n3,...)`接受一个或多个整数，然后将每个整数转换为对应的字符，最后返回这些字符组成的字符串。

```lua
string.char(65,66,67)   --> ABC
```

`string.byte(str,N)`返回str中第N个字符对应的整数值，如果不给N参数，则返回第一个字符的整数值。`string.byte(str,i,j)`返回str中从索引i到索引j之间所有字符的整数值。

```lua
string.byte("ABC")      --> 65
string.byte("ABC",2)    --> 66
string.byte("ABC",-1)   --> 67
string.byte("ABC",1,-1) --> 65 66 67  返回所有字符的整数值
-- 因lua小巧，限制了栈大小，最多只允许100W个返回值
-- 所以上面不支持返回1M字符串的整数值
```

# find()和match()

find()用于"正则"搜索字符串，它返回搜索到的第一个子串的起始和结束索引位置，搜索失败则返回nil。

lua为了保证语言的精巧，它的"正则"不是正则，是lua自己实现的模式搜索表达式(600行代码不到，POSIX正则4000多行)，功能上是一个比较精简的正则表达式。

例如，find()搜索子串：

```lua
string.find("jun ma jin long", " [a-z]+ ")  --> 4  7
string.find("junmajinlong", "[0-9]")   --> nil
```

find()有两个可选参数，第三个参数表示从哪个索引位置处开始搜索，第四个参数是布尔值，为true时表示精确搜索而非"正则"搜索。

```lua
s = "jun ma jin long"
string.find(s, " [a-z]+ ", 5)  --> 7   11
string.find(s, " [a-z]+ ",5,true) --> nil
string.find(s, "ma",5,true)    --> 5   6
```

## match()

match()和find()基本类似，但match()返回搜索到的第一个子串，搜索失败返回nil。如果有分组捕获，则返回所有分组捕获到的内容。

```lua
string.match("good 12 Job 34", "%d%d")  --> 12

s = "jun ma jin long"
a,b,c,d = string.match(s, "(%w+)%s+(%w+)%s+(%w+)%s+(%w+)")
a   --> jun
b   --> ma
c   --> jin
d   --> long
```

## gsub()

`gsub(str, lua_pattern, replacement)`全局替换匹配成功的字符串。还有可选的第四个参数，限制替换的次数。

gsub()返回替换后的字符串，同时返回替换的次数。如果匹配失败，则返回字符串本身，且第二个返回值为0。

```lua
s = "junmajinlong"
string.gsub(s, "n", "N")      --> juNmajiNloNg  3
string.gsub(s, "ong", "ing")  --> junmajinling  1
string.gsub(s, "eng", "ing")  --> junmajinlong  0
string.gsub(s, "n", "N",1)    --> juNmajinlong  1
```

gsub()的replacement中，可以使用`%0 %1 %2 %3`等这样的分组捕获的反向引用，其中`%0`引用的是整个匹配到的内容。

例如，移除字符串前缀、后缀空白：

```lua
s = "   hello world   "
string.gsub(s, "^%s*(.-)%s*$","%1")
```

gsub()的replacement部分，还可以是一个函数或者一个table：  

- 如果是table，则将匹配到的字符串作为key，并以table中该key对应的value做替换  
- 如果是函数，则将匹配到的字符串作为参数传递给函数，并以函数返回值做替换  
- 如果table中该key不存在(或其值为nil)，或函数的返回值为nil，则不进行替换  

例如，将匹配到的内容转换为大小：

```lua
s = "hello"
string.gsub(s,".",function(x) return string.upper(x) end)
```

## gmatch()

`string.gmatch(s,pattern)`返回一个可迭代的函数，每次迭代的内容是匹配到的内容。

例如，找出所有大写字母：

```lua
s = "Perl Python Shell Ruby Lua"
for v in string.gmatch(s, "[A-Z]") do print(v) end
P
P
S
R
L
```

# UTF8标准库

Lua 5.3引入了utf8标准库，可以处理Unicode字符。

它支持如下几个函数：

```
utf8.len
utf8.char
utf8.codes
utf8.codepoint
utf8.offset
```

`utf8.len()`返回字符数量：

```lua
utf8.len("acb")    --> 3
utf8.len("我是")   --> 2
```

utf8.char()类似于string.char()，utf8.codepoint()类似于string.byte()，但注意，它们都使用字节索引，如果要使用字符索引，使用utf8.offset()，它返回指定字符索引位置处字符的起始字节索引值。

```lua
utf8.codepoint("我是")       --> 25105 返回第一个字符的整数
utf8.codepoint("我")         --> 25105
utf8.codepoint("我是",1,-1)  --> 25105  26159
utf8.codepoint("我是",2)  --> 报错，索引第2个字节而非第2个字符
utf8.codepoint("我是",3)  --> 报错，索引第三个字节
utf8.codepoint("我是",4)  --> 26159，索引第四个字节，即第二个字符

utf8.char(25105, 26159)   --> "我是"

s = "我是中国人"
utf8.offset(s,-1)  --> 13 最后一个字符的其实字节索引
utf8.codepoint(s, utf8.offset(s,2))  --> 26159
utf8.codepoint(s, utf8.offset(s,3))  --> 20013
utf8.codepoint(s, utf8.offset(s,-1)) --> 20154
```

utf8.codes()用于迭代utf8字符串中的每个codepoint值。

```lua
s = "我是中国人"
for i,c in utf8.codes(s) do
  print(i, c)
end
1       25105
4       26159
7       20013
10      22269
13      20154
```

# Lua"正则"

Lua使用百分号代替反斜线作为转义符号。例如`%.`匹配字符点。

支持如下元字符：

```
%    转义符号
.    匹配任意单个字符
?    匹配0或1次
*    匹配0或多次
-    非贪婪匹配0或多次
+    贪婪匹配1或多次
[]   字符类
()   分组捕获，捕获的分组内容可用`%1 %2`等进行反向引用
^    锚定开头，在中括号字符类中开头时，表示取反
$    锚定结尾
```

支持如下一些**字符类**：

```
%l     匹配小写字母
%u     匹配大写字母
%a     匹配大、小写字母
%d     匹配数字
%x     匹配十六进制数字，即`[a-fA-F0-9]`
%w     匹配字母和数字
%c     匹配控制字符
%s     匹配空白符号
%p     匹配标点
%g     匹配除空格外的可打印字符，即`[:graph:]`
```

它们全都有**大写方式表示对应的补集**，例如`%L`表示匹配除小写字母外的任意字符：

```lua
string.gsub("jun Ma 123","%L","x")  --> junxxaxxxx   6
```

此外，Lua还支持两种特殊的模式：  

```
%bxy      匹配从x开始到y结束的子串，如%b()匹配括号和括号内的东西
%f[set]   环视锚定位置，且左右同时环视，
          要求左边的字符不在set中，右边的字符在set中，
          即逆向否定环视且顺序环视，类似于/(?<![set])xxx(?=[set])/
          由于Lua支持所有的大写补集(如%L)，所以这也可以实现2种环视
            1. %f[%d]表示锚定左边不是数字，但右边是数字的位置
            2. %f[%D]表示锚定左边是数字，但右边不是数字的位置
```

例如，对于`%bxy`来说：

```lua
s = "a (b (c and d) e) f"
string.gsub(s, "%b()", "")  --> a  f    1

string.gsub("hello worlod", "%boo", "")
--> hellrlod        1
```

对于`%f[set]`来说：

```lua
string.gsub("ab12cd34ef", "%f[%d]", "_")  
--> ab_12cd_34ef    2

string.gsub("ab12cd34ef", "%f[%d]%w", "_")    
--> ab_2cd_4ef      2
```