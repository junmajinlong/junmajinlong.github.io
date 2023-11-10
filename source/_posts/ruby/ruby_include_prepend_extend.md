---
title: Ruby include、prepend和extend用法分析
p: ruby/ruby_include_prepend_extend.md
date: 2020-05-22 11:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby Mix-in三种方式

Ruby提供了三种方式(include/prepend/extend)将模块中定义的方法(功能)混入(Mix-in)到类或对象中。

相关方法：

```
# Mix-in的三种方式
include()  prepend()  extend()

# 三种Mix-in方式的底层实现
append_features()  prepend_features()  extend_features()

# 三种方式Mix-in完成后的回调函数
included()         prepended()         extended()

# 判断模块是否被Mix-in，实际上是判断祖先链中是否有指定模块
# extend添加的模块不在祖先链中，所以对extend的模块返回false
include?()

# 列出祖先链中已Mix-in的所有模块
# extend添加的模块不在祖先链中，所以不包含extend的模块
included_modules()
```

一般使用include()去Mix-in一个模块，这时候的祖先链为(类C中include M)：
```
C -> M -> Object -> Kernel -> BasicObject
```

如果是在类C中通过下面三种方式mix-in一个模块M：

![](/img/ruby/1590418120195.png)


## Mix-in哪些方法

虽然有三种方式来Mix-in模块中的方法，但Mix-in模块时，只会在目标上下文中【注入】模块中定义的普通方法，不会【注入】模块方法(即模块自身的实例方法)。

以include为例：

```ruby
module M
  # 模块方法(该模块对象自身的实例方法)，只属于该模块自身
  # Mix-in该模块时，该方法不会被注入到其他上下文
  def self.m
    puts "in M.m"
  end
  
  # 模块中定义的普通方法，属于目标上下文，不属于模块自身
  # Mix-in时将被注入到对应上下文
  def mm
    puts "in mm"
  end
end

class C
  include M
end
C.new.mm       # 正确
C.new.m        # 错误
```


## 方式一：include

C继承B，C通过include方式mix-in模块M，祖先链`C->M->B->Object->...`：

```ruby
class B
  def foo
    p 'foo from B'
  end
end

module M
  def foo
    p 'foo from M'
    super      # 调用B中的foo
  end
end

class C < B
  include M    # include M
  def foo
    p 'foo from C'
    super      # 调用M中的foo
  end
end

C.new.foo
=begin
"foo from C"
"foo from M"
"foo from B"
=end
```

## 方式二：prepend

C继承B，C通过prepend方式mix-in模块M，祖先链`M->C->B->Object->...`：

```ruby
class B
  def foo
    p 'foo from B'
  end
end

module M
  def foo
    p 'foo from M'
    super       # 调用C中foo
  end
end

class C < B
  prepend M     # prepend M
  def foo
    p 'foo from C'
    super       # 调用B中foo
  end
end

C.new.foo
=begin
"foo from M"
"foo from C"
"foo from B"
=end
```

## 方式三：extend

extend()是比较有趣的一个功能，它直接将模块中的属性**添加**到某个对象的单例类中去，而且**Mix-in的模块不会直接出现在继承链中，而是以『隐式的』方式体现在继承链中**。

因为类自身也是对象，所以extend的目标对象可以是类的实例对象，也可以是类对象自身。

如果在类上下文中进行extend，表示扩展类自身属性，所以添加的方法是类方法。**此时extend扮演的是隐式的include，即可以在类方法中使用super来调用模块中的同名方法**。

在对象上下文中进行extend，表示扩展对象属性，所以添加的方法是对象的单例方法。**此时extend扮演的是隐式的prepend，即可以在模块方法中使用super来调用类中的实例方法**。

例如，将模块属性添加到具体的实例对象中：

