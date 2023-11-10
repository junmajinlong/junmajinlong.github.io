---
title: Ruby Class类和对象创建的过程
p: ruby/ruby_new_allocate_initialize.md
date: 2020-05-26 13:57:33
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby Class类和创建对象的过程

在Ruby中，所有的类都是Class类的实例对象，Class类中定义了两个new方法，一个是类方法new，一个是实例方法new：

![](/img/ruby/1590542460765.png)

## Class.new创建类

类方法new很简单，它返回一个新建的类，所以可赋值给任何一个变量名作为类名，即使是小写字母开头的非常量。因为常规类的类名都是常量，所以将返回值赋值给一个大写字母开头的变量，那么和常规类没有区别。

```ruby
new(super_class=Object) → a_class
new(super_class=Object) { |mod| ... } → a_class
```

例如：

```ruby
class C;end

cc = Class.new(C) do     # 类名cc，继承类C
  def self.f1            # 定义类方法f1
    puts "in self.f1"
  end
  def f2              # 定义实例方法f2
    puts "in f2"
  end
end

cc.f1
cc_obj = cc.new
cc_obj.f2
```

## new、allocate和initialize创建对象

Class的new实例方法用于创建对象，它会先调用allocate()创建对象(创建的是空对象，会为此对象分配内存)，allocate()返回后，new()会继续调用initialize()方法为对象做初始化操作，它会将new接收到的参数完整地传递给initialize()。

虽然不建议修改Class的new实例方法，因为对它的修改是牵一发而动全身，但它确实是可以被自定义的。官方给了一个示例：

```ruby
class Class
  alias old_new new
  def new(*args)
    print "Creating a new ", self.name, "\n"
    old_new(*args)
  end
end

class Name
end

n = Name.new
```

此外，也可以手动单独使用allocate()来创建空对象，从而绕过initialize()。

```ruby
class C
  def initialize(name,age)
    @name = name
    @age = age
    @initialized = true
  end
  def initialize?
    @initialized || false
  end
end

c1 = C.new("junmajinlong", 23)
p c1.initialize?      # true
c2 = C.allocate
p c2.initialize?      # false

# 为空对象设置实例变量
c2.instance_variable_set "@name", "gaoxiaofang"
c2.instance_variable_set "@gender", "male"
p c2.instance_variable_get "@gender"
```