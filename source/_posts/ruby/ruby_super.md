---
title: Ruby中的super
p: ruby/ruby_super.md
date: 2020-05-25 09:37:33
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby中的super

子类继承父类时，如果想要调用父类中(严格地说是继承链中)的同名方法，可使用`super`。super有几种用法：  

![](/img/ruby/1590385234172.png)

需注意：  
- super关键字和super()方法都会将调用父类的返回结果作为它们的返回值，这一点特性在实现ruby的装饰器可能可以排上用场  
- super也适用于模块(模块和类很多时候是相同作用的)，比如include模块时，super将会找到模块中同名方法，perpend模块时，模块中的方法使用的super将会找到类中的同名方法，具体可参见[Ruby include、prepend和extend用法分析](/ruby/ruby_include_prepend_extend)  

使用super关键字时，方法在被调用时传递了什么参数，就会直接原样转移给继承链中找到的同名方法。

```ruby
class C
  def f(msg)
    puts "in f"
  end
end

class D < C
  def f(msg)
    super      # 将调用父类的f("hello")
  end
end

D.new.f "hello"
```

super关键字还可以传递语句块：

```ruby
class C
  def f()
    yield
  end
end

class D < C
  def f()
    super
  end
end

D.new.f {p "hello world"}
```

如果把父类中的f参数去掉，将报错：

```ruby
class C
  def f        # 没有参数，但是被传递了参数
    puts "in f"
  end
end

class D < C
  def f(msg)
    super      # 将调用父类的f("hello")
  end
end

D.new.f "hello"  # ArgumentError (given 1, expected 0)
```

如果想指定传递给父类同名方法的参数，则使用super()，而非关键字super。

```ruby
class C
  def f
    puts "in f"
  end
  
  def g(msg)
    puts "in g: #{msg}"
  end
end

class D < C
  def f(msg)
    super()      # 调用父类f方法时不传递任何参数
  end

  # 调用父类方法时，只传递msg参数
  def g(msg, level = 0)
    super(msg) if level != 0
  end
end

D.new.f "hello"
D.new.g "world", 1
```



