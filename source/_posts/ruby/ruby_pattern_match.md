---
title: Ruby 2.7的模式匹配和in操作符
p: ruby/ruby_pattern_match.md
date: 2020-08-01 13:37:31
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby 2.7的模式匹配和in操作符

吼吼，Ruby 2.7支持模式匹配了。

## Ruby 2.7模式匹配简介

Ruby 2.7的模式匹配通过操作符`in`实现，匹配成功后可以直接绑定赋值，匹配失败时将报错(抛异常NoMatchingPatternError)。此外，除了以前就支持的case when的`===`匹配，现在还支持case in匹配。

所以，模式匹配有两种用法：
```ruby
# in单独使用
<value> in <pattern>

# case...in...
case <value>
in <pattern1>
  ...
in <pattern2>
  ...
else
  ...
end
```

注意，case...in和case...when不能混用，但case...in也具备case...when的`===`匹配功能。

目前模式匹配还是实验特性，如果代码中使用了模式匹配，将会给出警告，如果想要禁止警告信息，可：
```ruby
Warning[:experimental] = false
{a: 1, b: 2} in {a:}
```

或者以`ruby -W:no-experimental`方式运行ruby脚本。

## in单独使用

单独使用in操作符做模式匹配时，可做变量的解构赋值，而且模式不匹配时会报错。

### 值与值匹配

```ruby
# 下面都能匹配成功
0 in 0
[2, 3] in [2, 3]
{one: 1, two: 2} in {one: 1, two: 2}

# 下面的全都无法匹配，将报错
0 in 1
[2, 3] in [2, 33]
{one: 1, two: 2} in {one: 1, two: 22}   # vaule不匹配
{one: 1, two: 2} in {one: 1, twooo: 2}  # key不匹配
```

### 值与类型匹配

```ruby
# 下面均能匹配成功
0 in Integer
[2, 3] in Array
[2, 3] in [2, Integer]
{one: 1, two: 2} in {one: 1, two: Integer}

# 下面均报错
0 in String
[2, 3] in [2, String]
{one: 1, two: 2} in {one: 1, two: String} # value的类型不匹配
```

### in使用===做匹配

其实值与值、值与类型的匹配都是in使用`===`符号进行匹配的特列。

例如：
```ruby
0 in 0
0 in -1..1
0 in Integer
```

### 标量格式的匹配模式

例如：
```ruby
# 标量的模式匹配
0 in a
a     #=> 0

[2, 3] in b
b     #=> [2, 3]
```

### 数组格式的匹配模式

```ruby
# 数组的模式匹配
langs = %w[perl shell ruby]
langs in [x,y,z]
x #=> perl      
y #=> shell
z #=> ruby

[1, 2, 3, 4] in [first, second, *other]
first  #=> 1
second #=> 2
other  #=> [3, 4]

[0,[1,2,3]] in [a, [b, *c]]
a    #=> 0
b    #=> 1
c    #=> [2, 3]

[0,[1,2,3]] in [a, [1, b, 3]]
a    #=> 0
b    #=> 2
```

如果模式匹配失败，则报错：
```ruby
langs = %w[perl shell ruby]
langs in [a,b] # NoMatchingPatternError

[2,3] in [b]  # NoMatchingPatternError
```

数组格式的模式匹配是全匹配的，要求值中的每个元素都有对应变量去接收。

### hash格式的模式匹配

除了标量和数组的模式匹配，还支持hash的模式匹配，此时根据key进行匹配：

![](/img/ruby/1596810646747.png)

可见，hash格式的模式匹配是半匹配的，允许对部分key进行匹配。

可在pattern中使用`**KEY`匹配剩余的key：
```ruby
hs = {one: 1, two: 2, three: 3}
hs in {one:, **rest}
rest   #=> {two: 2, three: 3}
```

在做hash格式的模式匹配时，被匹配的value中可能会有剩余的key，如果要限制不允许有多余的key，可在pattern中使用特殊的值`**nil`，相当于强制hash格式的模式匹配根据pattern进行全匹配。

```ruby
hs = {one: 1, two: 2, three: 3}
hs in {one:, three:, two:}
hs in {one:, three:}           # 不报错
hs in {one:, three:, **nil}    # 报错
hs in {one:, three:, two:, **nil}  # 不报错
```

所以，`**nil`的含义是：除了pattern中指定的key外，被匹配的value中没有剩余的元素了。

### 匹配时检查数据类型

