---
title: 搞懂Ruby的单例类
p: ruby/ruby_singleton_class.md
date: 2020-05-25 01:13:16
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby的单例类空间

**每个对象(类或模块自身也是对象)都有自己所属的类，这个类具有名称(即类名)，称为对象的具名类**。可通过`Object#class()`方法来查看一个对象所属的具名类。例如：

```ruby
3.class        #=> Integer
"h".class      #=> String
Integer.class  #=> Class

Class C;end
C.class        #=> Class
```

![](/img/ruby/1590340647621.png)

```ruby
class C;end
C.new.singleton_class #=> #<Class:#<C:0x00007fffdb748488>>
```

注意返回结果的格式，`#<Class:xxx>`表示返回的是Class类的一个对象，即一个匿名类。`#<C:0x00xxx>`表示的是C的对象，结合起来，整个返回结果表示这个匿名类是C的对象的匿名类，即C.new()得到的对象的单例类。

以后在调试输出`self`的时候，如果发现了类似`#<Class:xxx>`这样的返回结果，就要考虑此时的self是不是单例类。

```ruby
class C
  class << self
    puts self      # 输出：#<Class:C>，类C类对象的单例类
  end
end
```

需注意，对于那些不可变对象，它们的单例类空间是特殊的：  

- 对于nil、true或false，它们均是各自所属具名类的唯一对象，也是全局唯一的对象，它们没有额外的单例类，或者说，它们的单例类就是具名类  
- 对于Integer(及Float)和Symbol，它们的对象不能创建单例类，其中一方面原因是这些对象的值不符合标识符名称规范。例如，为数值对象3创建一个单例方法m的语法为`def 3.m`，这显然是不合理的  

```ruby
nil.singleton_class     #=> NilClass
3.singleton_class       # TypeError: can't define singleton
```

## 类是具名类还是单例类？

为了知道某个类是普通的具名类还是单例类，使用Module模块提供的`singleton_class?`方法进行判断：

```ruby
Integer.singleton_class?
#=> false

# singleton_class()在单例类不存在时会自动创建对象的单例类
Integer.singleton_class.singleton_class?
#=> true
```

注意，`singleton_class?`是Module模块提供的，这意味着只有那些类或模块才能作为方法的接收者，而不能是普通对象。

```ruby
class C;end
C.singleton_class?
# C.new.singleton_class?    # 报错
```

## 单例方法

使用`def obj.f`的方式表示打开对象obj的单例类，并在单例类空间中定义方法f，方法f称为单例方法，它只属于对象obj，不属于对象obj1、obj2等。

```ruby
a = "abc"
def a.f
  puts "in singleton_class"
end
a.f     # in singleton_class

"abc".f # NoMethodError: undefined method `f' for "abc":String
```

也可以使用`class << obj`的方式打开单例类并进入单例类空间中去定义方法，这些方法都在单例类中，所以都是obj的单例方法：

```ruby
class C;end

c = C.new
class << c
  def f
    puts "in singleton_class"
  end
end
c.f   # 输出："in singleton_class"
```

使用`Object#singleton_methods`方法可以返回一个对象的所有单例方法，默认会包含单例空间中mix-in模块的方法，指定参数为false时则忽略mix-in：

```ruby
class Single
  def Single.four() end
end
Single.singleton_methods    #=> [:four]

module Other
  def three() end
end

a = Single.new
def a.one()
end

class << a
  include Other
  def two()
  end
end

a.singleton_methods(false)  #=> [:two, :one]
a.singleton_methods         #=> [:two, :one, :three]
```

其实，Ruby中**所有类方法都定义在类对象(类也是对象)的单例类空间中**：

```ruby
class C
  def C.f1;end
  def self.f2;end
  class << self
    def f3;end
  end
end
p C.singleton_methods  # [:f1, :f2, :f3]
```

需注意，不要在单例类内部使用`def obj.x`的方式去定义方法，这不是在定义类方法，而是定义单例类对象的单例方法。例如：

