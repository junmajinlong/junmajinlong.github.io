---
title: Ruby方法的定义和调用
p: ruby/ruby_methods.md
date: 2021-05-10 22:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby方法的定义和调用

Ruby中使用def...end语句块定义一个方法：

```ruby
def say_hello
  puts "hello"
end
```

方法名称后的括号可省略，下面两种定义方式等价：

```ruby
def add x, y
  x + y
end

def add(x, y)
  x + y
end
```

可以使用return指定方法的返回值，也可以省略return，此时方法中最后一条被执行的语句的评估结果将作为方法的返回值。下面两种定义方式在效果上等价：

```ruby
def add x, y
  puts "add: x + y"
  return x + y
end

def add x, y
  puts "add: x + y"
  x + y
end
```

定义方法时，可以同时指定如何处理方法体中抛出的异常：

```ruby
def abc
  ...
rescue
  ...
end

def abc
  ...
ensure
  ...
end

def my_method
  ...
rescue
  ...
else
  ...
ensure
  ...
end
```

调用方法时，多数时候可以省略括号，但如果有产生歧义的行为时(例如传递hash作为参数时，不加括号将认为大括号是语句块的大括号)，应带上括号。

```ruby
abc(x, y)
abc x, y
abc({a: 3, b: 4})  # hash参数
abc {}             # 语句块大括号
```

很多时候，以`obj.meth()`方式调用某方法时，它的接收者obj可能不存在或为nil，此时将报错提示NilClass未定义方法meth。Ruby提供了一种安全调用方法的方式，即使用`&.`取代`.`方式调用方法，例如`obj&.meth()`。

`&.`被成为安全导航符(safe navigation operator)，它表示，如果obj存在(不为nil)，则调用它的方法meth()，否则不调用meth()，并且直接返回nil。因此，下面两种写法在行为上等价：

```ruby
obj && obj.meth()
obj&.meth()
```

## 参数默认值

通过如下方式可为参数指定默认值：

```ruby
def add_values(a = 1, b = 2, c)
  a + b + c
end
```

Ruby中参数评估的方式是从左至右，而不是同时评估，因此右边的默认值可以使用左边的变量：

```ruby
def add_values(a = 1, b = a)
  a + b
end
add_values      #=> 2
```

如果有多个参数带有默认值，这些参数需放在连续的参数位置，而不能分开在不连续的参数位置，因为在调用方法时，不连续的默认值参数会产生歧义：

```ruby
# 下面的定义正确
def add_values(a = 1, b = 2, c)
  a + b + c
end

# 下面的定义报错
def add_values(a = 1, b, c = 1)
  a + b + c
end
# add_values(3, 4) # 这两个参数被传递给哪个形参呢？
```

## 参数解构：数组参数

调用方法时，有时候会直接传递一个数组参数，如果想要分解这个数组得到它的各元素，可直接在参数定义上完成这项分解任务。使用一个额外的小括号，即可解构传递过来的数组结构：

```ruby
def m((a, b), c)
  puts a: a, b: b, c: c
end
m([11, 22], 33)   # {:a=>11, :b=>22, :c=>33}
```

上面参数部分的小括号`(a, b)`结构了方法调用时传递的数组参数`[11,22]`。

如果传递的数组元素比小括号中定义的的变量更多，则多余的元素被忽略，如果更少，则多出的变量被赋值为nil：

```ruby
def m((a, b), c)
  puts a: a, b: b, c: c
end
m([11, 22, 222], 33)   # {:a=>11, :b=>22, :c=>33}

def add((a, b, c), d)
  puts a: a, b: b, c: c, d: d
end
add([11, 22], 33)   # {:a=>11, :b=>22, :c=>nil, :d=>33}
```

如果数组元素更多，也可以使用`*`来收集解构数组时剩余的元素：

```ruby
def m((a, *b), c)
  puts a: a, b: b, c: c
end
m([11, 22, 222], 33)   # {:a=>11, :b=>[22, 222], :c=>33}
```

## 数组参数(array argument)

定义方法时，可使用`*`来收集一定数量的参数到指定的数组中：

```ruby
def a(*args)
  p args
end
a 1, 2, 3 # [1, 2, 3]
```

数组参数也可以在形参的开头或中间：

```ruby
def a(x, *y, z)
  p y
end
a 1, 2, 3, 4 # [2, 3]

def a(*x, y, z)
  p x
end
a 1, 2, 3, 4 # [1, 2]
```

需注意，数组参数的定义必须在关键字参数之前。

另外，如果调用方法时在最后一个位置传递了hash，则数组参数可能会收集到这个hash参数，应小心：

