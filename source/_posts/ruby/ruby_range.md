---
title: Ruby中的Range类型
p: ruby/ruby_range.md
date: 2020-05-14 09:14:36
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby Range类型

Range是一个序列范围。使用`s..e`或`s...e`或`Range#new()`来创建范围对象（一般使用rng来表示）。

它们适用于整数值(不能是float类型)、字母。事实上，只要某个类定义了`<=>`方法以及`succ()`方法，那么自动就能支持Range。

```ruby
# range字面量创建Range
(-1..-5).to_a        #=> []
(-5..-1).to_a        #=> [-5, -4, -3, -2, -1]
('a'..'e').to_a      #=> ["a", "b", "c", "d", "e"]
('a'...'e').to_a     #=> ["a", "b", "c", "d"]

# 构造方法创建Range
Range.new(1,5)       #=> [1, 2, 3, 4, 5]
Range.new(1,5,true)  #=> [1, 2, 3, 4]
```

使用两点`..`创建的是包含结束边界元素的，使用三点`...`创建的是不包含结束边界元素的。如果`Range.new()`方法的第三个参数设置为true(默认new方法的第三个参数`exclude_end=false`)，则创建的range也是不包含结束边界的。

可以使用`exclude_end?`方法来检测一个rng对象是否包含结束边界。

```ruby
(1..10).exclude_end?         #=> false
>> (1...10).exclude_end?     #=> true
```

另外一点需要注意，反序Range是合理但无值的，它不适合直接使用，但适用于索引取值，例如`"abcde"[2..-1]`。

## 无上界和无下界的Range

有一种特殊的Range对象，它们是没有结束边界的。换句话说，它的上界为无穷。

```
1..
1...
1..nil
1...nil
Range.new(1,nil)
Range.new(1,nil,true)
```

需要注意两点：

![](/img/ruby/1589465005749.png)

```ruby
(1..) == (1..)    #=> true
(1..) == (1...)   #=> false
```

无上界的范围有时候非常实用，特别是在某些不知道上边界的时候。例如，取数组第一个元素到最后一个元素，但是最后一个元素的index是不固定的，使用无上界的Range非常容易取得这区间的元素：

```ruby
[1,2,3,4,5][2..]  #=> [3, 4, 5]
```

在Ruby 2.7中开始支持无下界Range(之前的版本并不支持)：

```ruby
arr = [3,2,1,5,4,6]
arr[..3]     #=> [3,2,1,5]
arr[...3]    #=> [3, 2, 1]

CLASSES = {
	..59 => :fail,
	60..79 => :first,
  80..89 => :second,
	90.. => :good
}
```

## 指定步长

`rng.step(N)`方法或百分号`rng % N`可以设置一个Range对象rng的步长为N。

但是，step一般需要结合语句块来使用，否则的话返回的是一个Enumerator。而`%`的方式无法结合语句块，只能先返回它的Enumerator，再来手动迭代。

例如：
```ruby
r = ((1..10).step(2))  #=> ((1..10).step(2))
r.to_a                 #=> [1, 3, 5, 7, 9]

(1..10).step(2) {|x| puts x}
                       # 输出：1 3 5 7 9

# 下面的括号不能省略
# 省略括号表示：1..10 % 2，即：1..0
rr = (1..10) % 2    #=> ((1..10).%(2))
rr.each {|x| puts x}
```


## Range对象迭代

Range类自身只实现了一个each()方法。
```ruby
(1..10).each {|n| puts n}
```

但它mix-in了Enumerable模块，所以这个模块中的方法都能使用。

但是注意，Range对象如果起点是浮点数，那么是不可迭代的，因为中间的浮点数是无穷多。但终点是浮点数与否则无所谓，它会按整数方式迭代到终点截断小数位后的整数。

```ruby
(1..3.5).each {|x| puts x}    # 1 2 3
(1..3.0).each {|x| puts x}    # 1 2 3
(1.1..3.0).each {|x| puts x}  # TypeError
```

## Range的一些常用操作

- `size()`：返回Range对象中包含的元素个数
- `to_a()`或`entries()`：它们等价，返回Range对象各元素组成的数组
- `begin()`：返回Range对象的第一个元素
- `end()`：返回Range对象的最后定义元素
- `first()`：返回Range对象的第一个或前N个元素，用法见下文示例
- `last()`：返回Range对象的最后一个或N个元素，用法见下文示例

`begin()`和`end()`分别返回Range对象的首尾定义元素，不论它是否包含结束边界。例如：
```ruby
(1..10).end     #=> 10
(1...10).end    #=> 10
```

对于`first()`和`last()`，均可指定参数或不指定参数。不指定参数时分别等价于`begin()`和`end()`，指定参数N时，返回实际包含的前N个或最后N个值。

```ruby
(10..20).first     #=> 10
(10..20).first(3)  #=> [10, 11, 12]

10..20).last       #=> 20
(10...20).last     #=> 20
(10..20).last(3)   #=> [18, 19, 20]
(10...20).last(3)  #=> [17, 18, 19]
```

