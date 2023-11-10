---
title: Ruby Enumerable模块详解
p: ruby/ruby_enumerable.md
date: 2020-05-14 13:57:32
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------


# Ruby Enumerable模块详解

Ruby中最出色的特性之一就是它的语句块，几乎贯穿在Ruby的方方面面。语句块在使用上颇为简单，但深究之下，却也并非浅显易懂，其所涉及的知识也是不少。

这里主要介绍和语句块有关的迭代器/枚举器相关的内容，打算分两部分写，本篇写Enumerable的一些方法和该模块的特性，另一篇写Enumerator枚举器：[Enumerator枚举器详细说明](/ruby/ruby_enumerator)。

## 使用Enumerable

![](/img/ruby/1589464390839.png)
```ruby
class C
  # mix-in Enumerable模块
  include Enumerable

  # 还需定义each方法
  def each
    ...CODE...
  end
end
```

这样，C类的对象就能使用自己定义的each方法，也能使用mix-in Enumerable之后『赠送』的一大堆迭代类的方法。当然，如果不需要使用Enumerable中的功能，只需定义each方法即可。

各个类中定义的each方法的作用是不一样的，比如Array类中定义的each方法是每次迭代一个元素，Hash类中定义的each方法是每次迭代一个键值对。

于我们而言，如果我们也想要在自己的类中定义each方法，那么也根据自己的需求去迭代。例如，下面定义的each方法是每次迭代一个1到100之间的整数值。

```ruby
class C
  include Enumerable

  def each
    i = 1
    while i<=100
      yield i
      i += 1
    end
  end
end
```

`yield`的用法在稍后会解释，这里将其理解为向之后的代码块中传递变量i的值即可。例如，第一次传递给代码块的值为1，第二轮迭代传递的值为2。

现在，这个类的对象就可以使用each方法，并且由于包含了Enumerable，所以也可以使用该模块中的方法，例如select筛选。

```ruby
o1 = C.new
o1.each {|x| puts "x: #{x}"}
p o1.select {|x| x > 90 }
```

## yield的基本用法

在使用代码块的时候，几乎总是需要在代码块内部写上代码块的变量，代码块中的变量用来保存每次迭代时取得的数据。

例如:
```ruby
a.each {|x| ...code...}
a.map {|x|  ...code...}
```

当然，这也并非必须要写，要看具体需求场景，但绝大多数代码块的使用都是使用变量的。
```ruby
# 不使用代码块变量
a.each { puts "haha" }

(1..3).map {"hello"}
#=> ["hello", "hello", "hello"]
```

例如，对于数组的each迭代而言，每次迭代时，从数组中取一个元素传递给代码块，并保存在代码块的变量中。

之所以能够对数组的所有元素进行迭代，且每次迭代都传递一个元素给代码块，得益于Array类each方法中使用的yield：每yield一次，表示迭代一次，yield将一个值的**引用**传递给代码块。

yield是一个Ruby关键字，就像def/class/while一样，它表示开始执行代码块。或者说，yield之后，将进行跳转，跳转到调用者的代码块上去执行，所以控制权暂时从yield处交出去了。当代码块执行完成，将重新跳回到yield的位置处，继续向下执行后续的代码，这时控制权又回到了yield上，或者说回到了迭代方法each上。

正如上面的示例：
```ruby
class C
  include Enumerable

  def each
    i = 1
    while i <= 100
      yield i
      i += 1
    end
  end
end

o1 = C.new
o1.each {|x| puts "x: #{x}"}
```

这里有两个部分需要关注，一是each方法的代码块，一是each方法的定义。

当each方法被调用后，将从main方法跳转到each方法(方法的调用需要跳转)，并且main方法停留在each方法的调用位置处，**只有each方法执行完毕**，main方法才从该位置继续向下执行其它代码。

重点关注each方法的执行过程：
1. 当each调用成功之后，将转到each方法的正文段，即each方法的代码定义段；
2. 在此方法中首先定义了一个局部变量i并赋值为1，然后进入while循环，while循环中的第一步就是`yield i`，这表示向each代码块`{|x| puts "x: #{x}"}`发送局部变量i的值(的引用)，即数值1。发送之后，将从yield语句的位置处立即跳出each方法，跳转到代码块中去执行；
3. 于是第一次迭代传递的数据保存在代码块变量x中，并且在代码块执行时被处理；
4. 当代码块执行完毕后，又跳转回each方法yield位置处，继续向下执行，即将局部变量i加1，然后进入下一个循环；
5. 下一个循环的第一步又是yield，但这次发送的变量i的值为2，继续从yield处跳出，执行代码块，代码块执行完又跳回each方法的yield位置处，继续向下执行代码；
6. 100次循环之后，while条件判断失败，不再进入循环体，也就不再yield，于是整个each方法退出，回到main方法调用each方法的位置处继续向下执行后续代码。

