---
title: Ruby eval, class_eval, instance_eval
p: ruby/ruby_eval.md
date: 2020-05-21 19:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby eval简单用法

```
eval(string)
```

![](/img/ruby/1590064449838.png)

例如：

```ruby
str="hello"
eval 'str'               # 返回"hello"
eval 'p str'             # 输出"hello"，返回hello(p()返回输出的内容)

eval '"hello"'           # 返回"hello"，双引号不能省略
eval 'puts "hello"'      # 输出"hello"，返回nil
eval('p "hello"') + "!"  # 输出"hello"，返回"hello!"
puts eval("3+4") + 2     # 输出9
puts eval('"3+4"') + 2   # 报错，类型转换错误
```

需注意，eval有自己的作用域环境，它相当于在一个语句块或Proc上下文，参考本文结尾的深入Ruby eval。

# class_eval和instance_eval的用法

除了eval，Ruby还有`class_eval`(还有等价的module_eval)和`instance_eval`。用法为：

```
# C或i是方法的receiver
C.class_eval
i.instance_eval
```

- class_eval：在指定的接收者C的上下文中执行class_eval中定义的相关代码，C是类名或模块名(类也是模块)  
- instance_eval：在指定的接收者i的上下文中执行instance_eval中定义的相关代码，i是实例对象名，注意，自定义类自身也是Class类的实例对象  

使用这两个方法的优势在于接收者`C`或`i`允许是变量形式，从而动态地定义代码。

例如：

```ruby
class A;end

# 与下面等价代码：
# A.class_eval "def m1;p 'm1' end; def self.m2;p 'm2' end"
A.class_eval do
  def m1           # 定义的是实例方法
    p 'm1'
  end
  
  def self.m2      # 定义的是类方法
    p 'm2'
  end
end

#A.m1       # 报错
A.new.m1    # 输出'm1'

A.m2        # 输出'm2'
A.new.m2    # 报错
```

`A.class_eval`表示在类A的上下文中执行指定的代码(即定义m1方法和m2方法)，因为上下文是类A，所以相当于：

```ruby
class A
  def m1
    p 'm1'
  end
  
  def self.m2
    p 'm2'
  end
end
```

再例如instance_eval：

```ruby
class A;end

A.instance_eval do
  def m1
    p 'm1'
  end
end

A.m1
# A.new.m1    # 报错

a_obj = A.new
a_obj.instance_eval do
  def m2
    p 'm2'
  end
end

a_obj.m2
```

`A.instance_eval`中，因为A是自定义的类名，自定义类都是Class类的实例对象，所以`A.instance_eval`表示在类A这个类实例中执行相关代码。所以，上面的m1是A的类方法，而不是实例方法。

`a_obj.instance_eval`则表示在a_obj实例中执行相关代码，所以m2是a_obj的单例方法。

因为`class_eval`和`instance_eval`中执行的代码分别绑定在各自的上下文中，所以在各自的上下文中可以访问一些属于该上下文的属性。

例如：

```ruby
class A
  def initialize
    @x = 44
  end
end

A.class_eval do
  def m1
    p @x
  end
end

A.new.m1

a = A.new
p a.instance_eval "@x"       # 直接在实例上下文中评估实例变量
```

# 深入Ruby eval

当eval只有一个参数时，默认在『全局』上下文解析变量、方法等。但实际上，**eval有自己的作用域环境，它相当于在一个语句块或Proc上下文**。因此，eval中可以访问或修改上层变量，但如果在eval中定义局部变量，eval退出后局部变量将消失。

```ruby
a=3
eval ("a = 4")  # 可访问或修改上层变量
puts a     #=> 4

eval ("n = 'junmajinlong'")
puts n   # 报错，n是eval中的局部变量，eval退出后就消失

nn = eval("n = 'junmajinlong'")  # 可返回
puts nn  # junmajinlong
```

可以为eval其指定第二个参数，使得string参数代表的Ruby Code在该参数对应的上下文中执行。换言之，eval将在指定的上下文中解析执行Ruby Code。

第二个参数是一个特殊参数：Binding对象，一般通过`Kernel#binding`方法返回。**Binding对象是一个绑定了指定上下文(作用域)的特殊对象**，相当于封装了一个上下文对应的路径查找表(lookup table)，通过Binding对象，可以找到其所绑定上下文中的属性，包括变量、方法等。

例如：方法m定义了局部变量a和b，同时返回一个Proc对象。

```ruby
def m
  a = 3
  b = 4
  Proc.new { c=5 }
end
p eval("a * b", m.binding)    # 输出12
```

注意，返回的这个Proc能够访问a和b和c变量，**但是a和b变量在定义Proc时就已经能访问，而c变量只有在调用proc时才能访问**(即类似于编译期间和运行期间的区别)。

所以，该Proc对象的上下文中，除了继承自Object而来的属性外，还有a和b两个变量，且在其执行时，其上下文还包括c变量。

所以，m.binding封装了m所返回的Proc对象的上下文环境，即包含变量a和b但不包含变量c的环境。将其指定为eval的第二个参数，使得eval第一个参数对应的代码中可以访问a、b两个变量。

如果在上面eval第一个参数中使用变量c，则报错：

```ruby
p eval("c * a * b", m.binding)
#=> undefined local variable or method `c' for main
```

如果在函数内执行binding()返回Binding对象，则该Binding封装的是该函数的上下文：

```ruby
def m
  a = 3
  b = 4
  c = Proc.new { a * b }
  binding
end

# 这里m返回的是函数的上下文
eval("p c.call", m)   #=> 12


############## 与下面的比较 ############
def m
  a = 3
  b = 4
  Proc.new { a * b}
end

# 这里m.binding是调用m后返回的Proc对象的上下文
eval("p c.call", m.binding)  #=> 12
```

可以在eval中修改Binding对象中封装的变量：

```ruby
def m
  a = 3
  Proc.new {a * a}
end

pc = m  # pc是一个Proc对象

# pc.binding封装的是Proc对象上下文
eval("a=4", pc.binding)  # a=4意味着在Proc内修改了上层变量
p pc.call     # 16
```

最后简单分析一下eval官方手册给的示例：

```ruby
def get_binding(str)
  return binding
end

str = "hello"
eval "str + ' Fred'"     #=> "hello Fred"
eval "str + ' Fred'", get_binding("bye")  #=> "bye Fred"
```

上面的`get_binding()`返回的是函数`get_binding`的上下文，该上下文中包含了参数str对应的局部变量，所以在eval第一个参数字符串中可以使用该局部变量。