```ruby
class C;end
class << C
  # 类方法
  def f1
    puts "in C's singleton_class"
  end
  
  # C的单例类对象的单例方法，即进入了C的单例类的单例类
  def self.f2
    puts "singtleton_class of C's singleton_class"
  end
end

C.f1
C.singleton_class.f2
```

## 理解普通具名类空间和单例类空间的本质

每个对象都有两个类空间，一个是普通具名类，一个是单例类，这两个类空间中定义的方法都属于实例对象。只不过普通具名类中定义的实例方法是所有对象共享的。且单例类空间中的方法优先于普通具名类中的实例方法。

从本质上来说，单例类空间的作用是随时随地地为对象提供该对象的类空间上下文，且优先级高于该对象所属的具名类空间。

```ruby
class C
  # C的实例对象的普通类空间
  # 此处定义的实例方法属于所有C的实例对象
end

c1 = C.new
class << c1
  # 对象c1的单例类空间
  # 此处定义的实例方法只属于c1对象
end
```

类(或模块)也是对象，是Class类的实例对象，它们也有自己的单例类空间和普通具名类空间。

```ruby
class C
  class << self
    # 类C的单例类空间，为类C对象自身提供类上下文空间
    # 相当于进入了类对象所属的普通具名类空间(即Class类)，但先于Class
    # 此处定义的实例方法，只属于类C，即类C的类方法
  end
end

class Class
  # 所有类或模块的普通具名类空间
  # 此处定义的实例方法，属于所有类或模块
end
```

![](/img/ruby/1590853260345.png)

需要特别注意的是，在Module模块中定义了不少方法是直接在模块或类内部使用，但操作的目标却是实例对象的方法。比如Module中定义的private方法：

```ruby
class C
  def f;end
  private :f  # private在类上下文，但操作的是实例方法f
end
```

换句话说，**Module中的这些方法需要在被操作对象所属的普通具名类空间或单例类空间中使用**。

例如，想要让类方法私有，那么操作的目标是类对象的实例方法，需要在类对象的类空间中设置，比如在单例类中：

```ruby
class C
  def self.f;end
  class << self
    private :f
  end
end
```

再比如，Module中提供的define_method()方法用于定义实例方法，`define_method(:f)`相当于`def :f `：

```ruby
class C
  class << self
    define_method(:f1) {puts "class_method"}
  end
  
  define_method(:f2) {puts "instance_method"}
end

C.f1
C.new.f2
```

## def obj.f和class << obj的区别

虽然下面两种定义对象单例方法的方式在效果上是等价的：

```ruby
class C;end
c = C.new

def c.f;end
class << c
  def f;end
end
```

但是它们身处的环境不同：

- `def obj.f`的内部是在obj对象的上下文，此时的self为obj自身  
- `class << obj`的内部是在obj对象的单例类上下文，此时的self是obj的单例类空间  
- 实际上，`class << obj`内部的`def f`的上下文才等价于`def obj.f`的上下文  

```ruby
class D
end
d = D.new

class << d
  puts "self in << d: #{self}"  #=> #<Class:#<D:0x000000000517ebc0>>
end
def d.f
  puts "self in d.f: #{self}"  #=> #<D:0x000000000517ebc0>
end

d.f
```

这有什么影响？影响太大了。比如，我有一次想要通过define_method定义一个类方法：

```ruby
class C
  class << self
    define_method(:f) {puts "class method"}
  end
end
C.f
```

上面是正确的定义方式。但是我想要将define_method()委托给另一个方法来调用：

```ruby
class C
  def self.create_method(name, &block)
    define_method(name, &block)
  end
end
C.f
```

这想当然的做法是错误的，因为define_method等价于`self.define_method`，而此时的self为类C对象自身，这等价于`C.defined_method`，这仍然是定义了一个C的实例对象的实例方法。

正确的定义方式应该是：

```ruby
class C
  class << self
    def create_method(name, &block)
      # self是C类对象自身
      # 进入C的单例类，define_method定义的是C的类方法
      self.singleton_class.define_method(name, &block)
    end
  end
end
C.create_method(:f) {puts "f"}
C.f
```

## 单例方法的回调函数

在BasicObject中定义了三个和单例方法相关的回调函数：

