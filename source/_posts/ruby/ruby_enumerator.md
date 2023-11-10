---
title: Ruby Enumerator枚举器详细说明
p: ruby/ruby_enumerator.md
date: 2020-05-14 13:57:33
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Enumerator枚举器用法详细说明

## Enumerator和Enumerable的关系和区别

在开始Enumerator之前，需要搞清楚Enumerator和Enumerable之间的区别。

Enumerable是可枚举的意思，它是一个模块，内部定义了很多枚举方法，或者说是迭代方法，用来枚举容器中的所有元素。

Enumerator是枚举器，它的每一个对象是可枚举对象。既然是枚举器，自然可以枚举各个元素，只不过它相比于Enumerable中的各种迭代方法总是一次性枚举完所有元素才罢休，Enumerator是一个对象，保存了枚举状态，可以在任何时候停下来，然后在需要的地方继续枚举。

所以，Enumerable的迭代是『原子性』的，Enumerator的迭代是可暂停的。就像视频播放一样，Enumerable是只要开始播放了就停不下来，直到播放完成，除非中间出错。而Enumerator提供了暂停键和继续键，还支持枚举的重置操作，即从头开始播放。

Enumerator之所以能迭代，是因为该类定义了each方法(而Enumerable模块中并没有each方法)，而且该类默认已经mix-in Enumerable，所以已经具备了Enumerable中的各种迭代方法。


## 创建枚举器：new方法

枚举器即Enumerator对象。创建枚举器的方式有2种：通过Enumerator类的new方法创建，通过Object类提供的enum_for(或别名to_enum)创建。

先看看使用`Enumerator::new`创建的方式。

```ruby
e = Enumerator.new do |y|
  i = 0
  while i < 10
    y << i
    i += 1
  end
end

puts e
e.each { |i| puts "hello #{i}" }
p e.take(5)
```

输出结果：
```
#<Enumerator:0x000000023ebed8>
hello 0
hello 1
hello 2
hello 3
hello 4
hello 5
hello 6
hello 7
hello 8
hello 9
[0, 1, 2, 3, 4]
```

上面的new方法使用了语句块，语句块有一个变量y，这个y是一个Yielder对象，Yielder对象可以使用`<<`方法来yield数据给语句块。

![](/img/ruby/1589464480710.png)

这就是通过Enumerator.new创建的Enumerator对象，创建了这个对象之后，就自动具有了each方法，加上Enumerator默认已经mix-in Enumerable模块，这个Enumerator对象就可以使用很多的迭代方法。

<a name="blogmy_times2"></a>

现在，重新定义前面实现的my_times方法，要求在没有给定语句块的时候返回一个Enumerator对象。
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

