---
title: Ruby条件判断语句
p: ruby/ruby_condition.md
date: 2020-05-11 10:37:39
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby条件判断语句

## Ruby中条件判断说明

Ruby中支持if、unless、case三种条件判断语句，还支持三目条件运算符`expr ? stat1 : stat2 `，此外if/unless还可以作为修饰符来简化简单的条件判断语句。

条件判断涉及到true/false的判断，**在Ruby中，除了false和nil这两个结果，其它所有值都是true**。

另外，Ruby是表达式驱动而非语句驱动的语言，比如if/unless/case都是表达式，在Ruby中**它们有返回值：最后执行的那条语句的返回值就是表达式的返回值**。

例如：
```ruby
mark = 90
a = if mark > 80  # 将整个if结构赋值给a
  "good"
end

puts a      #=> 输出"good"
```

## Ruby逻辑运算符

![](/img/ruby/1589464194734.png)

例如：

```ruby
expression1 or expression2
```

此外，双叹号`!!`可将数据按照原true/false值转换成对应的布尔类型。比如：

```ruby
!!3     #=> true
!!0     #=> true
!!""    #=> true
!!true  #=> true
!!nil   #=> false
!!false #=> false
```

## if和unless条件判断

if和unless是相反的条件判断，它们语法完全一样。

```ruby
# 其中then可以省略
if CONDITION then
  STATEMENT
end

if CONDITION then
  STATEMENT1
else
  STATEMENT2
end

if CONDITION1 then
  STATEMENT1
elsif CONDITION2
  STATEMENT2
else
  STATEMENT3
end
```

其中if/unless的then关键字可以在存在换行符或分号的时候省略，例如下面是等价的：
```ruby
if CONDITION1
  STATEMENT1
end

if CONDITION1; STATEMENT1; end
if CONDITION1 then STATEMENT1; end
```

## if和unless作为修饰符

作为修饰符表示if和unless的条件判断语句放在将执行代码的后面。从原来的『如果怎样，就怎样』变成『我要这样，如果是这样的话』。

```ruby
a=0
p a if a.zero?

a=0
p a unless !a.zero?
```

当有多条语句时，可使用begin...end语句块：

```ruby
a=0
begin
  a+=1;p a
end if a.zero?
```

if和unless修饰符后面不能加else类语句。

注意，begin...end没有自己的作用域，它属于语句式代码块。

## if中赋值

if是一个语句式的语句块，Ruby中的语句式语句块不会创建新的作用域。

但是有一个比较有趣且某时候很便利的功能，在if中进行赋值会出现什么结果？这分两种情况：(1)在if条件语句中赋值；(2)在if语句体中赋值。

先考察下在if语句体中赋值的有趣现象。

### 在if语句体中赋值

不管if的判断是否成功，语句体中的赋值语句都会初始化对应的变量。

例如：
```ruby
if false
  zz=1
end

puts zz     # 输出nil
puts zzz    # 报错
```

上面的语句，`zz=1`这个赋值语句是不执行的，但是zz却已经被初始化了(初始化值为nil)，而从未设置过赋值语句的zzz则未初始化。

这是因为**局部变量的创建是在编译阶段分配的，在这个阶段它不管赋值语句出现在哪，只要扫描到了就初始化，而且它不管所赋的具体值，仅作初始化动作。等到程序的运行阶段再根据赋值语句做具体的赋值操作**。

### 在if条件语句中赋值

这是个比较智能且有趣的功能。

例如：

```ruby
if x = 1
  puts "x in if: #{x}"
end

puts "x out if: #{x}"
```
运行结果:
```ruby
a.rb:1: warning: found = in conditional, should be ==
x in if: 1
x out if: 1
```

它会给个警告，提示用户条件中可能是`==`而不是`=`，但最终结果仍然是成功的。

因为赋值也是有返回值的，其返回值为所赋的值，即1。所以上面的语句总是返回成功。

