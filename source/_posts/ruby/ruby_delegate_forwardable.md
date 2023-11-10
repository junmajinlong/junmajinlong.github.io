---
title: Ruby中的委托和方法转发
p: ruby/ruby_delegate_forwardable.md
date: 2020-06-01 09:37:33
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby中的委托和方法转发

## 理解继承、组合和委托关系

参考[继承、组合和委托](/coding/inherit_composition_delegate)。

## Ruby委托的几种实现方式

Ruby中：  

![](/img/ruby/1590998260288.png)

当然，对于功能比较简单的委托，也可以手动编写代理方法。

## 手动实现组合和委托的代码

以房子和厨房、浴室的关系为例来描述组合和委托。

先实现组合：

```ruby
class Kitchen
  def cooking
    puts "Cooking in kitchen"
  end
end

class Bathroom
  def wash
    puts "Washing in bathroom"
  end
end

class House
  # 在创建House对象时，也同时创建组件对象
  # 从而实现组合：house对象和组件对象同生共死
  def initialize
    @kitchen = Kitchen.new
    @bathroom = Bathroom.new
  end
  
  def my_kitchen
    puts @kitchen
  end
  
  def my_bathroom
    puts @bathroom
  end
end
```

上面的代码实现了组合，但是下面的方法调用将报错，因为House的对象并没有cooking()和wash()方法：

```ruby
h = House.new
h.cooking   # 报错
h.wash      # 报错
```

如果要允许House的对象调用这两个方法，需要用到方法委托或方法转发，其实只是称呼高大上了一些，本质就是方法调用而已：

```ruby
class House
  def initialize
    @kitchen = Kitchen.new
    @bathroom = Bathroom.new
  end
  
  # 将cooking()方法的调用请求转发给组件对象的cooking()
  def cooking
    @kitchen.cooking
  end
  
  # 将wash()方法的调用请求转发给组件对象的wash()
  def wash
    @bathroom.wash
  end  
  
  def my_kitchen; puts @kitchen end
  def my_bathroom; puts @bathroom end
end
```

再来一个组合和委托的示例：电脑和内存。内存是电脑的一个部件，内存对象和电脑对象共存亡，且让电脑读、写内存，本质是让内存进行读写操作。

```ruby
class Memory
  def initialize
    @data = []
  end
  def write(data)
    @data << data
  end
  def read(index)
    @data[index]
  end
end

class Computer
  # 将一个Memory对象组合到电脑对象中
  def initialize
    @memory = Memory.new
  end
  
  # write方法委托
  def write(data)
    @memory.write(data)
  end
  
  # read方法委托
  def read(index)
    @memory.read(index)
  end
end
```

## Delegate的SimpleDelegator委托

Delegate有三种委托方式，其中最简单的一种是SimpleDelegator，它直接指定一个对象做为受托对象，只要是受托对象支持的方法，委托者也都能使用。另外，SimpleDelegator对象任何时候(中途)都可以使用`__setobj__(obj)`更改受托对象。

例如：

```ruby
class Memory
  def initialize
    @data = []
  end
  def write(data)
    @data << data
  end
  def read(index)
    @data[index]
  end
end

class Computer < SimpleDelegator
end
```

现在Computer是SimpleDelegator的子类，所以Computer的实例对象也是SimpleDelegator的实例对象，所以Computer的实例对象可以指定受托对象。

```ruby
comp = Computer.new(Memory.new)
```

这表示`Memory.new`对象是实例对象comp的受托方，只要是Memory实例对象支持的实例方法，都可以通过comp调用。

```ruby
comp.write("hello")
comp.write("world")
puts comp.read(0)    # 输出"hello"
puts comp.read(1)    # 输出"world"
```

这样似乎和继承实现了相同的功能。

还可以在实例方法内部进行委托(通过`self.meth`的方式)，这就是所谓的**装饰器模式**。比如，单独在Computer中定义一个实例方法，读完之后输出一点信息：

```ruby
class Computer < SimpleDelegator
  def read1(index)
    puts "read start..."
    read(index)
  end
end

puts comp.read1(0)
```

之所以可以在方法内进行委托，是因为其内部的read()等价于`self.read()`，`self.read()`自然会委托给受托方。

> **装饰器模式和代理模式**
>
> 设计模式中的代理模式和装饰器模式，都和委托相关。对于委托者A和受托对象B：  
>
> - 代理模式是委托者A直接转发方法调用给受托对象B，委托者A不定义被请求的方法  
> - 装饰器模式是在委托者A中定义相关方法，并在方法内部调用受托方B的某个或某多个方法  
>
> (就像装饰器函数一样，定义一个外部函数，内部封装另一个函数)
>
> 显然，没必要去关注这些细节，按需使用即可。

