---
title: Ruby方法可见性规则：private、public和protected
p: ruby/ruby_private_protect_public.md
date: 2020-05-24 11:25:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby设置方法可见性：private、public和protected

Ruby中有三种方式可设置实例方法的可见性规则：private(私有)、public(公共)和protected(受保护)。

它们有两种方式设置方法的可见性，以private为例：

![](/img/ruby/1590291640434.png)

## private规则

通过private可将实例方法私有化，**私有化的方法只允许在当前类(或子类)内部(严格来说是实例方法内)以省略self的无点引用方式来调用**。

私有方法不允许使用`obj.meth`的方式来调用，连`self.meth`也不允许(因为在对象上下文，self就等价于obj)，所以它不能在外界访问。

```ruby
class C
  def f
    puts "f"
  end
  private :f      # f方法被私有
  
  def g
    puts "g"
    f             # 实例方法内无点引用私有方法
    # self.f      # 报错，不能通过self.f访问私有方法f
  end
end

C.new.g
#C.new.f       # 报错，不能通过obj.f访问私有方法f
```

**私有方法能在子类中被访问**，比如puts等『内置』方法就是私有方法，但它们可以在任何地方被调用。这是因为**子类继承父类方法的同时还会继承该方法的可见性规则**。所以，父类中的私有方法m，在子类中也是私有方法m，所以子类中也能访问m。但是，子类可以更改某方法的可见性规则，也可以在子类中重新定义方法，这将使得私有方法重新变为public。

```ruby
class C
  def f
    puts "f"
  end
  private :f
end

class CC < C
  def g
    puts "g"
    f         # 子类中引用继承自父类的私有方法f
    # self.f  # 报错，不能通过self.f访问继承的私有方法f
  end
end

CC.new.g
# CC.new.f    # 报错，不能通过obj.f访问继承的私有方法f
```

子类更改继承自父类的方法可见性规则：

```ruby
class C
  def f
    puts "f"
  end
  private :f
end

class CC < C
  public :f    # 允许
  def g
    puts "g"
    self.f     # 允许
  end
end

CC.new.g
CC.new.f       # 允许
```

不能在同类的其它对象中访问，因为要在其它对象中访问私有方法，总是会使用`obj.x`的访问方式。这一点主要是为了和protected可见性规则做比较。

```ruby
class Person
  attr_reader :age   # 定义def age()
  private :age       # 将age方法设置为私有
  
  def initialize name, age
    @name = name
    @age = age
  end

  def gt?(other)
    # 错误，不能使用other.age访问other对象的私有方法age
    # 即不能在同类的其它对象中访问该对象的私有方法
    @age > other.age
  end
end

p1 = Person.new("junmajinlong", 23)
p2 = Person.new("gaoxiaofang", 22)
p p1.gt? p2
```

需注意，无点引用setter方法时会和创建同名局部变量冲突，Ruby对这种情况的私有方法调用做了特殊处理：遇到同名局部变量时，允许使用`self.x`的方式来引用私有方法x。

另外，如果访问私有方法时已存在同名局部变量，则在访问私有方法时必须不能省略括号，否则表示访问局部变量而非私有方法。

```ruby
class C
  attr_accessor :age
  private :age=
  
  def f
    puts "in f"
  end
  private :f

  def m
    # age = 33   # 表示创建局部变量age
    self.age = 33  # 允许self.age引用私有setter方法age=
    f = 3
    f()      # 因存在同名局部变量f，所以括号不能省略
  end
end

c = C.new
c.m
puts c.age
```

## protected规则

通过protected可将方法保护起来，它的私有性处在public和private的中间：  
- 在类内部，protected扮演的角色是public，即对内公开(当然，也包括子类)  
- 在类外部，protected扮演的角色是private，即对外隐藏  

所以，受保护的实例方法允许在实例方法的内部访问以任意方式访问(包括无点引用m和self.m和obj.m)，但不允许在外界通过obj.m的方式来访问。

```ruby
class C
  def f
    puts "f"
  end
  protected :f

  def g
    self.f      # 允许
    f
  end
end

C.new.g
#C.new.f    # 报错，不可在外界通过obj.m的方式访问受保护的m
```

protected主要是限制外界访问，类内部、子类或同类对象都是自由的。例如：

```ruby
class Person
  attr_reader :age     # 定义def age()
  protected :age       # 将age方法设置为protected
  
  def initialize name, age
    @name = name
    @age = age
  end

  def gt?(other)
    # 允许在同类其它对象中访问该对象的受保护方法
    @age > other.age
  end
end

p1 = Person.new("junmajinlong", 23)
p2 = Person.new("gaoxiaofang", 22)
p p1.gt? p2
```


## public规则

当定义某个对象的实例方法时(除了特殊的initialize方法外)，如果不做任何处理，它们默认就是public的方法，即直接暴露给外界，外界可以通过obj.x的方式来调用obj对象的x方法。

```ruby
class C
  def f
    puts "in f"
  end
end
c = C.new
c.f        # 通过obj.x的方式调用对象c的方法f
```

对于不做任何修改的方法是public的说法，有两个例外：  

- 初始化实例对象的方法initialize()永远是私有的，保证了对象只在最初被创建时被初始化  