## Range对象相等性判断

`rng1 == rng2`和`rng1.eql? rng2`用于比较两个Range对象是否相等。

比较具有两方面：  

- 分别使用`==`或`eql?`来比较rng的头尾元素是否相等  
- rng的边界性质是否一样，包含边界和不包含边界是不一样的  

换句话说，只有头尾两元素相等且边界性质相同，才是相等的范围。


## Range成员判断

非常常用的一个成员判断方式是Range实现的`===`方法。

```ruby
(1..10) === 3         #=> true
(1..10) === 3.1       #=> true
(1..10) === (2..3)    #=> false

("a".."z") === "c"    #=> true
("a".."z") === "cc"   #=> false
```

由于case的等值判断语法中是使用`===`来判断的，所以对于range而言，可直接使用case语句判断一个对象是否在range范围内。
```ruby
case 79
when 1..50   then   print "low\n"
when 51..75  then   print "medium\n"
when 76..100 then   print "high\n"
end
```


还有`include?()`方法和`member?()`，它们等价，也是用来判断是否是Range对象的成员。
```ruby
(1..10).include?(3.1)      #=> true
("a".."z").include?("g")   #=> true
("a".."z").include?("A")   #=> false
("a".."z").include?("cc")  #=> false
```

此外，还有`cover?()`方法判断另一个Range是否是该Range的子集。只不过它只是比较首尾两个元素而已，所以有时候的结果可能出乎想象。

```ruby
(1..10).cover? 3         #=> true
(1..10).cover? 3.0       #=> true
(1..10).cover? (2..9)    #=> true
(1..10).cover? (2...11)  #=> true

("a".."z").cover? ("c")  #=> true
("a".."z").cover? ("cc") #=> true
```

注意上面最后一个测试结果，『cc』本不是`"a".."z"`的成员，但是却返回true，因为它在做首尾比较的时候，发现`"a"<="cc"`且`"cc"<="z"`均为真，于是就返回真。

## 最大和最小值

```
max → obj
max {|a,b| block } → obj
max(n) → obj
max(n) {|a,b| block } → obj

min → obj
min {|a,b| block } → obj
min(n) → obj
min(n) {|a,b| block } → obj
```

返回Range对象中最大、最小的值，如果给了参数n，则返回n个最大、最小值组成的数组。默认使用`<=>`比较各元素。

如果给了语句块，则使用语句块的逻辑来比较而不是默认的`a<=>b`，其中a和b是遍历时每次所取的元素。

如果首元素大于尾元素，或者在不包含边界时首尾元素相等，那么直接返回nil或空数组(如果给了n参数)。

```ruby
(1..10).max   #=> 10
(1...10).max  #=> 9

(1...1).max   #=> nil
(10..1).max   #=> nil

(1..10).max(3) {|a,b| a<=>b}
=> [10, 9, 8]
```

## 自定义支持Range的类

只要某类重写了`<=>`和`succ()`方法，就支持range操作。其中succ()用于每次获取下一个元素，`<=>`用于判断每次所获取到的元素是否到达了结束边界。

例如，在官方手册上定义了一个类Xs，该类中重写了succ和`<=>`方法，于是就支持range操作。先把该方法中的inspect注释掉。看看它构造出的范围是什么样的。

```ruby
# represent a string of 'x's
class Xs
  include Comparable
  attr :length
  def initialize(n)
    @length = n
  end
  def succ
    Xs.new(@length + 1)
  end
  def <=>(other)
    @length <=> other.length
  end
=begin
  def inspect
    'x' * @length
  end
=end
end
```

构造一个Xs对象组成的序列范围：
```
rng = Xs.new(3)..Xs.new(6)
=> #<Xs:0x00007fffcab78280 @length=3>..#<Xs:0x00007fffcab78208 @length=6>
```

这样的结果看上去显然是不好看的。于是定义下`inspect()`方法，使其人类可读。在这里to_s是用不上的，因为对于range来说，它只需要显示对象信息。to_s是在转换成字符串时使用的。

```ruby
r = Xs.new(3)..Xs.new(6)  #=> xxx..xxxxxx
r.to_a     #=> [xxx, xxxx, xxxxx, xxxxxx]
```


## Ruby中的flip-flop

flip-flop借鉴自Perl，flip-flop有些时候非常实用，特别是在处理文本数据时，它返回flip为真和flop为真中间范围内的数据。

在Ruby中flip-flop的语法`flip..flop`和`flip...flop`看起来像是Range，但只是运算符相同，它本质并不是Range，它支持`..`和`...`，但效果是等价的。

```shell
$ cat c.txt
a
b
c
d
a
b
c
d

$ ruby -ne 'puts $_ if ~/b/..~/c/' c.txt
b
c
b
c

$ ruby -ne 'puts $_ if ~/b/...~/c/' c.txt
b
c
b
c
```

Ruby之前的版本曾经想要废弃flip-flop语法，在Ruby 2.6版本使用flip-flop时也会给出警告信息，但因很多程序员反对废弃，Ruby 2.7又添加回来了。
