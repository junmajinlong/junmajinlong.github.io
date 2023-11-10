---
title: Ruby语句块、Proc和Lambda
p: ruby/ruby_proc_block_lambda.md
date: 2020-05-14 14:20:27
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby可调用和可执行：语句块、Proc和Lambda

函数是可调用可执行的一段代码块。在Ruby中，除了函数之外，Proc对象和Lambda表达式也是可调用的。这些可调用对象在调用它们的call方法时，它们将被执行，也就成为了可执行的对象。

语句块看上去是可调用的，但严格地说语句块不是可调用的，因为在调用语句块的时候它会转成Proc对象，而Proc对象是可调用的。

## Proc对象

使用Proc.new()方法构建一个Proc对象，该Proc对象可以使用它的call方法来调用执行。

```ruby
p = Proc.new {puts "I am a Proc"}
p.call
```

## Proc对象和语句块的关系

每个Proc都需要有一个语句块，这个语句块就是这个Proc对象中要被执行的部分。

而语句块自身只是定义了要执行的内容，它自己并没有被调用和被执行的能力，它一定是作为Proc对象的一部分，要么是直接作为Proc对象的主体，要么是间接转换成Proc对象，然后被调用执行。

这就像是函数和函数的代码体关系一样，只不过函数体是必须依存在函数声明中的，而语句块则因为Ruby的独特性，可以脱离Proc，并在必要时间接地转换成Proc，从而让语句块的语法编码更加优雅。

换句话说，Proc对象和语句块之间是互相依存的：  
- 语句块定义了Proc对象要做的事  
- Proc对象要执行它绑定的语句块  

## 语句块和Proc对象的转换

![](/img/ruby/1589464908188.png)

所以，在声明方法时，`&`的作用是打包语句块成为Proc对象；在调用方法时，`&`的作用是解包Proc对象成为语句块。也可以看作是引用语句块和解除引用的关系。

事实上，`&`符号的作用就是调用to_proc方法将其转换成Proc对象，Proc对象是语句块的最终转换结果，所以`&`之后得到的其实都是Proc对象。

### 打包语句块：语句块转Proc对象

在声明方法时，可以在参数声明的最后一个位置使用`&blk`来将语句块打包成Proc对象blk。

```ruby
def func(a,b,&block)
  puts block.call(a,b)
end

func(2,3) { |x,y| x + y }  # 5
```

上面定义了一个方法func，其中最后一个参数是`&block`，这表示将调用func时的语句块`{ |x,y| x + y }`打包成Proc对象，这个对象中包含的语句块就是func方法后接的语句块，block是这个Proc对象的名称。然后`block.call`调用block这个Proc对象，同时还为其传递了两个参数，这两个参数会传递到语句块变量x和y中去。

从上面也可以看出，call方法可以传递参数给Proc对象，call方法也有返回值，它的返回值就是Proc对象的返回值。

### Proc对象转语句块

在方法调用时，使用`&`符号可以将Proc对象转换成语句块。

```ruby
def func(&block)
  puts "in func"
  block.call
  puts "exit func"
end

pr = Proc.new {puts "I am a Proc"}
func { puts "I am a block" }
func(&pr)
```

输出结果：
```
in func
I am a block
exit func
in func
I am a Proc
exit func
```

上面调用func方法时，通过语句块和通过`&pr`的结果是一样的，都是执行对应语句块。

注意，因为方法调用时`&pr`已经转换得到了一个语句块，所以这时不能再显式编写一个语句块，这样就有了两个语句块，这会报错。
```ruby
# 下面都是错误的
func(&pr) {puts "I am a block"}
[1,2,3].each(&pr) {puts "I am a block"}
```

正因为可以打包和解包语句块，使得在方法内部可以非常方便的把这个语句块传递给方法内部的其它方法。

例如，Ruby中定义递归方法时，如果这个方法包含了语句块，想要让方法内部调用的方法也有同样的语句块，只能打包外部语句块，并在调用时解包语句块。这样就能将最外层的语句块像传香火一样一层一层地传递进去。