这就是整个迭代的过程。可见，在使用yield之后，可以让程序在代码块这个特殊的代码段和each方法之间来回跳转，直到不再yield，each方法才执行完毕。并且，yield可以在两个代码段之间传递数据。


理解了上面的过程后，再理解下面的each方法将不再有任何难度：
```ruby
class D
  def each
    yield 1, 11
    yield 2, 22
    yield 3, 33
  end
end

o2 = D.new
o2.each {|x, y| puts "x: #{x}, y: #{y}"}
```

上面的each只迭代3次，因为each方法中只有3次yield。且因为yield的参数是两个，所以代码块中也需要使用两个变量去保存yield传递来的两个数据。

更为通用的，yield可以用在任何位置处，并非只能在each中使用，而且可以不用指定传递给代码块的参数。如果yield没有传递参数，那么语句块中如果使用了语句块变量，语句块变量将赋值为nil。例如:
```ruby
class C
  def act
    yield
  end
end

obj = C.new
obj.act {puts "hello world"}
obj.act {|x| x.nil?}
#=> true
```

再比如，Kernel中的无限循环loop方法，它的定义就是通过yield并不传递任何值实现的，以下是loop的部分代码：
```ruby
def loop
  begin
    while true
      yield
    end
  rescue StopIteration => e
    e.result
  end
end
```

所以，在loop循环中需要使用break来终止循环，它会退出这里的while循环。

yield是结合代码块一起使用的，但是如果使用了yield却没有在方法上使用代码块，那么会报错`LocalJumpError: no block given`。可以通过`block_given?`方法来判断是否给定了语句块。

例如，下面实现一个自己的times的迭代方法:
```ruby
class Fixnum
  def my_times
    for i in 1..self
      yield i if block_given?
    end
  end
end

10.my_times
10.my_times {puts "hello world"}
10.my_times {|i| puts "hello world: #{i}"}
```

