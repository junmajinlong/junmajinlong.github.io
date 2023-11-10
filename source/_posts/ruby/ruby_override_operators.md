---
title: Ruby重写操作符
p: ruby/ruby_override_operators.md
date: 2020-05-27 12:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby重写操作符

可以重写`+ - * / ~`等运算符，其实这些在Ruby中都是方法而不是操作符。

两个注意点：  

![](/img/ruby/1590573077744.png)

相关说明参考：<https://www.ruby-lang.org/en/documentation/faq/7/>。

例如，定义一个点Point，并提供点的`+ - * -@`方法：

```ruby
class Point
  attr_accessor :x, :y

  def initialize x, y
    @x, @y = x, y
  end

  def +(other)
    self.class.new(@x + other.x, @y + other.y)
  end

  def -(other)
    self.class.new(@x - other.x, @y - other.y)
  end

  def -@
    self.class.new(-@x, -@y)
  end

  def *(scalar)
    self.class.new(@x * scalar, @y * scalar)
  end
end

p1 = Point.new(1,2)
p2 = Point.new(3,4)

p p1 * 2 + p2
p -p1           # 将调用 -@ 方法
p p1 - p2
```

上面没有做任何类型检测，因为Ruby在运算过程中出错时自己也会抛出类型错误。

但是，如果想要定义自己的错误以便知道错在何方而不是使用千篇一律的类型错误。那么也可以做类型检测。

以加法运算为例：

```ruby
def +(other)
  self.class.new(@x + other.x, @y + other.y)
rescue
  raise TypeError, "argument does not quack like a Point"
end
```

另外，上面的`*`运算只支持`p * 2`，不支持`2 * p`，因为`2 * p`是调用数值2的方法`*`，但p此时不是数值。

```ruby
p 2 * p1
# 报错：Point can't be coerced into Integer (TypeError)
```

为了也支持`2 * p`，Point显然要定义`coerce()`，这里很简单，直接转换顺序即可，这样会重新执行`p * 2`。

```ruby
def coerce(other)
  [self, other]
end
```