- 在top-level中创建的方法虽不做任何可见性设置，但：  

  - 在irb、pry中，它们是public的  
  - 在Ruby程序文件中，它们是private的  
  - top-level是Object的对象main的单例空间，所以这些方法都定义在main的单例空间中  

```ruby
# top-level内的代码，都在main的单例空间内

# 在Ruby程序文件中
def f;end     # 自动私有的方法f
f
# self.f      # 报错，不能使用self.f调用私有方法f
p self.private_methods.include? :f  # true

# 在irb中
>> def f;end
>> self.f     #=> nil
>> self.public_methods.include? :f  #=> true
```

## 通过send()绕过可见性规则

当x方法被设置为private或protected后，外界将不可通过obj.x访问obj对象的x方法。

可通过`send()`或`__send__()`来绕过可见性规则，因为这时并不是使用obj.x的方式来访问的。`__send__()`和`send()`等价，它只是为了避免某些情况下send被用户自定义为其它方法而提供的。

如果明确想要使用send()时也遵守可见性规则，可使用`public_send()`，它只会调用public方法，当调用private或protected的方法时将报错。

```ruby
class C
  private :f
  def f(arg)
    puts "in f: #{arg}"
  end

  def g
    puts "in g"
    self.send :f, "ARG"   # 绕过私有规则，同时传递参数
  end
end

c = C.new
c.send :f, "ARG"   # 绕过私有规则，同时传递参数

C.new.g

C.new.public_send :f, "ARG"  # 报错
```

## 设置类方法的可见性

Ruby中的类也是对象，类方法实际上是定义在类对象上的(在类对象的单例空间内)。所以，也可以设置类方法的可见性。

因直接使用`private`等方法设置可见性时，只能设置实例方法，所以要设置类方法(类对象的实例方法)的可见性规则，需进入类的单例空间：

```ruby
class C
  def self.f
    puts "in self.f"
  end
  
  class << self
    # protected :f
    # public :f
    private :f
  end

  def self.g
    puts "in self.g"
    f
    # self.f     # 报错
  end
end

# C.f            # 报错
C.g
```

Ruby额外提供了设置类方法可见性规则的方法，使得无需进入类对象的单例空间。但只提供了私有和公开两种可见性规则设置的方法：  

- `private_class_method`：将类方法私有化  
- `public_class_method`：将类方法公开  

```ruby
class C
  def self.f
    puts "in self.f"
  end
  private_class_method :f

  def self.g
    puts "in self.g"
    f
    #self.f       # 报错
  end
end

#C.f       # 报错
C.g
```

有时候确实需要将类方法设置为私有方法，比如将类方法new()设置为私有，使之不允许在外界创建该类的实例对象，只允许在类的内部通过无点引用new的方式来创建实例，比如这样可以实现单例类(和前文描述的单例类概念不同，此处的单例类是只允许有一个实例对象的类)。

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

## 设置常量的可见性

除了方法，还可以设置常量的可见性。同样，常量只有private和public两种可见性规则。

```
# 可以同时设置多个常量参数
private_constant
public_constant
```

设置私有常量后，将不能使用`::`的方式去访问，只能通过常量名访问。

```ruby
class C
  Version = '0.1.1'
  private_constant :Version

  # p C::Version     # 报错
  p Version
end
# p C::Version     # 报错
```

## 和private、public、protected相关的方法

在Object和Module中定义了与private、public和protected相关的方法。

### Object中定义的方法

```
methods()
返回对象的所有public和protected方法，包括继承链中的方法。加上参数false，只返回对象的单例方法

private_methods
protected_methods
public_methods
分别返回对象的private、protected、public方法，加上参数false，只返回当前对象方法，不考虑继承链
```

Object中定义的**这些方法的接收者是对象实例obj**。但注意，类或模块也是对象。

例如：

```ruby
p 33.private_methods.size
p Integer.public_methods.size

class C
end
p C.methods.size
```

### Module中定义的方法

Module中定义的这些方法的接收者是**类名或模块名**，而不能是实例对象(在类或模块上下文，可省略receiver)。这意味着它们是在类或模块空间中查找或设置类方法、实例方法。

```
# 下面这些前文已解释过
private
public
protected
private_class_method
public_class_method
private_constant
public_constant

instance_methods
对于类或模块，返回继承链中所有public和protected方法，
指定参数false，则不考虑继承链

private_instance_methods
protected_instance_methods
public_instance_methods
分别返回类或模块中定义的私有、公开和受保护的实例方法，
指定参数false，则不考虑Mix-in的模块

method_defined?
只作用于类或模块，判断继承链中的public和protected方法
中是否有指定方法。
指定第二个参数false，则不考虑继承链

private_method_defined?
public_method_defined?
只作用于类或模块，分别判断继承链中的private和public方法中
是否有指定方法。
指定第二个参数false，则不考虑继承链
```

例如，官方手册上给的示例：

```ruby
module A
  def method1()  end
end
class B
  private
  def method2()  end
end
class C < B
  include A
  def method3()  end
end

A.method_defined? :method1                   #=> true
C.private_method_defined? "method1"          #=> false
C.private_method_defined? "method2"          #=> true
C.private_method_defined? "method2", true    #=> true
C.private_method_defined? "method2", false   #=> false
C.method_defined? "method2"                  #=> false
```