但是这样的定义有点丑陋，下面将介绍enum_for，通过它重构上面的my_times，将更加完善，my_times的重构代码参见：[enum_for重构my_times](#blogmy_times3)。

## 创建枚举器：enum_for/to_enum

这两个方法等价，是别名关系。在官方手册及其它资料上显示它们是Kernel提供的。但却是在Object手册中找到这两个方法的。

不管这两个方法是Object提供还是Kernel提供，它们都是用来创建枚举器即Enumerator对象，而且它们的创建方式和Enumerator.new方法创建的不太一样。

```
enum_for(method = :each, *args) → enum
enum_for(method = :each, *args){|*args| block} → enum
```

enum_for带语句块的语法用于定义枚举器的size，暂不考虑这个。

对于不使用语句块的enum_for，它用于绑定一个对象上的方法，默认是each方法，如果所绑定的方法需要参数则也可以在enum_for的第二个参数开始传递参数。

使用一个示例演示，则很容易理解：
```ruby
arr = %w(Perl PHP Shell Python)
e = arr.enum_for(:select)
#=> #<Enumerator: ["Perl", "PHP", "Shell", "Python"]:select>
e.each {|e| /p/i =~ e }
#=> ["Perl", "PHP", "Python"]
```

上面的arr是数组对象，将其作为接收者调用enum_for方法，该方法的参数是`:select`，这表示创建一个Enumerator对象，并且这个枚举器对象的each方法"重定向"到select方法。如果不给参数的话，则默认绑定的是arr这个对象的each方法。

也就是说，调用这个枚举器的each方法时，实际上会去调用arr的select方法，也就是数组的select方法。所以，上面e.each迭代的时候，语句块中的代码是为select服务的。

由于e对象是一个枚举器对象，它可以调用Enumerable模块的很多方法，该模块的方法都依赖于each方法，也就是依赖于枚举器的each，而枚举器的each又绑定了数组的select方法，所以Enumerable中的各迭代方法的调用变得依赖于select方法：只对筛选出来的元素进行迭代。

例如，调用这个枚举器对象的take方法：
```ruby
e.take(2)
#=> ["Perl", "PHP"]
```

其实，从返回的枚举器对象的输出结果看，不难发现绑定了select方法的枚举器对象e，其实等价于：
```ruby
arr = %w(Perl PHP Shell Python)
e = arr.select
#=> #<Enumerator: ["Perl", "PHP", "Shell", "Python"]:select>
```

既然现在e对象上的迭代方法都依赖于select方法，而select方法筛选元素是依赖于语句块条件的，如果不给语句块会如何？又或者，e对象上调用其它迭代方法时这个迭代方法自己也需要语句块，这时select的语句块和这个迭代方法的语句块该如何共存？

其实问题不大，因为不写上select的语句块时，它会迭代所有元素。而select和迭代方法都有语句块时，将其通过链式的方式链接起来即可。

```ruby
e.take(4)
#=> ["Perl", "PHP", "Shell", "Python"]

e.each {|x| /p/i =~ x}.map {|x| x.downcase}
#=> ["perl", "php", "python"]
```

这样select和map的两个语句块都具备了，只不过select的语句块是通过each去迭代的，在枚举器对象绑定了select方法后，枚举器的each和select就是等价的了。

并非总是能够链接多个枚举器，例如绑定的不是select方法而是find方法，find方法查询到一个元素后就立即返回这个元素，这个元素可能不再是容器对象，这时就不能再它后面链接map或其它迭代方法。

```ruby
arr = %w(Perl PHP Shell Python)
e = arr.enum_for(:find)
e.each {|x| /p/i =~ x}
#=> "Perl"
e.each {|x| /p/i =~ x}.map {|x| x.downcase}
# NoMethodError (undefined method `map' for "Perl":String)
```

报错已经很明显了，对于字符串『Perl』，它没有map方法。

可以在enum_for方法中指定绑定方法的参数。例如，绑定take方法，并指定take的参数：
```ruby
arr = %w(Perl PHP Shell Python)
e = arr.enum_for(:take, 2)
#=> #<Enumerator: ["Perl", "PHP", "Shell", "Python"]:take(2)>

e.each {}
#=> ["Perl", "PHP"]

e.each {|x| /abcd/i =~ x }
#=> ["Perl", "PHP"]
```

注意上面enum_for的第二个参数是2，而创建出来的枚举器中已经显示了`take(2)`，这就是指定参数的方式。在e.each时，代码块为空，其实代码块的逻辑可以随意定义，它不会影响结果，因为take直接从迭代器中取数据，代码块中的逻辑已经不需要。但是却不能缺少这个代码块，缺少代码块的each方法表示返回这个枚举器对象，而不会去迭代这个枚举器对象。

<a name="blogmy_times3"></a>

最后，再使用enum_for来重构my_times的实现：
```ruby
class Fixnum
  def my_times
    if block_given?
      for i in 1..self
        yield i
      end
    else
      self.enum_for(:my_times)
    end
  end
end

e = 10.my_times
e.each {|x| puts x}

=begin

# 之前Enumerator.new的版本
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

=end
```

当不给定代码块时，将返回`self.enum_for(:my_times)`，这样下次再使用这个枚举器的时候，它使用的仍然是my_times，如果此时给定代码块，则在`if block_given?`中判断通过，从而开始进行枚举。

相比于前面使用Enumerator.new实现的my_times版本，这个版本逻辑更加清晰，返回的Enumerator也更显优美。

再来看下，Rubinius中的times方法(定义在`rubinius-VERSION/core/integer.rb`文件中)的实现方式：
```ruby
  def times
    return to_enum(:times) { self } unless block_given?

    i = 0
    while i < self
      yield i
      i += 1
    end
    self
  end
```

## 链式枚举器

前面已经演示过my_times方法如何在不使用语句块的情况下返回一个它的枚举器对象。

其实很多迭代方法在不使用语句块的时候都是返回枚举器对象的。比如each、map、select、find、inject等等，这和绑定这些迭代方法的枚举器是等价的。正如前面的示例：

```ruby
arr = %w(Perl PHP Shell Python)
e1 = arr.enum_for(:select)
e2 = arr.select
```

这两个枚举器是等价的，它们都是将枚举器的each方法绑定了select方法。

唯一需要注意的是，枚举器对象虽然绑定的方法可能不是默认的each方法，但是枚举器的each方法却是相当于这个绑定方法的别名。

下面这两者是等价的。
```ruby
arr.select.map
e2.each.map
```

但是，select是需要语句块的，直接这样将它们链接起来就失去了select筛选的效果，它会迭代所有arr元素。所以，得按照下面这种方式来链接多个迭代方法：
```ruby
arr.select {|x| /p/i =~ e}.map {|x| x.downcase}
arr.each {|x| /p/i =~ e}.map {|x| x.downcase}
```

前面已经描述过，有时候是不能随便链接多个迭代方法的，因为中间的迭代方法返回的可能不能和链接的迭代方法兼容。正如前面所说的，当枚举器绑定的是find方法时，该枚举器筛选出来的将是一个字符串，它不是容器对象，不能在此基础上链接其它迭代方法。

```ruby
arr.find {|x| /p/i =~ x}.map {|x| x.downcase}
```

而且，绑定了迭代方法后，枚举器的each方法相当于是这个迭代方法的别名。只不过需要注意的是，很多迭代方法是基于each的，所以枚举器调用其它迭代方法时，可能会出现一些意料之外的诡异现象。

例如，hash结构的each方法返回的是hash结构，创建一个hash的枚举器，直接使用默认的绑定方法：hash.each：
```ruby
h = {a: "aa", b: "bb", c: "cc"}
e = h.enum_for

h.each {}
#=> {:a=>"aa", :b=>"bb", :c=>"cc"}
e.each {}
#=> {:a=>"aa", :b=>"bb", :c=>"cc"}
```

这时候hash对象的each和枚举器e的each方法是等价的。但是，如果调用枚举器的select方法，以及调用hash对象的select方法，它们将出现不同结果：
```ruby
h.select {|e| e}
#=> {:a=>"aa", :b=>"bb", :c=>"cc"}

e.select {|e| e}
#=> [[:a, "aa"], [:b, "bb"], [:c, "cc"]]
```

既然枚举器的select是基于枚举器自身的each方法的，而枚举器的each又绑定了hash对象的each方法，为何它们会产生不同结果？

其实原因很简单，这是因为e.select这个方法调用的是Enumerable中的select方法，而`Enumerable#select`返回的是一个数组。

所以，对于枚举器的使用来说，尽管知道枚举器上的各种迭代方法都基于枚举器自身的each方法，但还是需要注意到这些迭代方法都来自于Enumerable。这可能会导致容器对象自身调用迭代方法和枚举器调用迭代方法的结果不同。

## 分析枚举器的链式迭代

链式迭代应用非常广泛，但在用法上有几点需要注意。

### 链式迭代作为中间代理对象

例如：
```ruby
(1..10).map.find
```

上面这个链式的语句，无论是中间的map还是最终的find，都返回枚举器对象。而且到目前为止一直都没有去执行它，所以没有任何迭代操作的进行，因为迭代是需要yield的，上面还没有触发yield。回顾一下那些迭代方法的用法就知道了，当给定了语句块的时候，执行yield进行迭代，当没有给定语句块的时候，就返回枚举器，它们是两个分支操作。

所以，上面的语句几乎不消耗时间，无论需要迭代的内容有多少，迭代的操作有多复杂，它仅仅只是需要返回一个枚举器对象。就像是播放器播放视频一样，返回一个枚举器就相当于定义了播放器怎么打开、怎么播放这个视频，这几乎是不需要花时间的，只有在真正播放的时候才需要花时间，也就是需要枚举器进行枚举的时候才开始执行。

需要枚举元素的可能性有很多种，比如通过迭代方法进行迭代、调用to_a转换成数组，等等，它们都需要枚举出一些元素。

```ruby
(1..10).map.find {|x| x > 3}  # 花时间
(1..10).map.find.to_a         # 花时间

e = (1..10).map.find          # 不花时间
e.each {|x| x > 3}            # 花时间
```

所以，对于链式迭代语句中的中间枚举器，它们相当于是中间的代理对象。

但是有一点需要说明：对于链式迭代，虽然在尚未开始枚举的时候不会产生数据，但是在开始枚举之后，枚举的过程是从左至右的，它必须先枚举完一个迭代方法，使用中间变量(内存中)保存枚举出来的中间结果，然后再继续向右执行迭代方法，右边的迭代方法将基于此中间结果进行枚举。所以，默认情况下的链式迭代并不会因为链接在一起而减少内存使用量，也不会减少迭代时间，除非使用了`Enumerator::Lazy`惰性枚举器对象。

### 链式迭代的结果分析

在没有掌握分析方法的时候，链式迭代出来的结果还是挺难理解的。

先借助下面这两个语句进行分析，这两个语句在返回结果上是等价的。
```ruby
[1, 2, 3].each_with_index.map{|i, j| i * j}
#=> [0, 2, 6]

[1, 2, 3].map.each_with_index{|i, j| i * j}
#=> [0, 2, 6]
```

分析的方法很简单，把每个过程的迭代方法通过to_a转换成数组去观察返回结果即可。

例如：
```ruby
[1,2,3].each_with_index.to_a
#=> [[1, 0], [2, 1], [3, 2]]

[1,2,3].each_with_index.map.to_a
#=> [[1, 0], [2, 1], [3, 2]]
```

说明`each_with_index`之后的map方法进行迭代的时候，是根据`each_with_index`结果的模型进行迭代的，尽管它在开始迭代时产生的中间结果可能不是这样的数组。

所以，map的语句块中给定了两个语句块变量`|i, j|`，对于每一次迭代，它的赋值过程类似于：
```
i, j = [1, 0]  # 第一次迭代赋值
i, j = [2, 1]  # 第二次迭代赋值
i, j = [3, 2]  # 第三次迭代赋值
```

最后，语句块中的代码逻辑`i * j`是按照map方法给定的。根据map方法的特性，它是将`i * j`的结果放进新数组中进行返回，所以返回结果是`[0,2,6]`。

其实这里已经有问题出现了，为什么代码块的逻辑是根据map制定的，而不是根据`each_with_index`制定的？

上面第二条语句的分析过程也是一样的，通过to_a去分析。
```ruby
[1, 2, 3].map.to_a
#=> [1, 2, 3]

[1, 2, 3].map.each_with_index.to_a
#=> [[1, 0], [2, 1], [3, 2]]
```

所以，语句块中赋值的方式`|i, j|`和前面是一样的，需要两个语句块变量。

问题在于，这条语句中的map是在`each_with_index`的前面，但是为什么语句块的逻辑也是按照map制定的？

结合前面的问题，为什么map和`each_with_index`的顺序不同，但需要的语句块的逻辑却都按照map去制定？

这是因为`each_with_index`这类迭代方法比较特殊，它们的逻辑只是进行迭代，不考虑返回值问题。其它具有此特性的迭代方法还有each、with_index、each_index等。对于这类只迭代不考虑返回值的迭代方法，它们链接顺序在前面还是后面，都无关紧要，语句块仍然需要按照那些需要考虑返回值的迭代方法来制定。

比如：
```ruby
[1, 2, 3].select.each {|e| e > 1}
#=> [2, 3]
[1, 2, 3].each.select {|e| e > 1}
#=> [2, 3]
```

可能这里会对『不需要考虑返回值的迭代方法』有点疑惑，什么样的迭代方法是这类迭代方法？其实很简单，假如需要写它们的语句块，语句块中似乎需要puts操作才有结果，那么这个迭代方法就属于这类迭代方法。例如，each的语句块：
```ruby
(1..5).each {|e| puts e if e > 2}
(1..5).each {|e|      e if e > 2}
```

上面的第一条语句才有效果，它输出了满足条件的元素，而第二条语句是完全多余的，因为each不会根据语句块的逻辑来返回数据，虽然进行了筛选操作，但是所有的元素都直接被丢弃了。这类迭代方法，都需要在语句块中添加额外的逻辑来体现语句块确实执行了并产生了效果，而其它迭代方法直接通过语句块中的代码的返回值就能看出来它产生了效果。

所以，`map.each_with_index`和`each_with_index.map`是等价的，尽管顺序不一样。

但是，如果链式的迭代语句中包含了多个非此类迭代方法的方法，那么它们的语句块就需要各自单独制定。

例如，将map和find一起使用，将map和select一起使用等等。
```ruby
(1..5).map.select {|e| e > 2 }
#=> [3, 4, 5]

(1..5).select.map {|e| e > 2 }
#=> [false, false, true, true, true]

(1..5).select {|e| e > 2}.map {|e| e}
#=> [3, 4, 5]

(1..5).map {|e| e * 2}.select {|e| e > 6 }
#=> [8, 10]
```

了解了以上内容，就很容易分析链式迭代语句该怎么写才是正确的以及结果大概是怎样的。

## Enumerator自带的几种迭代方法

Enumerator自身定义了几个迭代方法：each、each_with_index、each_with_object、with_index、with_object。

根据上面的分析，这几个方法都是不考虑语句块代码逻辑返回值的迭代方法，所以它们可以在链式迭代语句中的任意位置，无关于顺序。

关于这几个方法的用法，可参见[Enumerable中each相关的迭代方法](/ruby/ruby_enumerable#blogiterator)。

## 扩展枚举器

Enumerator提供了一个方法`+`，它可以将一个枚举器直接扩展到另一个枚举器上。

```ruby
# 将数组调用to_enum转换成Enumerator
e1 = (1..3).each + [4, 5]
# => #<Enumerator::Chain: [#<Enumerator: 1..3:each>, [4, 5]]>
e1.to_a
#=> [1, 2, 3, 4, 5]

e2 = (1..3).each + [4, 5].each
#=> #<Enumerator::Chain: [#<Enumerator: 1..3:each>, #<Enumerator: [4, 5]:each>]>
e2.to_a
#=> [1, 2, 3, 4, 5]

e3 = (1..3).to_enum + [4, 5].to_enum
#=> #<Enumerator::Chain: [#<Enumerator: 1..3:each>, #<Enumerator: [4, 5]:each>]>
e3.to_a
#=> [1, 2, 3, 4, 5]
```

为什么要用枚举器的扩展方式，而不是使用各容器对象本身的扩展方式呢？
```ruby
[1,2,3] + [4,5]
```

这其实是有很大好处的。例如上面直接使用数组的扩展方式，它需要先迭代完这两个数组对象(5次迭代过程)，并产生一个中间结果集保存它的扩展结果：5个元素的数组。

而如果是`[1,2,3].to_enum + [4,5].to_enum`，则不会产生任何一次迭代操作。枚举器的扩展方式会将枚举操作延迟到需要枚举的时候再进行。


## 枚举器保护对象：避免对象被修改

枚举器对象有一个比较实用的技巧：保护对象不被修改。

Ruby中方法传参的方式是按引用传递而非按值传递的，所以将数组这种类似的容器结构传递给方法时，在方法中是可以直接修改这个对象的。例如:

```ruby
def m(arr)
  arr << "x"
end

arr = %w(a b c)
m(arr)

arr
#=> ["a", "b", "c", "x"]
```

但是，可以通过构建枚举器对象作为方法的参数，枚举器对象不可变的，所以方法内部不能修改。

```ruby
def m(arr)
  arr << "x"
end
arr = %w(a b c)
m(arr.to_enum)      # 报错
```

要避免原数组被修改，方式有多种，比如调用方法时先拷贝或克隆数组，传递这个副本数组给方法的参数，又或者直接冻结一个对象。例如：

```ruby
m(arr.dup)
m(arr.clone)
m(arr.freeze)
```

至于使用哪种方式，要看具体的需求：
- dup和clone都是拷贝副本，如果原容器对象是可变的，那么在方法内也能修改，只不过修改的不是同一个对象，方法内修改的是副本而已
- freeze这种方式不太可取，因为冻结是原处修改的状态，冻结之后它就成为一种永久状态，尽管在方法内不能再修改，但是在方法外也不能再修改，这是副作用
- to_enum这种方式则不影响外部，但却能保证方法内部不被修改


## 手动迭代枚举器对象

前面说过，那些迭代方法总是一次性将容器对象中的所有元素都迭代完。但创建它们的枚举器后，由于枚举器对象保存了枚举过程中的状态，可以通过枚举器对象手动一次迭代一个元素，当所有元素都迭代完后，将抛出`StopIteration`异常。

通常，通过next手动迭代的方式称为『外部迭代』，而通过迭代方法自动迭代的方式称为『内部迭代』。

### 通过next方法进行一次迭代

例如：

```ruby
arr = %w(Shell PHP Perl Ruby)

e = arr.to_enum
#=> #<Enumerator: ["Shell", "PHP", "Perl", "Ruby"]:each>
e.next      #=> "Shell"
e.next      #=> "PHP"
e.next      #=> "Perl"
e.next      #=> "Ruby"
e.next      # StopIteration (iteration reached an end)
```

每次调用枚举器的next方法，其实是跳转去执行一次yield操作，也就是控制权交给yield。而yield操作会进行跳转，跳转到调用者上，也就是next处，所以控制权又从yield回到了next处。

所以执行完next方法，也就是next方法返回后，并不会立即跳回yield的调用位置处，因为控制权已经从yield再次跳回next处。而本次next已经执行完了，所以开始执行next下面的代码。此处是执行下一个next，而next又开始跳回yield处，于是第一个yield执行完开始返回，并执行yield下面的代码，继续向下执行，进行下一次yield，然后又跳过next。

这个跳来跳去的过程分析起来比较绕，对于不了解yield机制的人可能比较难理解，不过通过下面的分析可能会对此理解有所帮助。

例如：

```ruby
o = Object.new
def o.each
  p "Enumerator start"

  yield 1
  p "first yield"

  yield 2
  p "second yield"

  yield 3
  p "third yield"
end

e = o.to_enum
e.next       # 输出"Enumerator start"
             # 并返回 1

# yield又跳回到了上面的next处，而该next已经执行完
# 所以继续向下执行，也即是第二个next
# 所以，第一个e.next之后，并未输出first yield
# 下次e.next才跳回yield 1位置处开始向下执行
# 才会输出first yield
e.next       # 输出"first yield"
             # 并返回 2

e.next       # 输出"second yield"
             # 并返回 3

e.next       # 输出"third yield"
             # 并报错：StopIteration
```

有时候也会使用next去实现一些迭代模式，此时需要记得去处理StopIteration异常：

```ruby
itera = 1.upto(10)
begin
  puts itera.next while true
rescue StopIteration => e
  e.result
end
```

Kernel的loop方法自带了StopIteration异常的处理，所以对于next类的外部迭代是非常方便的。
```ruby
itera = 1.upto(10)
loop do
  puts itera.next
end

=begin
# 以下是Kernel#loop的在Rubinius中的实现源代码
def loop
  unless block_given?
    return to_enum(:loop) { Float::INFINITY }
  end

  begin
    while true
      yield
    end
  rescue StopIteration => e
    e.result
  end
end
=end
```

上面手动loop循环中不断的进行next，而loop中包含了yield，每next一次就yield一次，而且loop中也包含了`rescue StopIteration`异常的处理，所以在loop循环时无需再去处理该异常。

### rewind重置枚举器

前面也提到过，枚举器支持重置操作(就像从头开始播放视频)，通过`rewind`方法即可实现，它其实是将迭代过程的指针重置到最开头，相当于是重新创建了一个全新的枚举器对象，只不过这个枚举器对象还是之前的那个对象而已。

```ruby
o = Object.new
def o.each
  p "Enumerator start"

  yield 1
  p "first yield"

  yield 2
  p "second yield"

  yield 3
  p "third yield"
end

e = o.to_enum
#=> #<Enumerator: #<Object:0x00007fffcc4f1ae0>:each>
e.next   # 输出"Enumerator start"，并返回 1
e.next   # 输出"first yield"，并返回 2

e.rewind
#=> #<Enumerator: #<Object:0x00007fffcc4f1ae0>:each>
# 注意返回了一个枚举器，但地址没有改变
# 所以，迭代将从头开始

e.next   # 输出"Enumerator start"，并返回 1
e.next   # 输出"first yield"，并返回 2
e.next   # 输出"second yield"，并返回 3
e.next   # 输出"third yield"，并报错：StopIteration
```

注意，虽然rewind方法定义在Enumerator类中，但有些枚举器对象是不支持rewind操作的。

### next_values

next方法是直接枚举下一个元素，而next_values则是枚举下一个元素的同时将该元素放进数组中返回。

通过下面的列出的示例可以很容易区分它们的不同点：

```ruby
o = Object.new
def o.each
  yield
  yield 1
  yield 1, 2
  yield nil
  yield [1, 2]
end

e = o.to_enum

e.next_values   #=> []
e.next_values   #=> [1]
e.next_values   #=> [1, 2]
e.next_values   #=> [nil]
e.next_values   #=> [[1, 2]]

e.rewind

e.next          #=> nil
e.next          #=> 1
e.next          #=> [1, 2]
e.next          #=> nil
e.next          #=> [1, 2]
```

所以，整理一下next和next_values的返回值，区别将很清晰：
```
yield表达式    next返回值    next_values返回值
 yield          nil          []
 yield 1        1            [1]
 yield 1,2      [1,2]        [1,2]
 yield nil      nil          [nil]
 yield [1,2]    [1,2]        [[1,2]]
```


### peek和peek_values

peek和peek_values类似于next、next_values。区别在于next和next_values在枚举一次之后，总是会将枚举指针也向后移动一位。而peek和peek_values则是枚举一次，但是枚举指针保持不变。

其实，peek在计算机的术语里，也几乎都表示这样一种含义：返回数据，但是不前移指针或删除元素。比如在缓存中，在文件IO中，都有peek来表示返回数据但不前移指针的含义；在一些容器类型的数据结构里，也有peek表示返回元素但不删除元素，类似于pop操作。

```ruby
o = Object.new
def o.each
  yield 1
end

e = o.to_enum

e.peek  #=> 1
e.peek  #=> 1
e.peek  #=> 1
e.peek  #=> 1
```

<a name="blogfeed"></a>

### feed指定yield的返回值

yield这个关键字是有返回值的，当给定语句块时，它的返回值来自于语句块执行的返回值。

```ruby
o = Object.new

def o.each
   var = yield 1
   puts "val: #{var}"
end


o.each {|x| puts "helloworld"; "yield_return"}
# 输出：helloworld
# 输出：val: yield_return
```

但是，当通过next手动迭代枚举器时，是没有语句块的，这时候的yield返回值需要通过枚举器对象的feed方法来指定。而且，该方法只能单次yield生效，yield完就立即清除feed标记。此外，当feed标记还存在的时候，不允许再次设置feed标记。

```ruby
o = Object.new
def o.each
  var = yield 1
  puts "var: #{var}"

  var = yield 2
  puts "var: #{var}"
end

e = o.to_enum
```

现在用next手动迭代这个枚举器。
```ruby
e.next     #=> 1 开始yield
# 但是next之后，yield不会直接结束，
# 因为yield已经跳回到了next位置处，
# 而next已经执行完，所以会继续向下执行代码，
# 只有在下一次next的时候才跳回yield，
# 也就是说在下一次next的时候，yield才开始返回值
# 这时才开始将yield的返回值赋值给变量var

e.feed "feed_value1" # 设置feed标记
# 因为还没有执行下一次next，所以还没有跳回yield
# 所以这里能设置yield的返回值
e.feed "feed_value1" # 报错，因为feed标记已存在

e.next     # 输出"var: feed_value1"并返回2
# 这一次执行next，会跳回到第一个yield位置处
# 于是第一个yield开始返回，并赋值给变量var
# 因为yield已经完成了，也因此feed标记被清除了
# 然后继续向下执行puts语句
# 再执行第二个yield，但是这个yield还没有完成
# 需要下一次next才返回这个yield
# 所以，这里可以再次设置feed标记
e.feed "feed_value2"
e.next   # 输出"var: feed_value2"
         # 并抛出异常StopIteration
```

从上面的分析中可以再次体会到，yield的跳转过程是非常精细的。


## 惰性枚举器对象：Lazy

默认情况下，枚举器是"积极的"(eager)，在链式迭代中，会先从左至右一个一个的枚举，枚举完第一个迭代方法的数据，再开始枚举第二个迭代方法的数据，且后面的迭代方法基于它前面的迭代方法的枚举结果进行枚举。

这种行为在某些情况下是不太适合的，例如无穷枚举的情况：
```ruby
1.upto(Float::INFINITY).map {|x| x * 2}
```

上面的语句开始执行后，upto迭代方法将无穷枚举，由于枚举不完，所以map方法将永远无法执行，map只有在它左边的迭代方法枚举完并创建好中间结果集后才开始枚举迭代。

而惰性枚举器则是枚举一个元素，就操作一个元素，它不会等待前面枚举完才执行后面的迭代方法。所以，lazy枚举器不会产生中间结果集，它是需要一个元素就枚举一个元素，至始至终其占用的内存仅是每次枚举出来的单个元素的大小。

"积极的"枚举器处理过程和惰性枚举器处理过程如下图：

![](/img/ruby/lazy-vs-non-lazy.gif)

Ruby中只需在某个枚举器后链接Enumerable提供的lazy方法即可让这个枚举器变成惰性枚举器。

```ruby
1.upto(Float::INFINITY)
#=> #<Enumerator: 1:upto(Infinity)>

1.upto(Float::INFINITY).lazy
#=> #<Enumerator::Lazy: #<Enumerator: 1:upto(Infinity)>>
```

注意返回值的区别，这个枚举器对象不再是`Enumerator`，而是`Enumerator::Lazy`，它是Enumerator的子类，构建方式大致如下：

```ruby
class Enumerator
  ...
  class Lazy < self
    ...
  end
  ...
end
```

当在lazy枚举器对象上链接其它迭代方法，都必须将代码块都制定好，否则就会报错。这些链接在lazy枚举器上的迭代方法都将在此lazy枚举器对象上活动。

例如：
```ruby
1.upto(Float::INFINITY).lazy.map  # 报错
#=> ArgumentError (tried to call lazy map without a block)

1.upto(Float::INFINITY).lazy.map {|x| x * 2}
#=> #<Enumerator::Lazy: #<Enumerator::Lazy: #<Enumerator: 1:upto(Infinity)>>:map>
```

从返回结果中可以看到(观察嵌套层次)，`1.upto`返回的枚举对象是lazy的，map对应的枚举也是lazy的。

既然lazy枚举器对象已经附带好了语句块，那么如何运行它，才能让这个lazy枚举器开始枚举？

这要继续链接某些表示取前几个元素的方法，比如first、take等。
```ruby
lazy_e = 1.upto(Float::INFINITY).lazy.map {|x| x * 2}

lazy_e.first(3)
#=> [2, 4, 6]

lazy_e.first(5)
#=> [2, 4, 6, 8, 10]
```

其实仔细一想就很容易理解，lazy枚举器主要是为了避免待枚举元素过多的情况，若让lazy枚举器仍然枚举所有元素，lazy枚举器就失去意义了，所以对于lazy枚举器来说，它最大的用处就是选取前有限个枚举值。当然，如果待枚举元素不是无限个的时候，lazy枚举器也是可以枚举所有元素的，例如下面的代码就枚举了惰性枚举器中的所有元素，只是这是不建议的，因为它会带来性能的损失。
```ruby
1.upto(5).lazy.map {|x| x * 2}.each {|x| puts x}
```

获取lazy枚举器数据的时候要注意，有些方法并不是lazy的，比如上面的first，它会直接返回数据。而有些方法是lazy的，比如take，对于lazy类型的方法它会返回一个枚举器，这时候需要加上force方法或to_a方法(Lazy类中它们是别名)强制返回才能获取到数据。

例如：
```ruby
lazy_e = 1.upto(Float::INFINITY).lazy.map {|x| x * 2}
lazy_e.take(5)
#=> #<Enumerator::Lazy: #<Enumerator::Lazy: #<Enumerator::Lazy: #<Enumerator: 1:upto(Infinity)>>:map>:take(5)>

lazy_e.take(5).force
#=> [2, 4, 6, 8, 10]
```

至于哪些是lazy方法，哪些是非lazy方法，不好直接判断。在`Enumerator::Lazy`类中定义的方法都是lazy的。可通过以下方式查询：
```ruby
>> Enumerator::Lazy.instance_methods(false)
```

其实，只要在lazy枚举器上直接返回数据的迭代方法，都是非lazy的，而返回的是一个枚举器对象的迭代方法，都是lazy的。

### lazy枚举器的性能好还是坏？

惰性枚举器不创建中间结果集，对于大量待枚举对象来说，这可能会减小很多内存消耗。

而『积极的』枚举器会创建中间结果集，对于大量待枚举对象来说，可能会消耗很多额外的内存来保存这些中间结果集，特别是链接了多个迭代方法时，占用的内存量将成倍增长。

但是，这并不代表lazy枚举器的性能要好于『积极的』枚举器。事实上，lazy枚举器要比『积极的』枚举器慢几倍。

```ruby
require "benchmark"

N = 9_999_999

Benchmark.bm 10 do |bm|
  bm.report "Eager:" do
    (0..N).select(&:even?).map { |x| x * x }
  end
  bm.report "Lazy:" do
    (0..N).lazy.select(&:even?).map { |x| x * x }
  end
end
```

运行结果：
```
           user   system    total      real
Eager: 0.950000 0.040000 0.990000 (0.996567)
Lazy:  1.880000 0.000000 1.880000 (1.873170)
```

这主要是因为，惰性枚举器每枚举一个元素就迭代一次并等待迭代完才枚举下一个元素带来的开销很大，至少相比于划分中间结果集的内存来说，它的开销要大的多。

这类似于家具商店要运一大卡车的家具到客户家，尽管一次性装完需要使用大卡车(内存资源)，但是如果使用小三轮的话，则需要分多次运送，装一次、运一次，再装一次运一次，这种效率是很差的。

lazy枚举器真正的性能优势在于它通常只取前有限个枚举结果，枚举完这几个元素之后就不再枚举，由于它不再需要枚举所有元素，所以可能会减少大量时间。而『积极的』枚举器，尽管最终想要的结果是前几个，但它也会将所有元素枚举完。

例如，只取前10个结果的性能对比：
```ruby
require "benchmark"

N = 9999999

Benchmark.bm 10 do |bm|
  bm.report "Eager:" do
    (0..N).select(&:even?).map { |x| x * x }.take(10)
  end
  bm.report "Lazy:" do
    (0..N).lazy.select(&:even?).map { |x| x * x }.take(10).force
  end
end
```

运行结果：
```
           user   system    total      real
Eager: 0.640000 0.010000 0.650000 (0.652713)
Lazy:  0.000000 0.000000 0.000000 (0.000035)
```

所以，关于lazy枚举器的性能是好是坏，取决于它需要枚举的元素数量：
- 如果只需枚举前有限个(且不会非常多)时，lazy枚举器性能很好
- 如果需要枚举所有元素，或者枚举的元素很多时，lazy枚举器的性能很差