```ruby
def abc(*args)
  p args
end
abc 1, a: 2  # [1, {:a=>2}]
```

可单独使用不带任何名称的`*`来忽略特定数量的参数：

```ruby
def ignore_arguments(*)
end
```

## 关键字参数(keyword argument)

关键字参数类似于默认值参数：

```ruby
def add(first: 1, second: 2)
  first + second
end
```

当然，关键字参数也可以不带默认值，调用方法时，必须为没有默认值的关键字参数传递参数值：

```ruby
def add_values(first:, second:)
  first + second
end
add_values  # ArgumentError
add_values(first: 1, second: 2) # => 3
```

可使用`**`收集剩余的关键字参数：

```ruby
def gather_arguments(first: nil, **rest)
  p first, rest
end

gather_arguments first: 1, second: 2, third: 3
# 1, {:second=>2, :third=>3}
```

关键字参数必须定义在位置参数之后，这是因为调用方法时，Ruby无法处理在非尾部位置的关键字参数：

```ruby
def abc(a, b:, c:)
  p a: a, b: b, c: c
end
abc 1, b: 2, c: 3   # {:a=>1, :b=>2, :c=>3}

# 错误
def abc(a, b:, c:, d)
  p a: a, b: b, c: c
end
abc 1, b: 2, c: 3, 4
```

可以单独使用没有名称的`**`忽略方法调用时传递的关键字参数：

```ruby
def ignore_keywords(**)
end
```

## Ruby 3的位置参数和关键字参数

在Ruby 3之前，调用方法时如果传递了关键字参数(关键字参数只能在最后位置)，其实质是传递了一个hash参数：

```ruby
def abc(x)
  p x
end
abc b: 2, c: 3  # {:b=>2, :c=>3}
```

上面调用方法时传递的关键字参数，等价于下面传递hash参数的方式：

```ruby
def abc(x)
  p x
end
abc({b: 2, c: 3})  # {:b=>2, :c=>3}
```

注意上面调用方法时的小括号必须不能省略，否则hash参数的大括号会被误认为是语句块的大括号。

换言之，在Ruby 3之前，所谓的关键字参数其本质仍然是位置参数，它是最后一个位置参数，是一个hash参数。

在Ruby 3中，提供了真正的关键字参数，并且在一定程度上保留了和Ruby 2的参数处理方式的兼容性。

在Ruby 3中，如果未在方法定义中定义关键字参数，则方法调用时传递的关键字参数，处理方式和之前的版本(Ruby 2)一样，即当作一个hash参数。

如果在方法定义中定义了关键字参数，则方法调用时传递的关键字参数会与方法定义中的关键字参数进行配对处理。**此时方法调用时在最后一个参数位置传递hash参数将仍然是当作单个hash位置参数，不会展开成为关键字参数**。

```ruby
# 下面代码在ruby 3中报错，但在ruby 3之前的版本可正确执行
def abc(x, a:, b:)
  p x: x, a: a, b: b
end

abc(1, {a: 2, b: 3})
```

官方说明：[Separation of positional and keyword arguments in Ruby 3.0](https://www.ruby-lang.org/en/news/2019/12/12/separation-of-positional-and-keyword-arguments-in-ruby-3-0/)。

## 语句块参数

方法定义时，最后一个位置且必须是最后一个位置处可以使用`&`前缀的形参：

```ruby
def my_method(&my_block)
  my_block.call(self)
end

my_method {|x| puts "hello world: #{x}" }
```

形参中的`&`前缀表示将调用方法时传递的语句块转换为Proc对象my_block。

`&`前缀除了将语句块转换为Proc对象，也有将Proc对象展开为语句块的功能。这使得可以直接将Proc对象作为实参传递给某个方法，相当于为该方法指定了对应的语句块：

```ruby
def each_item(&block)  # block是一个Proc对象
  @items.each(&block)  # 将block展开为each的语句块
end
```

上述介绍中是将方法的语句块转换为Proc对象并在方法体中执行Proc对象，与之功能相似的是，直接在方法体中使用yield去执行该方法的语句块，而无需通过`&block`参数将语句块转换为Proc对象再执行：

```ruby
def my_method
  yield self
end
```

## 参数转发(argument forwarding)

Ruby 2.7引入了一种新的参数语法`...`，它可用于转发参数。

例如：

```ruby
def wrapper(...)
  meth(...)
end
```

这表示将wrapper接收到的所有参数，原封不动的传递给meth方法。

在Ruby 3中进一步改进了参数转发的功能，允许混用`...`和具体的形参：

```ruby
def wrapper(name, ...)
  if(name == "junmajinlong")
    meth(...)
  end
end
```