在pattern中可以限制匹配时某变量的数据类型：
```ruby
0 in Float => a  # NoMatchingPatternError
0 in Integer => a

langs = %w[perl shell ruby]
langs in [String => a, String => b, c]
langs in [Integer => a,*other] # NoMatchingPatternError

# hash格式时可使用key: Type方式
hs = {one: 1, two: 2, three: 3}
hs in {one: Integer => a, two: Integer => b, **rest}
a #=> 1
b #=> 2

## 注意，下面仅仅只是对value的类型进行了匹配，没有为value绑定变量
hs in {one: Integer, three: Integer}
```

### 丢弃不需要的值

在模式匹配中进行变量的绑定赋值时，可使用`_`丢弃不感兴趣的元素，它是一个元素占位符，但不对对应的元素做匹配操作。

```ruby
[0, [1, 2]] in [0, [1, _] => a]
a   #=> [1,2]

[2,3,4] in [a,b,_]   # a=2, b=3
```

此外，对于数组格式和hash格式的模式匹配，可分别使用`*`或`**`来表示丢弃剩余元素：
```ruby
[1,2,3] in [Integer, *]
{a: 1, b: 2, c: 3} in {a:, **}
```

## case...in模式匹配

case...in使用in进行模式匹配：
```ruby
case <value>
in <pattern1>
  ...
in <pattern2>
  ...
else
  ...
end
```

匹配成功后选择对应的分支执行。如果所有的分支都未匹配成功，此时，如果有else分支，则选择else分支，否则报错。所以，如非必要，强烈建议写else语句，而不是去捕获case...in无法匹配时抛出的异常。

因为case...in使用in做模式匹配，所以前面介绍的【单独使用in】的模式匹配方式也都适用于case...in的分支中。除此之外，case...in还支持一些额外的语法。

case...in的分支中，如果是数组格式、hash格式的匹配方式，可省略pattern中**最外层**的中括号或大括号。

```ruby
case [1, 2]
in Integer, Integer
  "matched"
else
  "not matched"
end
#=> "matched"

case {a: 1, b: 2, c: 3}
in a: Integer
  "matched"
else
  "not matched"
end
#=> "matched"

case {name: 'John', friends: [{name: 'Jane'}, {name: 'Rajesh'}]}
in name:, friends: [{name: first_friend}, *]
  "matched: #{first_friend}"
else
  "not matched"
end
#=> "matched: Jane"
```

pattern中可以使用二选一符号`|`来表示逻辑或的关系：只要匹配其中一个表达式，就算成功。
```ruby
case 0
in 0 | 1 | 2
  "0 or 1 or 2"
in Hash | Array
  "Hash or Array"
else
  "not matched"
end
```

但如果存在变量绑定赋值，则不能使用`|`进行多选一，这会报语法错误：
```ruby
case {a: 1, b: 2}
in {a: } | Array
  "matched: #{a}"
else
  "not matched"
end
# SyntaxError (illegal variable in alternative pattern (a))
```

case...in有时候会带来很大便利，特别是在处理hash结构的数据时(或类hash的结构，比如struct、json、object等)优势更大，熟悉函数式编程的都知道模式匹配的威力(尽管模式匹配和函数式编程没有必然的关联关系)。

例如：
```ruby
require 'json'

json = '{
  "name": "Alice",
  "age": 30,
  "children": [
    {
      "name": "Bob",
      "age": 2
    }
  ]
}'

# 筛选出只有一个小孩，且孩子名字为Bob，输出孩子的年龄
case JSON.parse(json, symbolize_names: true)
in {children: [{name: "Bob", age: Integer => age}]}
  puts age
end

# 不使用case...in
person = JSON.parse(json, symbolize_names: true)
children = person[:children]
if children.length == 1 and children[0][:name] == "Bob"
  puts children[0][:age] if children[0][:age].is_a? Integer
end
```

case...in在模式匹配时还支持if/unless条件判断，即使某分支的模式能够匹配成功，但如果条件判断失败，仍然不会选择该分支。条件判断中可使用模式匹配时绑定的本地变量。

例如，上面的case中加一个条件：父亲年龄大于等于30岁
```ruby
# 使用case...in
case JSON.parse(json, symbolize_names: true)
in {age: p_age, children: [{name: "Bob", age:}]} if p_age >= 30
  puts age
end

# 不使用case...in
person = JSON.parse(json, symbolize_names: true)
if person[:age] >= 30
  children = person[:children]
  if children.length == 1 and children[0][:name] == "Bob"
    puts children[0][:age] if children[0][:age].is_a? Integer
  end
end
```

由于pattern中会进行变量的绑定赋值，如果变量是已存在的变量，那么不会将其值替换在pattern中，而是进行变量赋值：
```ruby
a=0
case 1
in a    # 匹配该分支成功，a赋值为1
  puts "matched: #{a}"
else
  puts "not matched"
end
```

