---
title: Ruby复制对象
p: ruby/ruby_dup_clone.md
date: 2020-05-29 22:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby对象复制

## 对象复制：dup()、clone()

Ruby中可以使用`Object#dup()`或`Object#clone()`来**浅拷贝**对象，它们基本等价，区别稍后再谈。下面是使用dup()复制一个对象的示例：

```ruby
class C
  attr_accessor :name, :age, :create_time

  def initialize(name, age)
    @name, @age = name, age
    @create_time = Time.now
  end
end

c = C.new("junmajinlong", 24)
sleep 1.5
c1 = c.dup
p c1.name
p c.create_time
p c1.create_time

=begin
"junmajinlong"
2020-05-20 20:09:26 +0800
2020-05-20 20:09:26 +0800
=end
```

上面使用dup()复制对象c得到对象c1，c和c1两个对象除了少数几个属性，其它状态和属性是一样的，比如实例变量的值相同，具有的实例方法相同，方法的可见性规则相同(即public/private/protected)，等等。

dup()和clone()有几点不同：  

![](/img/ruby/1590842985513.png)

## initialize_dup、initialize_clone和initialize_copy的区别

无论是dup()还是clone()，它们在处理了一些逻辑后，会分别调用`initialize_dup()`和`initialize_clone()`。

dup()和clone()的伪代码大概是这样的：

```ruby
# 伪代码：在Object中如下定义dup和clone方法
class Object
  def clone
    # 分配一个空对象
    clone = self.class.allocate

    # 向空对象中填充拷贝的实例变量和单例类
    clone.copy_instance_variables(self)
    clone.copy_singleton_class(self)

    # 调用initialize_clone
    clone.initialize_clone(self)
    # 拷贝frozen状态
    clone.freeze if frozen?

    # 返回克隆的对象
    clone
  end

  def dup
    dup = self.class.allocate
    dup.copy_instance_variables(self)
    dup.initialize_dup(self)
    dup
  end
end
```

`initialize_dup()`和`initialize_clone()`在内部又会继续调用`initialize_copy()`，在Rubinus的core/kernel.rb中有如下代码，可知它们的关系：

```ruby
# 在core/kernel.rb中：
def initialize_clone(other)
  initialize_copy(other)
end
private :initialize_clone

def initialize_dup(other)
  initialize_copy(other)
end
private :initialize_dup

def initialize_copy(other)
  # 内部一些判断操作
end
private :initialize_copy
```

所以，如果自己手动重写`initialize_dup`和`initialize_clone`时，建议也调用一下`initialize_copy`，或者直接super一下(其实就是在调用initialize_copy)。

那么重写这三个方法有什么用呢？**如果在复制对象的时候，有些属性不想复制，比如随机数、时间点、ID等各对象独立的属性，那么可以重写这些方法**。

要注意：  

- 在重写这些方法的时候，这些方法是在复制并填充完对象属性后再调用的，所以在这些方法中只需要修改那些需要修改的属性即可  
- 这三个方法内的self指向的是拷贝出的对象，它们还接受一个参数other，other是被复制的源参照对象，例如`c1 = c.dup`，那么other是源对象c，这三个方法内的self是新对象c1  

例如，不想在复制对象的时候连对象的`@create_time`时间点也复制，而是定义为复制时的时间点。直接重写`initialize_copy`即可。

```ruby
class C
  attr_accessor :name, :age, :create_time

  def initialize(name, age)
    @name, @age = name, age
    @create_time = Time.now
  end
  
  def initialize_copy(other)
    # 其它属性已经填充好了，
    # 这里只修改需要修改的属性@create_time
    # 因为self是新对象，所以直接使用@create_time
    @create_time = Time.now
  end
  
  # 或者重写initialize_dup/clone，使用super
  #def initialize_dup(other)
  #  super
  #  @create_time = Time.now
  #end
end

c = C.new("junmajinlong", 24)
sleep 2
c1 = c.dup
p c1.name
p c.create_time
p c1.create_time

=begin
"junmajinlong"
2020-05-20 20:40:06 +0800
2020-05-20 20:40:08 +0800
=end
```