如果，赋值一个false或nil，那么将if语句的判断将失败，但是赋值操作却已经完成了：
```
if z=false
  ...code...
end

puts z   # fasle
```

上面的if语句体中的语句虽然不执行，但是z已经赋值为false，且在if语句之后能使用，因为它不会创建新的作用域。

Ruby对于条件式语句的处理比较智能，如果不是将字面量赋值给变量，就不会出现上面的警告信息。

例如：
```ruby
z = 1
if x = z
  puts "x in if: #{x}"
end

puts "x out if: #{x}"
```

一般来说，对于这种类型的代码，更习惯写成如下形式：
```ruby
z = 1
x = z
if x
  puts "x in if: #{x}"
end

puts "x out if: #{x}"
```

但有时候在if条件式中赋值也是比较妙的。比如，在if条件式中将正则匹配的结果赋值给变量，如果匹配成功，则变量就被赋值为MatchData对象，语句体就可以执行，如果匹配不成功，则赋值为nil，于是if语句体就不执行。

```ruby
name = "Gao Xiaofang"
if n = /ao/.match(name)
  p n
  puts n
else
  puts "match failed"
end
```
输出结果：
```
#<MatchData "ao">
ao
```

当然，这也等价于：
```ruby
name = "Gao Xiaofang"
n = /ao/.match(name)
if n
  ...
else
  ...
end
```


## 三目运算符

`?:`用来简化逻辑简单的if...else...end语句，此外三目运算是一个表达式，它可以直接返回值，于是通过三目运算符来按条件赋值变得非常轻松。

```
score=59
what = score > 60 ? "ok" : "not ok"
```

## case运算符

case实现分支判断，语句格式为：case ... when ... then ... else ... end，else分支是全都when分支都不匹配时将执行的默认分支，then可以省略。

```ruby
a = 5

case a
when 5 then
  puts "a is 5"
when 6
  puts "a is 6"
else
  puts "a is neither 5, nor 6"
end
```

可以直接将case作为赋值语句的一部分，因为case被执行分支的最后一条语句可以作为返回值。

```ruby
a = 5

answer = case a
  when 5
    "a is 5"
  when 6
    "a is 6"
  else
    "a is neither 5, nor 6"
  end

puts answer    #=> "a is 5"
```
上面的示例中，都是`case X when EXPR....`格式的，这时是将X的值和EXPR以`===`运算符做比较的。而`===`是智能比较符号，如果某个类没有重写这个方法(即继承Object的`===`)，那么它等价于`==`，如果重写了，则根据重写规则来判断。通常用于以下几种判断：
  - **Array和String没有实现`===`，所以这两个类中的`===`等价于`==`的行为**
  - **对于Range的行为，所定义的是某对象是否在某个Range范围内**
  - **对于Module的行为，所定义的是某对象是否是某模块的实例或后裔**
  - **对于Regexp的行为，所定义的是某对象是否能匹配给定模式**，等价于`=~`

所以，case语句的功能就更加智能了，例如下面的示例做的是Module测试，测试某对象是否是某模块的实例或子孙。

```ruby
arr = ["a", 1, nil]
item = arr[0]
case item
when String
  puts "item is a String"
when Numeric
  puts "item is a Numeric"
else
  puts "item is a something"
```

但是，**也可以省略case后的值，直接在when后面写条件表达式**，这样就更加灵活：

```ruby
a = 5

answer = case
  when a <= 5
    "a is 5"
  when a == 6
    "a is 6"
  else
    "a is neither 5, nor 6"
  end

puts answer   # "a is 5"
```

多个条件可通过逗号分隔，它们表示`or`的关系，当第一个条件为真时，将不会再继续向后匹配：
```ruby
a = 2
case
when a == 1, a == 2
  puts "a is one or two"
else
  puts "a is not 1 or 2"
end
```

`a == 1`的时候匹配失败，于是匹配`a == 2`，匹配成功，于是退出。