还可以使用`__setobj__()`临时修改对象的受托方，使用`__getobj__()`获取当前的受托方。比如除了Memory对象具有读写功能，磁盘(Disk)也具有读写功能，如果一开始Computer实例对象委托给了Memory对象，之后想要委托给Disk进行读写，就可以使用`__setobj__()`。

```ruby
class Storage
  def initialize
    @data = []
  end
  def write(data)
    @data << data
  end
  def read(index)
    @data[index]
  end
end
class Memory < Storage
end

class Disk < Storage
end

class Computer < SimpleDelegator
end

# 委托给Memory对象
comp = Computer.new(Memory.new)
comp.write("hello")
comp.write("world")
puts comp.read(0)         # 输出"hello"

# 更改委托对象：委托给Disk对象
comp.__setobj__(Disk.new)
comp.write("HELLO")
comp.write("WORLD")
puts comp.read(1)         # 输出"WORLD"
```

当然，也可以在方法内部修改受托对象。例如，在源码`lib/ruby/2.6.0/delegate.rb`中给出了一个示例：

```ruby
class Stats
  def initialize
    @source = SimpleDelegator.new([])
  end

  # stats方法自身设置委托对象：委托给records
  def stats(records)
    @source.__setobj__(records)

    "Size: #{@source.size}"
  end
end

s = Stats.new
puts s.stats(%w{James Edward Gray II})    # 委托给这个数组
puts
puts s.stats([1, 2, 3, nil, 4, 5, 1, 2])  # 委托给另一个数组
```

## Forwardable转发方法

Forwardable模块提供了`def_delegator()`和`def_delegators()`：

```
def_delegator(accessor, method, ali = method)
# accessor是方法名或实例变量名或常量名
# 为区分方法和实例变量，实例变量要指定为`:@var`的形式
# 所有对ali的方法调用，都将转发给accessor.method
# 如果不指定ali，则默认转发给accessor的同名方法

def_delegators(accessor, *methods)
# 一次性定义多个同名方法转发。
# 下面是等价的：
# def_delegators :@records, :size, :<<, :map
# 
# def_delegator :@records, :size
# def_delegator :@records, :<<
# def_delegator :@records, :map
```

另外需要注意，Forwardable模块需要使用`extend`的方式扩展到类中，因为def_delegator等方法要作为类方法。

例如：

```ruby
require 'forwardable'

class Memory
  def initialize
    @data = []
  end

  def write(data)
    @data << data
  end

  def read(index)
    @data[index]
  end

  def size
    @data.size
  end
end

class Computer
  extend Forwardable
  # 所有对push方法的调用，都将转发给@memory.write
  def_delegator :@memory, :write, :push
  def_delegators :@memory, :read, :size

  def initialize
    @memory = Memory.new
  end
end
comp = Computer.new
comp.push "hello"
comp.push "world"
comp.push "nihao"
p comp.read(1)
p comp.size
```

## SingleForwardable为单个对象转发方法

某个类中使用`Forwardable`模块设置了要被代理的实例方法后，所有实例对象都会转发这些实例方法的调用请求。

而SingleForwardable是Forwardable的单个对象版本，它在具体的实例对象上进行方法代理，不会影响其它对象。就像为某个对象定义单例方法一样。

```ruby
obj.def_delegator(accessor, method, new_method=method)
# 所有对obj.new_method的方法调用将转发给accessor.method
# 不指定new_method参数时，则同名转发

obj.def_delegators(accessor, *methods)
# 一次性设置多个被转发的方法
# 以下等价：
# def_delegators :@records, :size, :<<, :map
# 
# def_delegator :@records, :size
# def_delegator :@records, :<<
# def_delegator :@records, :map
```

例如：

```ruby
require 'forwardable'

class Memory
  def initialize
    @data = []
  end

  def write(data)
    @data << data
  end

  def read(index)
    @data[index]
  end

  def size
    @data.size
  end
end

class Computer
  def initialize
    @memory = Memory.new
  end
end
comp = Computer.new

comp.extend SingleForwardable
comp.def_delegator "@memory", :write
comp.def_delegator "@memory", :read, "rd"
comp.def_delegators "@memory", :size, :read
comp.write("hello")
p comp.rd(0)
p comp.size
```