```ruby
def recur(a, b, &block)
  return if a == 4
  puts "res: #{block.call(a, b)}"
  recur(a + 1, b + 1, &block)  # 传递语句块给内部方法
end

recur(1, 1) {|x, y| x * y}
```

结果：
```
res: 1
res: 4
res: 9
```

## Proc对象的call方法

call方法的作用是执行可调用对象的代码，比如Proc对象中的语句块代码：
1. 调用call时，可以指定参数，这些参数将传递给语句块变量
2. call方法有返回值，它的返回值就是语句块的返回值

call方法还有其它几种同义词。以下4种调用方式是等价的：
```ruby
# pr是一个Proc对象
pr[1,2,3]
pr.call(1,2,3)
pr.(1,2,3)
pr.yield(1,2,3)
```

例如：
```ruby
pr = Proc.new {|x, y, z| puts x + y + z}
pr[1, 2, 3]
pr.call(1, 2, 3)
pr.(1, 2, 3)
pr.yield(1, 2, 3)
```

注意其中的yield方法，它是Proc对象的方法而非保留关键字yield，但它们的目的是一样的，都是为了跳转过去执行语句块对象。

call方法传递参数时，如果和语句块变量的数量不一样，则以语句块中的变量数量为准。即如下规则：
1. call传递是参数少于语句块变量时，则语句块中多出的参数赋值为nil
2. call传递的参数多于语句块变量时，则多余的参数丢弃

例如：
```ruby
pr = Proc.new {|x| puts x}
pr.call("Hello", "World")  # World被丢弃
pr.call()     # 语句块变量x被设置为nil
```

最后注意，call方法或其它同义词是去执行可调用对象，Proc对象只是可调用对象的其中一种，还有特殊的Proc对象lambda、封装后的Method对象，都可以使用call或它的同义词来调用。

### 再理解迭代方法的语句块

有了语句块和Proc对象的基础知识后，再来理解each、map、select等迭代方法的语句块。

例如：
```ruby
(1..10).each {|x| puts x*2}
```
它在每次迭代元素的时候，都通过内部的yield将元素传递给语句块，而语句块自身是不可调用的，这实际上是跳转到语句块对应的Proc对象，并将元素作为参数传递给Proc对象中的语句块。

## &和to_proc

实际上，`&`的作用是调用对象的to_proc方法，to_proc方法内部返回一个Proc对象，从而无论是打包语句块还是解包语句块时都得到一个Proc对象。

下面一个示例，比较能说明问题：
```ruby
class Person
  attr_accessor :name

  def self.to_proc
    Proc.new {|person| puts person.name}
  end
end

p1, p2 = Person.new, Person.new
p1.name = "malongshuai"
p2.name = "gaoxiaofang"

[p1, p2].each(&Person)
```

这里对Person定义了一个类方法`to_proc`。当each方法被调用时，会将`&Person`通过调用Person的to_proc方法解包成一个Proc对象，然后将p1和p2这两个Person对象传递给Proc中的语句块，所以语句块变量person分别对应p1和p2，然后输出它们的name属性。

需要注意的就是这里的to_proc是Person的类方法，但是里面的person语句块变量却是Person的对象p1和p2，这是由each方法迭代的元素决定的。

一般情况下，是无需自己定义to_proc方法的，它们的使用场景并不多。但在Proc类和Symbol类中，都自带了to_proc方法，Proc中的to_proc自不必说，它就是返回Proc对象自身，而Symbol的to_proc则有用的多。此外，Method类也实现了to_proc方法，它也非常有用。

### Symbol的to_proc

Symbol中定义了一个to_proc方法，这使得可以使用类似于`&:downcase`、`&:+`的方式去替代语句块。不仅如此，由于所有方法的名称都是Symbol对象，所以可以对所有方法都采用这种方式代表语句块，这使得编码变得非常简洁。

例如：
```ruby
p (1..5).select(&:even?).map {|x| x * x}
p %w(Perl PHP Ruby).map(&:downcase)
p (1..5).inject(&:+)
```

结果：
```
[4, 16]
["perl", "php", "ruby"]
15
```

可以想想Symbol的to_proc方法是怎么实现的。假设想要实现的是只使用单个语句块变量的方法的to_proc，可以如下定义：
```ruby
class Symbol
  def to_proc
    Proc.new {|obj| obj.send(self) }
  end
end
```