如果想要在pattern中使用已存在变量的值，使用`^VAR`表示法：
```ruby
a = 0
case 1 
in ^a       # 相当于in 0，所以该分支匹配失败
  puts "aaa"
end
```

`^VAR`的VAR也可以来自于当前pattern中已绑定的变量：
```ruby
jane = {school: 'high', schools: [{id: 1, level: 'middle'}, {id: 2, level: 'high'}]}
john = {school: 'high', schools: [{id: 1, level: 'middle'}]}

case jane
# select the last school, level should match
in school:, schools: [*, {id:, level: ^school}]
  "matched. school: #{id}"
else
  "not matched"
end
#=> "matched. school: 2"

case john 
# the specified school level is "high", but last school does not match
in school:, schools: [*, {id:, level: ^school}]
  "matched. school: #{id}"
else
  "not matched"
end
#=> "not matched"
```

## 自定义对象的模式匹配

如果对象具有deconstruct()方法返回数组，那么该对象可以进行数组格式的模式匹配；

如果对象具有deconstruct_keys(keys)方法返回hash，那么该对象可以进行Hash格式的模式匹配，其中参数keys是要匹配的key的数组。

例如：
```ruby
class Point
  def initialize(x, y)
    @x, @y = x, y
  end

  def deconstruct
    [@x, @y]
  end

  def deconstruct_keys(keys)
    {x: @x, y: @y}
  end
end

case Point.new(1, -2)
in px, Integer
  "matched: #{px}"
else
  "not matched"
end
"matched: 1"

case Point.new(1, -2)
in x: 0.. => px
  "matched: #{px}"
else
  "not matched"
end
#=> "matched: 1"
```

目前，Ruby已支持Struct对象进行数组格式和hash格式的模式匹配：
```ruby
Color = Struct.new(:r, :g, :b)

p Color[0, 10, 20].deconstruct #=> [0, 10, 20]
p Color[0, 10, 20].deconstruct_keys([:r, :g])  #=> {:r=>0, :g=>10}

Color[0,10,20] in [r,g,b]
Color[0,10,20] in {r:,g:,**}

case color
in Color[0, 0, 0]
  puts "Black"
in Color[255, 0, 0]
  puts "Red"
in Color[r, g ,b]
  puts "#{r}, #{g}, #{b}"
end
```

## 实验阶段需要注意的事项

目前模式匹配还是实验特性，虽然将来不会改变模式匹配的大方向，但有些行为细节可能在将来会发生变化。

下面这个特性是需要特别注意的，将来可能会发生改变：即使分支匹配失败，但仍然会绑定变量。

例如：
```ruby
case [1, 2]
in aa, String
  "matched 0"
in [xx, yy]
  "matched 1"
else
  "not matched"
end
```

很明显，上面会匹配第二个分支成功，所以`xx=1,yy=2`，但第一个分支已经匹配过，且其中的一部分匹配成功了，只是整个分支没有匹配成功，这里仍然会设置`aa=1`。

但如果第一个分支改成`in String => aa, String`，那么每一个元素都匹配失败，此时aa将会赋值为nil。

```ruby
case [1, 2]
in String => aa, String
  "matched 0"
in [xx, yy]
  "matched 1"
else
  "not matched"
end
```

因为匹配失败的分支可能会对变量进行赋值，所以写在下面的分支中可以使用写在上面的分支中绑定的变量：

```ruby
case [1, 2]
in aa, String
  "matched"
in [xx,yy] if aa == 1
  "matched 1"
else
  "not matched"
end
```

但这并非好事，因为使用它上面的分支中的变量时，并不确定那个分支能成功绑定所使用的变量：
```ruby
case [1, 2]
in {aa:}
  "matched 0"
in [xx,yy] if aa == 1   # 无法匹配
  "matched 1"
else
  "not matched"       # 选中该分支
end
```

另外，如果case...in之前已经存在变量a，那么case...in中分支绑定的变量a可能会覆盖已存在的变量a，即使该分支匹配失败。

```ruby
a = 5
case [1, 2]
# 整个分支无法匹配任何一个元素，不会覆盖已存在的变量a
in String => a, String
  "matched"
else
  "not matched"
end
#=> "not matched"
a      #=> 5

case [1, 2]
# 整个分支匹配失败，但a能匹配数组第一个元素，会覆盖已存在的变量a
in a, String
  "matched"
else
  "not matched"
end
#=> "not matched"
a      #=> 1
```