```ruby
module M
  def foo
    p 'foo from M'
    super        # 调用实例对象的foo
  end
end

class C
  def foo
    p 'foo from C'
  end
end

c = C.new
c.extend(M)     # 在实例上扩展
c.foo           # 调用来自于M的实例方法foo

=begin
"foo from M"
"foo from C"
=end
```

将模块添加到类中，即添加类方法：

```ruby
module M
  def foo
    p 'foo from M'
  end
end

class C
  def self.foo
    p 'foo from C'
    super      # 调用来自M的方法foo
  end
  extend M     # 在类上扩展
end

# 也可以在外面extend，效果相同
# C.extend(M)
C.foo          # 调用来自C自身的类方法

=begin
"foo from C"
"foo from M"
=end
```

## Mix-in多模块的顺序

使用include、prepend、extend去Mix-in多个模块时有两种方式：  

- 这些方法接多个模块作为参数，一次性Mix-in多个模块  
- 多次调用这三个方法，每次单独Mix-in一个模块  

但是这两种Mix-in多个模块的行为不一样。所以，Mix-in多个模块时，需考虑Mix-in各模块时的顺序。

例如，定义两个模块M和N：

```ruby
module M
  def foo
    p 'foo from M'
  end
end

module N
  def foo
    p 'foo from N'
  end
end
```

### include多个模块时

结论：  

```
(1).在类C中单独include M和N：[C, N, M, Object...]
include M
include N

(2).在类C中一次性include M和N：[C, M, N, Object...]
include M,N
```

其实结论很容易理解。每次调用include加入模块时，都是在当前类及其父节点中间插入模块节点。所以方式(2)直接将M和N插入在当前类的后面，所有N在M的后面。而方式(1)先在当前类的后面插入M，第二次仍然在当前类的后面插入N，所以N在M的前面。

例如：通过include()一次性Mix-in这两个模块时：

```ruby
class C
  include M, N
end

p C.ancestors
C.new.foo

=begin
[C, M, N, Object, Kernel, BasicObject]
"foo from M"
=end
```

分开include这两个模块：

```ruby
class C
  include M
  include N
end

p C.ancestors
C.new.foo

=begin
[C, N, M, Object, Kernel, BasicObject]
"foo from N"
=end
```

### prepend多个模块时

按照分析include()多个模块时的逻辑一样分析Prepend多个模块：  

- `prepend(M,N)`时，祖先链`[M,N,C,Object...]`  
- `prepend(M);prepend(N)`时，祖先链`[N,M,C,Object...]`  

例如：

```ruby
module M
  def foo
    p 'foo from M'
    super       # 调用N中的foo
  end
end

module N
  def foo
    p 'foo from N'
  end
end

class C
  prepend M,N
end

C.new.foo

=begin
"foo from M"
"foo from N"
=end
```

### extend多个模块时

虽然extend是将方法添加在对象的单例类中，而不会直接体现在继承链中，但为了方便分析，仍然考虑隐式链的存在。但注意，没有隐式链。

当在类上下文extend多个模块时：  

- `C.extend(M,N)`：隐式链`[C,M,N]`  
- `C.extend(M);C.extend(N)`：隐式链`[C,N,M]`  

当在对象上下文extend多个模块时：  

- `obj.extend(M,N)`：隐式链`[M,N,obj]`  
- `obj.extend(M);obj.extend(N)`：隐式链`[N,M,obj]`  

例如：

```ruby
module M
  def foo
    p 'foo from M'
  end
end

module N
  def foo
    p 'foo from N'
    super        # 调用M中的foo
  end
end

class C;end

a = C.new
a.extend M
a.extend N

a.foo
=begin
"foo from N"
"foo from M"
=end
```

## included、prepended和extended

它们分别是include()、prepend()和extend()的回调函数。

以prepended()为例：

```ruby
module M
  def self.prepended(othermod)
    puts "#{self} prepended in #{othermod}"
  end
end

class C
  prepend M    # prepend完成后，调用M.prepended(C)
end

=begin
M prepended in C
=end
```