```
singleton_method_added       ：当添加单例方法时被调用
singleton_method_removed     ：当remove单例方法时被调用
singleton_method_undefined   ：当undefine单例方法时被调用
```

例如：

```ruby
class C;end
c = C.new

def c.singleton_method_added(meth)
  puts "Adding singleton_method: #{meth}"
end

def c.singleton_method_removed(meth)
  puts "Removing singleton_method: #{meth}"
end

def c.singleton_method_undefined(meth)
  puts "Undefining singleton_method: #{meth}"
end

def c.f1;end
def c.f2;end

class << c
  remove_method :f1
  undef_method :f2
end
```

输出结果：

```
Adding singleton_method: singleton_method_added
Adding singleton_method: singleton_method_removed
Adding singleton_method: singleton_method_undefined
Adding singleton_method: f1
Adding singleton_method: f2
Removing singleton_method: f1
Undefining singleton_method: f2
```

从结果可见，为对象设置这三个回调函数(也是单例方法)时也会触发`singleton_method_added`。如果想要忽略这三个方法，可在added回调中稍作判断：

```ruby
def c.singleton_method_added(meth)
  case meth
  when :singleton_method_added,
       :singleton_method_removed,
       :singleton_method_undefined
    return
  end
  puts "Adding singleton_method: #{meth}"
end
```

<a name="singleton_ancestors"></a>

##  单例类的祖先链

对于类或模块来说，有祖先链的概念，某个类或模块的祖先链可通过`ancestors()`方法查看：

```ruby
class C;end
class D < C;end
p D.ancestors 
#=> [D, C, Object, Kernel, BasicObject]
```

单例类也是类(只不过是匿名类)，它也有祖先链。因为子类D的单例类`#<Class:D>`也将会是父类C的单例类`#<Class:C>`的子类，即`D.singleton_class.superclass == C.singleton_class`。

例如：

```ruby
class C;end
class << C;end
class D < C;end
class << D;end
p D.singleton_class.ancestors
=begin
[#<Class:D>, #<Class:C>, 
#<Class:Object>, #<Class:BasicObject>, 
Class, Module, Object, Kernel, BasicObject]
=end
```

可见，单例类的祖先链规则为：

```
当前单例类 -> 父类的单例类 -> 父类的父类的单例类 -> ... 
-> Class -> Module -> Object -> Kernel -> BasicObject
```

这里有两条分支链：  

- 属于单例类的链，实际上很容易理解，因为子类会继承父类的类方法，而父类的类方法即父类的单例类中的单例方法  
- 单例类作为类对象，其类为Class，Class是Module的子类，Module是Object的子类，Object中混入了Kernel模块，Object是BasicObject的子类  
- 注意，单例类的链中不包含模块的单例类，模块不是类，不能继承，但模块是对象，所以在模块单例类的继承链中，除了模块自身的单例类外，其余的是[Module,Object,Kernel,BasicObject]  

而这条**单例类的祖先链正是类方法的查找规则**。先找所有单例类链，单例类中找不到类方法时再将类当作对象，去找类实例方法(即类方法)。

例如：

```ruby
class C;end
class << C
  def f1
    puts "in C.f1"
  end
end

class D < C;end

D.f1        # 在单例类链分支中找到
p D.superclass  # 在Class中找到实例方法superclass
```

上面单例类的继承链中，一个有趣的现象是：**祖先类BasicObject的单例类`#<Class:BasicObject>`，它既是Class类的子类，又是Class类的实例对象**。

```ruby
class C
end
class D< C
end

p D.singleton_class.superclass.superclass.superclass
p D.singleton_class.superclass.superclass.superclass.superclass
p D.singleton_class.superclass.superclass.superclass.class
=begin
#<Class:BasicObject>
Class
Class
=end
```


# 单例类：Singleton模块

在设计模式中，单例类指的是只有一个实例对象的类。

在Ruby中，单例类这个名称别有它意，但Ruby同样提供了一个实现单例类设计模式的**Singleton模块**，使得类只允许创建一个实例。