下面是Rubinius中Symbol的to_proc定义：
```ruby
class Symbol
  def to_proc
    sym = self
    Proc.new do |*args, &b|
      raise ArgumentError, "no receiver given" if args.empty?
      args.shift.__send__(sym, *args, &b)
    end
  end
end
```

## Ruby匿名方法：Lambda

Kernel中的`lambda`方法或字面符号`->`可以定义一个特殊的Proc对象：
```ruby
lam = lambda {puts "I am a Lambda"}
lam.call

lam = -> {puts "I am a Lambda"}
lam.call
```

还可以在声明lambda时指定参数：注意两种构造方式的参数指定方式不一样。
```ruby
lam = lambda {|a,b| puts a+b }
lam = ->(a,b) {puts a+b}

# 指定参数b的默认值
lam = lambda {|a,b=1| puts a+b }
lam = ->(a,b=1) {puts a+b}
```

注意，并没有Lambda类，lambda方法返回的是一种特殊风格的Proc对象。它与普通的Proc对象有些不同：
1. lambda需要明确的创建过程，而Proc对象可以是直接Proc.new()创建的，也可以是语句块间接转换的。换句话说，所有隐式转换得到的Proc对象都是普通的Proc对象，不可能是lambda，这意味着方法调用时`&blk`的blk不能是一个lambda。
2. lambda是以Proc对象的方式体现为匿名方法。lambda中的return语句只会返回lambda自身，而Proc对象中的return语句会返回调用语句块所在的方法。
3. 参数处理不一样。lambda严格要求参数声明的参数数量和调用时的参数数量要一致。

其实第二点和第三点很容易理解，lambda就是一个匿名方法，它的返回方式和参数处理方式和方法一样。

例如，对于第二点：
```ruby
def return_test
  l = lambda {return}
  l.call # 退出lambda
  puts "Still here!"
  p = Proc.new {return}
  p.call # 退出return_test方法
  puts "no this message"
end

return_test
```

## 将方法转换成方法对象

方法自身不是对象，但可以转换成对象。将一个非对象转换成对象，这称为对象化（objectify）操作。

转换后的方法对象是Method类的实例。

```ruby
def func1
  puts "hello world"
end

meth = self.method(:func1)
meth.call
p meth     # #<Method: main.func1>
```

这里的meth就是一个Method对象，这个对象可以像Proc一样被调用。

注意上面调用的method()方法，它以`obj.method(Sym)`的方式将Sym对应的方法转换成一个方法对象。这个方法对象绑定在obj上，这使得以后调用meth都可以省略obj这个名称。

自定义一个类和一个实例方法来理解更容易：

```ruby
class C
  attr_accessor :name
  def talk
    puts "hello: #{self.name}"
  end
end

c = C.new
c.name = "cccc"
c.talk      # 通过对象名显式调用它的方法talk

meth = c.method(:talk)  # meth绑定在c上，对应c.talk
meth.call   # 通过Method对象的方式直接调用，其内部等价于c.talk
```

输出：
```
hello: cccc
```

obj.method()返回的方法对象meth是绑定在obj上的，所以通过meth.call执行时，talk内部的self也仍然对应于对象c。

所以，obj.method()的作用是将一个对象的方法封装成一个Method对象，这个Method对象中有它绑定的对象obj。

### 解绑和重绑

可以使用`unbound()`对一个Method对象进行解绑，然后再用`bind(obj1)`绑定到另一个对象obj1上，obj1可以是同类对象或子类对象。
```ruby
class C
  attr_accessor :name

  def talk
    puts "say hello: #{self.name}"
  end
end

c = C.new
d = C.new
c.name = "cccc"
d.name = "dddd"

c_meth = c.method(:talk) # 绑定在对象c上
p c_meth     # #<Method: C#talk>

unmeth = c_meth.unbind  # 从对象c上解绑
p unmeth     # #<UnboundMethod: C#talk>

d_meth = unmeth.bind(d)  # 绑定到对象d上
p d_meth     # #<Method: C#talk>
d_meth.call  # say hello: dddd
```

