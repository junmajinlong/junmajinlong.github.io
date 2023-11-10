---
title: Ruby的self含义解释
p: ruby/ruby_self.md
date: 2020-05-22 19:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby的self含义解释

在类定义中、模块定义中，可以使用`self`来表示当前的对象。在不同上下文中，『当前』的概念不同：   

![](/img/ruby/1590229304564.png)

```ruby
# 顶层空间的self
puts "self in Top-Level: #{self}"     # self == main

module M1
  # 模块上下文的self
  puts "self in M1: #{self}"      # self == M1
  
  # 模块方法中的self
  def self.m
    puts "self in M1.m: #{self}"  # self == M1
  end
  
  # 被Mix-in时的方法内的self
  # 等价于实例方法中的self
  def mf1
    puts "self in mf1: #{self}"   # self == <caller_obj>
  end
  
  # 嵌套模块或嵌套类上下文的self
  class C
    puts "self in M1::C: #{self}" # self == M1::C
  end
end

M1.m

class Cls
  # 类上下文的self
  puts "self in Cls: #{self}"     # self == Cls
  
  include M1

  # 实例中的self
  def f
    puts "self in f: #{self}"     # self == <caller_obj>
  end

  # 嵌套类上下文的self
  # 和嵌套模块的self一样
  class Cls1
    puts "self in Cls1: #{self}"  # self == Cls::Cls1
  end
end

Cls.new.f
Cls.new.mf1
```

因为类上下文和模块上下文的self都代表自身(类和模块自身也是对象)，所以经常见到下面两种方式定义类方法：

```ruby
class C
  # self.Method定义类方法
  def self.f1
    puts "class method: f1"
  end
  
  # 第二种定义类方法的方式
  class << self
    def f2
      puts "class method: f2"
    end
  end
end
```

在Ruby中，`def x.y`和`class << x`是等价的，都表示打开对象x的单例类空间，进入该单例空间的上下文。所以，上面的示例是打开C对象(类也是对象)的单例类上下文，并在单例类中定义方法f1和f2。

换句话说，**Ruby中的类方法是通过类的单例类实现的**。

```ruby
p C.singleton_methods()
# [:f1, :f2]
```

如果打开C的单例类后，在单例类中继续使用`def self.f3`来定义方法f3，因为此时的`def self.`又打开了一层单例类，即C的单例类的单例类，所以方法f3定义在C的单例类的单例类中，它不再是C的类方法，而是C的单例类的类方法。如果要调用f3，需`C.singleton_class.f3`：

```ruby
class C
  class << self
    def self.f3
      puts "class method: f3"
    end
  end
end

C.singleton_class.f3
```

## 无点引用时省略的self

**在对象内部，所有『无点引用』的方式其实都是省略了self的引用**：  
- 如果无点引用一个方法时省略了方法调用的括号(如`name`)，如果正好又存在一个同名局部变量name，则局部变量优先，如果此时要访问同名方法而非局部变量，则加上括号name()或self.name   
- 如果无点引用后有一个`=`，要注意它可能是在创建局部变量，而非访问setter方法  

```ruby
class C
  attr_reader :name, :age

  def initialize name, age
    @name = name
    @age = age
  end

  def f
    puts "in f"
    # name = "gaoxiaofang" # 取消该注释，下行将访问该局部变量
    puts "i am #{name}"    # 无点引用，等价于self.name
    # age = 23             # 无点引用，但这是创建局部变量age
    # self.age = 23        # 这是在访问age的setter方法
  end

  def g
    puts "in g"
    f       # 无点引用，等价于self.f
  end
end

C.new("junmajinlong", 23).g
```