Ruby中能使用语句块，但不给语句块的时候，一般是返回一个Enumerator对象的。至于如何返回这个对象，以下是一个简单示例，该示例的具体分析参见对[Enumerator::new重构my_times](/ruby/ruby_enumerator#blogmy_times2)的介绍。

```ruby
class Fixnum
  def my_times
    if block_given?
      for i in 1..self
        yield i
      end
      return
    else
      e = Enumerator.new do |y|
        for i in 1..self
          y << i
        end
      end
      return e
    end
  end
end

puts 10.my_times
10.my_times.each { |i| puts "hello #{i}" }
```

此外，yield是有返回值的，它的返回值的详细说明，参见[yield返回值](/ruby/ruby_enumerator#blogfeed)。

yield的用法基本上就上面这些内容，它的用法很灵活，但也需要理解它的工作原理才能准确使用yield。

### yield细节：传递引用而不是值

注意，yield每次传递给语句块的是它的引用而不是直接传递值，所以效率相对较高，但正因为如此，在语句块中可以通过语句块变量去修改原始值。

例如：
```ruby
class C
  def f()
    a = "hello"
    puts a.object_id
    yield a
  end
end

c = C.new
c.f {|x| puts x.object_id}
```
输出结果：
```
41534500
41534500
```

基本上Ruby中的所有迭代方法都使用each实现，而each使用yield，所以Ruby中迭代元素时往往可以通过语句块变量去修改原始的元素，这一点需要特别注意。

```ruby
arr = %w(perl shell ruby python)

arr.each {|x| x[0]=x[0].upcase}
p arr
```

输出结果：
```
["Perl", "Shell", "Ruby", "Python"]
```

## Enumerable的一些迭代方法

Enumerable模块中定义了不少的迭代方法，它们都基于对象自己的each方法，前面已经介绍过了。此处，简单介绍这些方法的用法，更具体的需找官方手册查阅。

在介绍下面这些方法之前，先引入一个概念：容器类对象。容器类对象表示定义了each方法可以进行迭代的对象，比如数组、hash、Range等。这些容器类对象mix-in Enumerable之后，就能使用这个模块中的方法。

<a name="blogiterator"></a>

### each相关的迭代方法

- each_with_index
- with_index
- each_cons
- each_slice
- each_with_object
- each_entry
- reverse_each
- cycle

---------

**1.each_with_index()**

```ruby
each_with_index { |obj, i| block } → enum
each_with_index → an_enumerator
```

迭代容器每个元素，将元素和其对应的index传递给语句块中的两个变量。它和each迭代是很类似的，仅仅多传递一个元素的数值索引序号。

```ruby
hash = Hash.new
%w(cat dog wombat).each_with_index { |item, index|
  hash[item] = index
}
hash   # {"cat"=>0, "dog"=>1, "wombat"=>2}
```

对于hash来说，each_with_index传递给代码块的元素是key/value组成的小数组，index是每一个元素的数值索引序号。所以，应当类似如下方式去迭代hash结构：
```ruby
h = {a: "aa", b: "bb", c: "cc", d: "dd"}
h.each_with_index do |(key,value),idx|
  puts "#{key} => #{value} at #{idx}"
end
```

输出结果：
```
a => aa at 0
b => bb at 1
c => cc at 2
d => dd at 3
```

如果第一个参数不是`(key,value)`，而是类似`|pair,idx|`这种方式传递代码块参数，那么pair是一个包含了key和value的数组，idx是数值索引号。
```ruby
h.each_with_index {|pair,idx| puts "#{pair} at #{idx}"}
```
输出结果：
```
[:a, "aa"] at 0
[:b, "bb"] at 1
[:c, "cc"] at 2
[:d, "dd"] at 3
```

但是hash的key通常被称为它的索引，而这个方法却额外使用数值索引序号去做索引，显得很怪异，所以，很少会用这个迭代方法去迭代hash结构。

---------

**2.with_index()**

这是Enumerator中的迭代方法，只要以`with_`开头的迭代方法，都是来自于Enumerator(其实也就两个，另外一个是with_object)，除非自定义或重写了。

```ruby
e.with_index(offset = 0) {|(*args), idx| ... }
e.with_index(offset = 0)
```

迭代容器每个元素，将元素和对应的index传递给语句块中的两个变量。可以指定参数offset，使得传递给语句块的index变量从offset开始(即传递每个原始index加上offset后的值)。默认offset=0，等价于each_with_index。

```ruby
a = %w(a b c d e)

a.each.with_index do |x,idx|
  p "index: #{idx}, value: #{x}"
end
## 输出：
=begin
"index: 0, value: a"
"index: 1, value: b"
"index: 2, value: c"
"index: 3, value: d"
"index: 4, value: e"
=end

a.each.with_index(2) do |x,idx|
  p "index: #{idx}, value: #{x}"
end
## 输出：
=begin
"index: 2, value: a"
"index: 3, value: b"
"index: 4, value: c"
"index: 5, value: d"
"index: 6, value: e"
=end
```

同样的，对于hash结构也要注意传递是元素和数值索引序号。
```ruby
h = {a: "aa", b: "bb", c: "cc", d: "dd"}
h.each.with_index(1) {|pair,idx| puts "#{pair} at #{idx}"}
```
输出结果：
```
[:a, "aa"] at 1
[:b, "bb"] at 2
[:c, "cc"] at 3
[:d, "dd"] at 4
```

---------

**3.each_cons()**

```ruby
each_cons(n) { ... } → nil
each_cons(n) → an_enumerator
```

迭代容器中的每个元素，都从其开始向后取连续n个元素组成一个数组传递到语句块中。

```ruby
(1..10).each_cons(3) { |a| p a }
## 输出：
=begin
[1, 2, 3]
[2, 3, 4]
[3, 4, 5]
[4, 5, 6]
[5, 6, 7]
[6, 7, 8]
[7, 8, 9]
[8, 9, 10]
=end
```

---------

**4.each_slice()**

```ruby
each_slice(n) { ... } → nil
each_slice(n) → an_enumerator
```

每次从容器中取出n个元素组成数组传递到语句块中。

```ruby
(1..10).each_slice(3) { |a| p a }
## 输出：
=begin
[1, 2, 3]
[4, 5, 6]
[7, 8, 9]
[10]
=end
```

---------

**5.each_with_object()**

```ruby
each_with_object(obj) { |(*args), memo_obj| ... } → obj
each_with_object(obj) → an_enumerator
```

实现类似于inject/reduce的功能。迭代每个元素，然后将元素传递给语句块中的变量(args)，于此同时，还会指定一个obj参数对象作为memo_obj变量的初始值，最后经过语句块的操作之后，返回obj**最初引用**的对象。

所以，args是每次迭代时传递的元素值，obj是memo_obj的初始值，memo_obj在每次迭代过程中会改变。最后，返回最初obj所指向的对象。

必须注意，**obj应该传递可变对象，并保证在语句块中没有改变obj对象的引用，否则each_with_object将总是返回初始值**。见下面示例分析。

```ruby
evens = (1..10).each_with_object([]) {|i, a| a << i*2}
#=> [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

上面的例子中，迭代Range容器中的每个元素并将之传递给语句块中的变量i，同时传递一个初始空数组对象给语句块中的变量a，这就像是在语句块中初始化了一个空数组。然后，每次迭代过程中都将i乘2后放入数组的尾部。最后返回这个数组对象a。

再比如下面的例子中，传递初始字符串对象`x`，两个语句块都将每次迭代的字母追加到这个字符串对象的尾部，但是结果却不同。

```ruby
a = ("a".."c").each_with_object("x") {|i,str| str += i}
b = ("a".."c").each_with_object("x") {|i,str| str << i}
p a      # "x"
p b      # "xabc"
```

这是因为，虽然`str += i`每次都会创建新的对象并赋值给str，使得str从引用原有的字符串对象`x`改变为引用另一个新对象，每次迭代都会改变引用目标，使得最后返回时，只能返回最初始的字符串对象`x`。

而`str << i`的方式是直接在原字符串上追加字母的，str所引用的对象一直都未改变，最后返回的原始对象也是更改后的。

而对于数值对象来说，它是不可变对象，意味着操作这个对象一定会返回一个新对象，而且下面也使用`sum += i`的方式，它本身就是返回新对象的。于是，下面的例子将总是返回初始数值对象0。

```ruby
a = (1..10).each_with_object(0) {|i, sum| sum += i}
p a   # 0
```

要实现数值相加，可以使用reduce/inject()来实现。
```ruby
a = (1..10).inject(:+)
p a  # 55

a = (1..10).inject {|sum, x| sum + x}
p a  # 55
```

---------

**6.each_entry()**

传递容器中每个元素给语句块，并从语句块中返回一个包含所有迭代时已经yield的元素组成的枚举器对象。
```ruby
class Foo
  include Enumerable
  def each
    yield 1
    yield 1, 2
    yield
  end
end
e = Foo.new.each_entry{ |o| p o }

## 输出：
=begin
1
[1, 2]
nil
=end

e.to_a     #  [1, [1, 2], nil]
```

---------

**7.reverse_each**

反向迭代容器对象中的元素。但是要小心使用该方法，因为它依赖于最后一个元素，而有些容器对象是无穷的，这时反迭代也会无穷迭代下去。

```ruby
(1..3).reverse_each { |v| p v }
```

---------

**8.cycle**

```
cycle(n=nil) { |obj| block } → nil
cycle(n=nil) → an_enumerator
```

迭代数组每个元素并调用语句块，然后循环n次整个数组的迭代过程(注意是按整个数组计算次数，而不是对每个元素，所以是先迭代完一次数组，再循环迭代第二次数组，以此类推)。所以，如果参数n=1，则cycle等价于each。

如果不给参数或参数为nil，则无限循环迭代。

```ruby
a = ["a", "b", "c"]
a.cycle {|x| puts x}     # a,b,c,a,b,c, ... forever
a.cycle(2) {|x| puts x}  # a,b,c,a,b,c
```

循环迭代有些场景是比较适用的。例如，定义一副扑克牌，包含四个花色，每个花色13张牌，不考虑王牌，一副牌共52张，一副牌需要迭代一个循环即可，2副牌则迭代2个循环。下面是这个示例：

```ruby
class Pai
  COLORS = %w[hongtao fangkuai meihua heitao]
  CARD = %w(2 3 4 5 6 7 8 9 10 J Q K A)
  attr_reader :cards

  def initialize(n = 1)
    @cards = []
    COLORS.cycle(n) { |x|
      CARD.each { |y|
        @cards << "#{x}: #{y}"
      }
    }
  end
end
```

然后使用这个类，分别创建一副牌、两副牌：
```ruby
pai = Pai.new
p pai.cards

pai2 = Pai.new(2)
p pai2.cards
```

### 布尔查询类迭代

比如，查询容器对象中的所有元素是否满足某条件、是否包含某元素、是否全都不满足某条件，或部分满足等等。

```
include?(obj) → true or false
member?(obj) → true or false

all? [{ |obj| block } ] → true or false
all?(pattern) → true or false

any? [{ |obj| block } ] → true or false
any?(pattern) → true or false

none? [{ |obj| block } ] → true or false
none?(pattern) → true or false

one? [{ |obj| block } ] → true or false
one?(pattern) → true or false
```

`include?`和`member?`等价，都使用`==`来判断容器对象中是否包含obj元素。

对于`all? any? none? one?`这几个方法，均有三种行为：
- 当使用语句块时，将判断容器中是否所有元素(all)、是否任一元素(any)、是否没有元素(none)、是否有且只有一个元素(one)满足语句块中的条件
- 当不使用语句块但给定参数时，将使用`===`的测试符号去判断容器中是否所有元素(all)、是否任一元素(any)、是否没有元素(none)、是否有且只有一个元素(one)满足条件
- 当不使用语句块且不给定参数时，将判断容器中是否所有元素(all)、是否任一元素(any)、是否没有元素(none)、是否有且只有一个元素(one)为true

**需要特别对待的是空容器和Hash结构。(1)Hash类如果采用Enumerable中的迭代方法，那么每次迭代的元素是整个键值对，但Hash重写了一些方法，使得传递的可能是key部分，不同的方法要区别对待。(2)空容器中没有元素，上面的语义可能会产生歧义，所以要关注上面四个方法的侧重含义**：

- 对于all?()，其侧重点在于没有元素返回false，由于空容器没有元素，所以没有元素返回false，于是`all?()`返回true
- 对于any?()，其侧重点在于至少有一个返回true，由于空容器没有元素，所以返回false
- 对于none()，其侧重点在于没有元素返回true，由于空容器没有元素，所以返回true
- 对于one()，其侧重点在于必须有有一个要返回true，由于空容器没有元素，所以返回false

以下是`include? all? any? none? one?`的使用示例：

```ruby
# include?
a = [ "a", "b", "c" ]
a.include?("b")       # true
a.include?("z")       # false
[1, 2.0].include?(2)  # true
```

```ruby
# all?()
%w[ant bear cat].all? { |word| word.length >= 3 } # true
%w[ant bear cat].all? { |word| word.length >= 4 } # false
%w[ant bear cat].all?(/t/)      # false
[1, 2i, 3.14].all?(Numeric)     # true
[nil, true, 99].all?            # false
[].all?                         # true
```

```ruby
# any?()
%w[ant bear cat].any? { |word| word.length >= 3 } # true
%w[ant bear cat].any? { |word| word.length >= 4 } # true
%w[ant bear cat].any?(/d/)       # false
[nil, true, 99].any?(Integer)    # true
[nil, true, 99].any?             # true
[].any?                          # false
```

```ruby
# none?()
%w{ant bear cat}.none? {|word| word.length == 5} # true
%w{ant bear cat}.none? {|word| word.length >= 4} # false
%w{ant bear cat}.none?(/d/)     # true
[1, 3.14, 42].none?(Float)      # false
[].none?                        # true
[nil].none?                     # true
[nil, false].none?              # true
[nil, false, true].none?        # false
```

```ruby
# one?()
%w{ant bear cat}.one? {|word| word.length == 4}  # true
%w{ant bear cat}.one? {|word| word.length > 4}   # false
%w{ant bear cat}.one? {|word| word.length < 4}   # false
%w{ant bear cat}.one?(/t/)       # false
[ nil, true, 99 ].one?           # false
[ nil, true, false ].one?        # true
[ nil, true, 99 ].one?(Integer)  # true
[].one?                          # false
```

下面是几个关于hash结构的布尔测试示例，并没有难度，只是在处理hash类型数据的时候要额外注意以下即可。

```ruby
h = {a:"aa",b:"bb",c:"cc"}
h.include?(:a)       # true
h.include?(:d)       # false
h.all? {|key,value| value.size >=2 } # true
h.all? {|key,value| value.size >=3 } # false
h.one? {|key,value| key =~ /b/ }     # true
```

### 搜索和选择容器内元素

---------

**1.find和detect：它们是别名，用于搜索第一个满足条件的元素**

```
find(ifnone = nil) { |obj| block } → obj or nil
find(ifnone = nil) → an_enumerator
```

返回第一个找到的满足语句块中条件的元素。如果找不到，则返回nil。如果给定了参数，则在找不到时调用这个参数，这个参数是一个回调函数，并根据这个参数的返回值作为find的返回结果。

```ruby
[1,2,3,4,5].find {|x| x>3}   #=> 4
[1,2,3,4,5].find {|x| x>10}  #=> nil

cb = lambda {"no elements"}
[1,2,3,4,5].find(cb) {|x| x>10}
# => "no elements"
```

需要注意的是，因为find找不到元素时返回的是nil。如果要查找的条件本身就是判断容器中是否包含nil，那么结果将总是返回nil。
```ruby
[1,2,nil,4,5].find {|x| x.nil?}  #=> nil
[1,2,3,4,5].find {|x| x.nil?}    #=> nil

cb = lambda {"no elements"}
[1,2,3,4,5].find(cb) {|x| x.nil?}
#=> "no elements"
```
所以，这时候find的查找成功与否就无法判断，但通过查找失败执行回调函数，或者通过`include?`等其它方法也是能判断的。

---------

**2.find_all和select和filter：等价，用于找出容器中所有满足条件的元素，以数组方式返回**

还有与之对应的原处修改版本`select! filter!`方法。

```ruby
(1..10).find_all {|i| i % 3 == 0}   #=> [3,6,9]
[1,2,3,4].select {|num| num.even?}  #=> [2,4]
[:foo, :bar].filter {|x| x == :foo} #=> [:foo]
```

---------

**3.reject：和select相反，筛选出不满足条件的元素，以数组方式返回**

```ruby
(1..10).reject {|i| i % 3 == 0}
#=> [1,2,4,5,7,8,10]

[1,2,3,4,5].reject {|num| num.even?}
#=> [1,3,5]
```

---------

**4.grep：以`===`方式筛选容器中元素，以数组方式返回**

```
grep(pattern) → array
grep(pattern) {|obj| block} → array
```

将容器中各个元素e按照`pattern === e`的方式去比较，筛选出true的元素。如果给定了语句块，则将筛选得到的元素传递给语句块进行处理后返回，相当于使用了map方法。

所以等价于：
```ruby
enumerable.select {|e| pattern === e}
enumerable.select {|e| pattern === e}.map{|e| ...}
```

由于是使用`===`方式比较，这个方法就不再仅限于正则表达式的匹配了，所以非常好用。

```ruby
%w[abc def ABC].grep(/[A-Z]+/)
#=> ["ABC"]

["a","b",32,1..20,"c"].grep(1..40)
#=> [32]

[:abc,"def","ghi"].grep(Symbol)
#=> [:abc]

%w[ABC def GHi Jkl].grep(/[A-Z]+/) {|e| e.downcase}
#=> ["abc", "ghi", "jkl"]
```

需要注意的是，使用grep代码块方式时是返回代码块处理后的结果，而不是原始元素。

---------

**5.grep_v：和grep相反，筛选不满足匹配条件的元素**

```ruby
%w[abc def ABC].grep_v(/[A-Z]+/)
#=> ["abc", "def"]

["a","b",32,1..20,"c"].grep_v(1..40)
#=> ["a", "b", 1..20, "c"]

[:abc,"def","ghi"].grep_v(Symbol)
#=> ["def", "ghi"]
```

---------

**6.group_by：按规则对容器中的元素分组，并放进hash结构中返回**

```
group_by { |obj| block } → a_hash
group_by → an_enumerator
```

根据语句块中指定的分组依据，对容器中的所有元素进行分组并归类到hash结构中返回。分组依据作为hash的key，符合某分组的所元素放进一个数组作为hash的value。

看示例很容易理解：

```ruby
%w(Perl Shell PHP Golang Ruby C Python).group_by {|e| e.size}
=begin
{4=>["Perl", "Ruby"],
 5=>["Shell"],
 3=>["PHP"],
 6=>["Golang", "Python"],
 1=>["C"]}
=end
```

上面按照元素的字符数量进行分组，只有一个字符长度的元素是`C`，4个字符长度的元素是Perl和Ruby，等等。它的分组依据即字符串长度作为hash结构的key，满足各分组的元素放进数组中作为hash结构的value。

---------

**7.partition：类似于group_by，只不过只分两类：true和false，语句块返回true的元素放进一个子数组，语句块返回false的元素放进另一个子数组，这两个数组作为子数组返回**

```ruby
(1..10).partition {|e| e > 5 and e < 8}
#=> [[6, 7], [1, 2, 3, 4, 5, 8, 9, 10]]
```

上面语句块的条件是筛选大于5小于8的元素，所以Range对象中的6和7返回true，其余的均返回false，它们均放于子数组返回。

所以，返回结果的数组的第一个元素是true对应的子数组。例如:
```ruby
(1..10).partition {|e| e>5 and e< 8}[0]
#=> [6, 7]
```

---------

**8.first：选择前几个元素，默认选择第一个元素**

值得关注的是，Enumerable并没有对应的last方法，因为有些容器类型是无穷迭代下去的，没有最后一个元素，它无法定义通用的last方法。但有些容器类型自己定义了last方法，比如Array和Range都定义了last。

```ruby
first → obj or nil
first(n) → an_array
```

如果容器为空，则第一种语法返回nil，第二种语法返回空数组。

```ruby
%w(abc def ghi jkl).first      #=> "abc"
%w(abc def ghi jkl).first(2)   #=> ["abc", "def"]
%w(abc def ghi jkl).first(200)
#=> ["abc", "def", "ghi", "jkl"]

[].first     #=> nil
[].first(2)  #=> []

{a:"aa", b:"bb"}.first  #=> [:a, "aa"]
```

---------

**9.take和drop：选择前几个容器元素、删除前几个容器元素**
**10.take_while和drop_while：按语句块要求选择、删除容器元素**

```ruby
drop(n) → new_ary
take(n) → new_ary

drop_while {|obj| block} → new_ary
drop_while → Enumerator

take_while {|obj| block} → new_ary
take_while → Enumerator
```

例如：
```ruby
[1,2,3,4,5,0].take(2) #=> [1,2]
[1,2,3,4,5,0].drop(2) #=> [3,4,5,0]

[1,2,3,4,5,0].drop_while {|x| x<3}
#=> [3, 4, 5, 0]
[1,2,3,4,5,0].take_while {|x| x<3}
#=> [1, 2]
```

---------

**11.max和min：选择容器中最大、最小的一个或n个元素**
**12.max_by和min_by：根据语句块选择容器中最大、最小的一个或n个元素**
**13.minmax和minmax_by：选择容器中最大、最小的元素放进数组中返回**

```
max|min → obj
max|min { |a, b| block } → obj
max|min(n) → array
max|min(n) { |a, b| block } → array

max|min_by {|obj| block } → obj
max|min_by → an_enumerator
max|min_by(n) {|obj| block } → obj
max|min_by(n) → an_enumerator

minmax → [min, max]
minmax { |a, b| block } → [min, max]

minmax_by { |obj| block } → [min, max]
minmax_by → an_enumerator
```

容器元素之间使用`<=>`符号比较大小。

如果没有给定参数n，则表示只选出一个即可，给定参数n表示选出前n个。

对于max和min方法，如果没有给定语句块，则两两元素之间直接比较，如果给定语句块，则按照语句块中的比较规则进行比较。

```ruby
arr = %w(Perl PHP Shell Python Ruby)

arr.max     #=> "Shell"
arr.min     #=> "PHP"
arr.min(2)  #=> ["PHP", "Perl"]

arr.min {|a,b| a.size <=> b.size}
#=> "PHP"
arr.min {|a,b| b.size <=> a.size}
#=> "Python"
arr.max {|a,b| a.size <=> b.size}
#=> "Python"

arr.min_by {|e| e.size}
#=> "PHP"

arr.minmax
#=> ["PHP", "Shell"]
arr.minmax {|a,b| a.size <=> b.size}
#=> ["PHP", "Python"]

arr.minmax_by {|e| e.size}
#=> ["PHP", "Python"]
```

---------

**14.uniq：去除容器中重复元素**

```
uniq → new_ary
uniq { |item| ... } → new_ary
```

使用`eql?`比较各元素，然后去除重复元素。可以使用代码块，在代码块中指定比较依据。

```ruby
arr = %w(aa bb bB cc cC aa)
arr.uniq
#=> ["aa", "bb", "bB", "cc", "cC"]

arr.uniq {|e| e.downcase }
#=> ["aa", "bb", "cc"]

arr.uniq {|e| e.size }
#=> ["aa"]
```

---------

**15.inject和reduce：等价**

```
reduce(initial, sym) → obj
reduce(sym) → obj
reduce(initial) { |memo, obj| block } → obj
reduce { |memo, obj| block } → obj
```

用法见示例。
```ruby
# 以下等价，均返回45
(5..10).inject { |sum,e| sum + e }
(5..10).inject(0) { |sum,e| sum + e }
(5..10).inject(:+)
(5..10).inject(0,:+)
```

当使用语句块时，传递两个变量给语句块，第一个变量是初始化变量且在执行结束后需要返回的变量，正如上面的sum。

如果不给参数，则sum直接取第一个元素作为其初始化值。如上面第1条语句，sum初始化值为5。执行过程如下：取第一个元素5赋值给sum，取第二个值6赋值给e，进行加总得到11再次保存在sum变量中，进入下一次迭代，取第三个值7赋值给e，进行sum和e的加总得到18再次保存在sum变量中。以此类推，直到最后一个元素迭代完，所有元素的加总值45保存在sum变量中，inject返回这个sum变量，即45。

如果给定参数，则sum的初始化值取该参数值，如上面第2条语句，sum初始化值为0。于是执行过程为：sum初始化赋值为0，第一次迭代，取第一个元素5赋值给e，进行加总得到5再次保存在sum变量中，第二次迭代，取第二个元素6赋值给e，进行加总得到11再次保存在sum变量中，直到迭代完成。

如果使用了sym参数，它实际上是语句块的一种简写方式，例如`:+`表示调用每个元素的`+`方法，并将加法运算的结果保存到一个临时变量中，最后返回这个临时变量的值。

如果使用的是`inject(:*)`，则等价于`{|prod,e| prod * e}`，表示阶乘。

再比如，取出字符长度最大的元素：
```ruby
arr = %w(PHP C Java Perl Python)

arr.inject { |var, e| var.size > e.size ? var : e }
#=> "Python"
```

---------

**16.zip**：容器的交织

```
obj.zip(arg, ...) → new_ary
obj.zip(arg, ...) {|arr| block} → nil
```

将0或多个容器对象的元素一一对应地合并起来(obj和arg都是容器对象)。合并时，数量以obj的元素数量为准，不足的以nil补足，多余的元素则忽略。

```ruby
a = [ 4, 5, 6 ]
b = [ 7, 8, 9 ]
[1, 2, 3].zip(a, b)   # [[1,4,7],[2,5,8],[3,6,9]]
[1, 2].zip(a, b)      # [[1, 4, 7], [2, 5, 8]]
a.zip([1, 2], [8])    # [[4,1,8],[5,2,nil],[6,nil,nil]]
```

如果使用语句块的方式，那么每次合并后的子数组将传递给语句块中的变量，然后应用语句块的逻辑，但注意它返回的结果为nil。所以，zip()语句块中的block应当是那些能做实际操作的语句，而不是像一种返回值的方式返回操作后的结果，这样会丢弃结果。看下面示例：

```ruby
a = [ 4, 5, 6 ]
b = [ 7, 8, 9 ]
[1, 2].zip(a, b)      # [[1, 4, 7], [2, 5, 8]]

[1, 2].zip(a, b) do |x|
  x.reduce(:+)         # (1).不合理
end

[1, 2].zip(a, b) do |x|
  p x.reduce(:+)       # (2).合理
end

sum = 0
[1, 2].zip(a, b) do |x|
  sum += x.reduce(:+)  # (3).合理
end
p sum
```

首先，上面zip()两次传递到语句块中的变量分别是`[1, 4, 7]`和`[2, 5, 8]`。`x.reduce(:+)`表示将x容器(此处为数组)中的元素全都相加。所以，第一次迭代语句块时，`x.reduce(:+)`的结果是1+4+7=12，第二次迭代的结果是2+5+8=15。

但是在(1)中，它仅仅只是相加了，加了之后结果就被丢弃了，它不会作为新数组的元素返回，因为zip()使用语句块时返回的是nil，而不是新数组。

所以，在(2)中，对相加之后的结果加了一个p()动作将其输出，也就是使用了x.reduce的结果，并没有丢弃。

同理，在(3)中，将相加之后的结果加总到sum上，使得最后sum的值被保留，这里也使用了x.reduce的结果，并没有丢弃。


### map和collect方法

这两个方法是等价的别名。其含义是将一个数组按一定的方式映射为另一个数组。

```
map { |obj| block } → array
map → an_enumerator
```

map方法是一个加工厂。用于将容器中的数据进行一番处理，然后将每个处理后得到的结果放到一个新数组中，最后返回这个数组。

```ruby
(1..3).map { |i| i*i }  #=> [1,4,9]
(1..3).collect {"a"}  #=> ["a","a","a"]

%w(Perl PHP Ruby).map {|e| e.downcase}
#=> ["perl", "php", "ruby"]
```

注意，Ruby的map方法是完全等量映射，返回和原容器元素数量相等的数组，如果某个元素不满足条件，则自动在对应数组索引处设置为nil。

```ruby
(1..5).map {|e| if e>3;e;end }
#=> [nil, nil, nil, 4, 5]

%w(Perl PHP Ruby).map {|e| if /p/i =~ e;e;end}
#=> ["Perl", "PHP", nil]
```

Ruby map方法在某些情况下可以简写：如果语句块中仅是通过一个方法来操作迭代的元素，则可以简写。

例如，上面语句块中进行e.downcase操作，可以简写为：
```ruby
%w(Perl PHP Ruby).map(&:downcase)
#=> ["perl", "php", "ruby"]
```

一个`&`相当于调用的意思(也许Ruby中该符号的含义是取自Perl的子程序调用)，表示对每个迭代的元素去调用`:downcase`方法，然后将操作之后的元素放进数组中返回。

所以，map方法的简写只适用于某些情况，像语句块中包含了if语句的，这是无法简写的。但是，可以将这些语句块中的语句写成Proc对象，然后再简写调用Proc对象。

其它方法有时候也能做这样的简写，不过这里先不多做解释。

另外，需注意的是map的返回值问题。它是将迭代中的每个元素进行操作后，根据语句块中的操作返回值放进数组的。但语句块中的返回值在不小心的时候可能会出乎意料。

```ruby
%w(Perl PHP Ruby).map {|e| puts e.downcase}
perl
php
ruby
#=> [nil, nil, nil]
```

可见，上面的map直接输出了小写的元素，但是返回的数组却全是nil。这是因为puts方法的返回值是nil，map会将这个返回值放进数组。

最后，数组结构还定义了一个`map!`原处修改的方法，也只有数组这种容器类才有这个方法，因为map返回的是数组，其它容器类型想定义`map!`原处修改也不允许。

### 容器类对象的排序

要排序某个容器类对象，这个容器对象必须定义`<=>`方法。

当然，对于Array/Range等容器对象来说，它们已经设计好了`<=>`。对于其它容器对象，比如Hash来说，比如自定义的容器类，它们会从Object中继承`<=>`，只不过继承而来的`<=>`可能并非期望的比较方法：Object中的`<=>`只做`==`比较，在为真的时候返回0，其它时候无论是大于还是小于均返回nil。

定义了`<=>`之后，容器对象就能使用sort和sort_by这两个方法来排序。`sort_by`采用的是Schwartzian转换排序方式，它将每个元素的排序依据存储起来，使得每个元素都只需要计算一次。对于小型容器来说，`sort_by`的效率可能比sort还低一点，因为它多了额外数据存储的过程，但是对于大型容器，特别是每次取值速度较慢的场景(比如比较文件大小`File.size`)，因为其每个元素只计算一次，所以`sort_by`效率要高于sort。

```ruby
arr = %w(C C++ Ruby Shell PHP Python)

arr.sort
#=> ["C","C++","PHP","Python","Ruby","Shell"]

arr.sort {|a,b| a.size <=> b.size}
#=> ["C","C++","PHP","Ruby","Shell","Python"]

arr.sort_by {|e| e.size}
#=> ["C","C++","PHP","Ruby","Shell","Python"]

# 可以指定多个排序依据
arr.sort_by {|e| [e.name,e.size] }
```