输出结果：
```
#<Method: C#talk>
#<UnboundMethod: C#talk>
#<Method: C#talk>
say hello: dddd
```

可以直接在类上使用`instance_method()`类方法返回该类的一个未绑定的Method对象。它等价于先绑定在一个对象上，再从这个对象上解绑：
```ruby
unmeth1 = C.instance_method(:talk)
p unmeth1    # #<UnboundMethod: C#talk>
```

### 调用指定父类的方法

得到一个未绑定的方法有时候是有奇效的。例如，在子类对象上直接调用某方法时，它只会执行第一个查找到的方法，如果想要明确执行父类或某个层次的祖先类中的同名方法，这时只能通过未绑定方法来实现。

例如：

```ruby
class A
  def talk
    puts "I am A"
  end
end

class B < A
  def talk
    puts "I am B"
  end
end

class C < B
end

c = C.new
c.talk  # 调用的父类B中的talk
```

这时想要通过对象c来执行祖先类A中的talk方法，可以获取祖先类A的一个未绑定talk方法，然后再将其绑定到c上。
```ruby
meth = A.instance_method(:talk)
meth.bind(c).call  # 输出"I am A"
```

如果想要在C类中定义一个调用父类的方法，可以将上面的过程封装到C类中：
```ruby
class C<B
  def call_super
    A.instance_method(:talk).bind(self).call
  end
end
```

### Method对象的to_proc效果

Method对象实现了to_proc，所以可以使用`&meth_obj`这种方式替代语句块进行迭代。

```ruby
[1, 2, 3].each(&method(:puts)) #=> prints 3 lines to stdout

out = File.open('test.txt', 'w')
[1, 2, 3].each(&out.method(:puts)) #=> prints 3 lines to file

require 'date'
%w[2017-03-01 2017-03-02].collect(&Date.method(:parse))
#=> [#<Date: 2017-03-01 ((2457814j,0s,0n),+0s,2299161j)>,
     #<Date: 2017-03-02 ((2457815j,0s,0n),+0s,2299161j)>]
```


## Ruby闭包

由于Ruby自身语法特性，允许方法可以不带括号直接调用，这使得无法在方法内部返回方法。再加上Ruby方法内无法访问方法外变量，使得Ruby中的方法不是一等公民，无法使用方法来实现闭包。

Ruby中的一等公民是Proc对象，所以要实现闭包，需使用Ruby Proc或lambda来实现。

例如，按照其它编程语言的编码方式，下面本是想在func1中返回func2方法，但实际上却是将func2执行后的返回值给返回了。
```ruby
def func1()
  def func2()
    puts "hello"
  end
  return func2  # 这里的func2被认为是方法调用
end
```

所以，要在Ruby中定义闭包，要么在方法内部返回Proc对象、要么返回lambda、要么将方法转换成方法对象后返回。但返回转换后的方法对象是理论上的，实际上并不可行，因为def中嵌套def后，内部的def无法访问外部def的局部变量。

例如：
```ruby
def test_by_Proc(x)
  return Proc.new {|y| x + y}
end

def test_by_lambda(x)
  return ->(y) {x + y}
end

# 下面是错误的
def test_by_method(x)
  def closure(y)
    x + y   # 内部def不能访问外部def的局部变量x
  end
  return self.method(:closure)
end
```

注意，返回闭包后(即Proc对象)，通过call方法或其它同义词方法进行调用执行。例如下面是等价的：
```ruby
puts test_by_Proc(3).call(4)
puts test_by_Proc(3)[4]
puts test_by_Proc(3).(4)
puts test_by_Proc(3).yield(4)
```

最后还需注意的是Ruby中的变量作用域问题。Ruby采用的是词法作用域，它定义于何处，将见到何处的变量，这与它的调用位置无关。
```ruby
def func1(pr)
  a = "hello"
  puts a
  pr.call
end

a = "world"
pr = Proc.new {puts a} # 记住的是上一行的a
a = "WORLD"  # 但是将它记住的a修改为WORLD
pr.call
func1(pr)
```

输出结果：
```
WORLD
hello
WORLD
```
