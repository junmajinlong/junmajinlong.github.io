---
title: Ruby定义方法、移除方法和它们的回调
p: ruby/ruby_define_remove_method.md
date: 2020-05-31 11:13:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby定义方法、移除方法和它们的回调

## define_method

一般定义方法的方式是使用`def`保留关键字，也可以使用eval或`define_method`方法来动态定义方法。

`define_method`和private等方法用法类似，都是在类上下文中使用，但操作的却是实例方法。

```ruby
class C
  define_method(:f1) {|name| puts "who am i: #{name}"}
end
C.new.f1("junmajinlong")
```

可以通过方法创建方法(委托给另一个方法来调用)：

```ruby
class C
  def create_method(name, &block)
    self.class.define_method(name, &block)
  end
end

a = C.new
a.create_method(:f1) {|name| puts "who am i: #{name}"}
a.f1("junmajinlong")
```

可以动态定义方法：

```ruby
a = 80
if a > 60
  class C
    define_method(:good) {puts "good"}
  end
else
  class C
    define_method(:bad) {puts "bad"}
  end
end

C.new.good
#C.new.bad    # 将报错
```

如果想要访问外面的局部变量，则采用class_eval：

```ruby
class C;end

a = 80
if a > 60
  C.class_eval do
    define_method(:good){puts "score: #{a}"}
  end
else
  C.class_eval do
    define_method(:bad){puts "score: #{a}"}
  end
end

C.new.good     # 80
#C.new.bad      # 报错
```

也可以使用`define_method`定义类方法：

```ruby
class C
  class << self
    define_method(:f) {puts "hello"}
  end
end

C.f
```

如果是通过方法委托的方式调用`define_method`来创建类方法，则要复杂一点。

```ruby
class C
  def self.create_class_method(name, &block)
    # 方法内部的self是对象自身，而不是单例类
    self.singleton_class.define_method(name, &block)
  end
end
C.create_class_method(:f) {puts "class method"}
C.f
```

虽然单例类空间中的self是单例类，但是单例类空间中方法内部的self却是对象自身。所以，要为类C(是一个对象)定义实例方法(即类方法)，需要进入类C的单例类。

## remove_method和undef_method

![](/img/ruby/1590895875621.png)

```ruby
class Parent
  def hello
    puts "In parent"
  end
end
class Child < Parent
  def hello
    puts "In child"
  end
end

c = Child.new
c.hello         # 可执行，输出In child

class Child
  remove_method :hello  # remove from child, still in parent
end
c.hello      # 可以执行，输出In parent

class Child
  undef_method :hello   # prevent any calls to 'hello'
end
c.hello    # 报错(NoMethodError): undefined method `hello'
```

当某方法被移除后，它将在方法列表中被移除。

```ruby
class C
  def f;end
end

p C.new.public_methods(false)   # [:f]

class C
  remove_method :f
end
p C.new.public_methods(false)   # []
```

## 定义方法和移除方法的回调函数

当def或define_method定义方法、remove_method或undef_method移除方法时，Ruby提供了对应的回调函数，它们将在方法被定义或方法被移除时自动被调用。

```
# 来自于Module
method_added 
method_removed(method_name)
method_undefined(p1)

# 来自于BasicObject
singleton_method_added 
singleton_method_removed(method_name)
singleton_method_undefined(p1)
```

对于非单例方法版本的回调函数，以`method_added`为例：

```ruby
class C
  def self.method_added(meth_name)
    puts "adding method: #{meth_name}"
  end

  def f1;end      # 触发回调，输出adding method: f1
  def self.f2;end # 不触发上述回调
end
```

对于单例方法版本的回调函数，以`singleton_method_added`为例：

```ruby
class C
  def self.method_added(meth_name)
    puts "adding method: #{meth_name}"
  end

  def f1;end
end

c1 = C.new
class << c1
  def singleton_method_added(meth_name)
    puts "adding singleton method: #{meth_name}"
  end
end

def c1.f2;end
=begin
adding method: f1
adding singleton method: singleton_method_added
adding singleton method: f2
=end
```

如果想要监视类方法的定义和移除行为，因类方法定义在类对象的单例类空间中，所以需在类对象的单例类空间中定义回调，即使用`singleton_method_xxx`。

```ruby
class C
  class << self
    def singleton_method_added(meth_name)
      puts "adding class method: #{meth_name}"
    end
  end

  def self.f2;end
end

=begin
adding class method: singleton_method_added
adding class method: f2
=end
```