```ruby
class C
  include Singleton
end

c1 = C.instance
c2 = C.instance
c1 == c2   #=> true
# c3 = C.new     # 报错，不能使用new创建实例，new是私有的
# c1.dup         # 报错，不能dup单例对象
# c1.clone       # 报错，不能clone单例对象
```

当Klass类mix-in Singleton模块时，它做了以下几件事：

```
- 将Klass.new和Klass.allocate私有
- 重写Klass.inherited(sub_klass)和Klass.clone()以保证单例类在被继承和克隆时保留原属性
- 提供Klass.instance()方法创建唯一的实例，如果实例已存在，则返回该实例
- 重写Klass._load(str)使其调用Klass.instance()
- 重写Klass#clone和Klass#dup以保证单例对象在clone和dup时报错
```

实际上，自己也可以实现只允许单个实例对象的单例类。比如，简单版的：

```ruby
class C
  # 将new()私有化，不允许外界创建实例
  # 提供类方法instance()来创建唯一的实例对象
  # 并使用类实例变量@instance保存唯一的实例对象
  class << self
    private :new
    def instance
      @instance = @instance ? @instance : new("Admin")
    end
  end

  def initialize name
    @name = name
  end
end

c1 = C.instance
c2 = C.instance
p c1
p c2
p c1 === c2
=begin
#<C:0x0000000005189570 @name="Admin">
#<C:0x0000000005189570 @name="Admin">
true
=end
```

下面是Singleton模块的源码，挺有学习意义的：

```ruby
# 大致逻辑为：
# 1.禁用C实例对象的clone和dup实例方法
# 2.当在类C中include Singleton的时候，禁用C.new、C.allocate
# 3.当类C对象被clone和继承的时候，保留单例类的状态属性，将它们扩展为类方法即可
# 4.在include Singleton的时候，创建instance()类方法

module Singleton
  # 禁用对象的实例方法clone、dup
  # Raises a TypeError to prevent cloning.
  def clone
    raise TypeError, "can't clone instance of singleton #{self.class}"
  end

  # Raises a TypeError to prevent duping.
  def dup
    raise TypeError, "can't dup instance of singleton #{self.class}"
  end

  # By default, do not retain any state when marshalling.
  def _dump(depth = -1)
    ''
  end

  # 下面这些方法将通过extend的方式扩展到类对象中，即作为类方法
  # 定义在一个子模块中，主要是为了封装多个方法
  module SingletonClassMethods # :nodoc:

    def clone # :nodoc:
      Singleton.__init__(super)   
    end

    # By default calls instance(). Override to retain singleton state.
    def _load(str)
      instance
    end

    private

    def inherited(sub_klass)
      super
      Singleton.__init__(sub_klass)
    end
  end

  # 设置模块环境：定义模块方法__init__
  # 将在included时被调用(klass是目标类)，于是instance()被声明了
  class << Singleton # :nodoc:
    def __init__(klass) # :nodoc:
      klass.instance_eval {
        # 初始化一个类对象的实例变量用于保存唯一的单例对象
        @singleton__instance__ = nil
        @singleton__mutex__ = Thread::Mutex.new
      }
      def klass.instance # :nodoc:
        # 如果单例对象已存在，直接返回
        return @singleton__instance__ if @singleton__instance__
        # 否则，创建单例对象，且某一时刻只允许一个线程创建，否则线程不安全
        @singleton__mutex__.synchronize {
          return @singleton__instance__ if @singleton__instance__
          @singleton__instance__ = new()
        }
        @singleton__instance__
      end
      klass
    end

    private

    # 移除extend Singleton模块的扩展方式
    # extending an object with Singleton is a bad idea
    undef_method :extend_object

    def append_features(mod)
      #  help out people counting on transitive mixins
      unless mod.instance_of?(Class)
        raise TypeError, "Inclusion of the OO-Singleton module in module #{mod}"
      end
      super
    end

    def included(klass)
      super
      klass.private_class_method :new, :allocate   # 私有化类方法.new和.allocate
      klass.extend SingletonClassMethods
      Singleton.__init__(klass)
    end
  end

  ##
  # :singleton-method: _load
  #  By default calls instance(). Override to retain singleton state.
end
```

