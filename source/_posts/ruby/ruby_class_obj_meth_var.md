---
title: 厘清Ruby中类和对象的各种方法、各种变量
p: ruby/ruby_class_obj_meth_var.md
date: 2020-05-24 15:37:39
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# 厘清Ruby中类和对象的各种方法、变量

Ruby中有以下一些概念(当然，还有其它类型的变量)：  

![](/img/ruby/1590318070174.png)

例如，下面代码中出现的一大堆乱七八糟的东西，后文会一一解释。

```ruby
class C
  # 类C的局部变量
  a = 3
  puts "a in C: #{a}"      # 输出：a in C: 3

  # 类C的实例变量(类自身也是对象)
  @b = 33
  puts "@b in C: #@b"      # 输出：@b in C: 33

  # 类C的实例方法，即类方法
  # 类方法是定义在类C对象的单例空间中
  def C.f
    puts "@b in C.f: #@b"  # 输出：@b in C.f: 33
  end

  # 类变量，被所有实例和类自身所共享的变量
  # 其特性类似于实例方法，被所有实例共享
  # 但类变量还能被类自身访问
  @@x = 333
  puts "@@x in C: #@@x"    # 输出：@@x in C: 333

  def m
    puts "@@x in m: #@@x"  # 输出：@@x in m: 333
  end
end

C.f
C.new.m
```

## Ruby局部变量

Ruby中有几种局部环境：  
1. def函数定义开启的作用域  
2. class关键字开启作用域  
3. module关键字开启作用域  
4. 语句块或Proc内声明的变量是局部变量  

```ruby
class C
  a = 3   # class上下文中的局部变量
  
  def f
    b = 4  # def上下文中的局部变量
  end
  
  # x和y都是语句块上下文新建的变量，都是局部变量
  1.upto(5) {|x| y=0;puts x,y}
end

module
  aa = 33  # module上下文中的局部变量
end
```

注意，虽然class和module上下文中定义的局部变量无法被方法访问，但此时可以考虑使用常量：

```ruby
class A
  name = "junmajinlong"
  Name = "junmajinlong"
  def f
    #puts name    # 报错
    puts Name     # 正确
  end
end
A.new.f
```

## 实例变量和实例方法

实例变量是每个对象都独立拥有的变量，以`@`符号开头，按照常规定义的方式，它只能在实例方法内部被创建(赋值)或被访问(实际上也能在class和module内部创建，见下文)。实例方法是所有对象都共享的属于对象的方法。

子类会继承实例方法，但不会继承实例变量，因为实例方法是共享的，实例变量是每个对象独有的。

```ruby
class C
  def initialize(name, age)  # 实例方法
    @name = name    # 创建实例变量@name
    @age = age      # 创建实例变量@age
  end
  
  def get_name       # 定义实例方法
    return @name     # 访问实例变量name
  end
end

class D < C
end

c1 = C.new("junmajinlong", 23)
puts c1.get_name

d1 = D.new("gaoxiaofang", 24)
puts d1.get_name
```

由于Ruby中的类自身和模块自身也是对象，因此，除了上述在实例方法中创建实例变量的常规方式，还能在class和module内部创建各自的实例变量：

```ruby
# class内部的实例变量，属于类对象，
# 可在类上下文和类方法中被访问
class A
  # @name属于A
  @name = "junmajinlong"
  puts "in A: #{@name}"
  def self.m
    puts "in A.m: #{@name}"
  end
  
  # 在实例方法中使用`self.class.VAR`访问类的实例变量
  def f
    self.class.@name
  end
end
A.m

# module内部的实例变量，属于模块对象，
# 可在模块上下文和模块方法中被访问
module M
  # @name属于M
  @name = "junmajinlong"
  puts "in M: #{@name}"
  def self.m
    puts "in M.m: #{@name}"
  end
end
M.m
```

## 类变量

类变量是所有对象都共享的变量，以`@@`符号开头，可在类上下文定义，也可以在实例方法内部被定义。

子类会继承类变量，因此子类的类上下文以及子类的实例方法内也可以访问父类的类变量。这是因为类变量的目的是为了让所有该类的对象都能访问该变量，而子类对象也属于父类。

```ruby
class C
  @@num = 0  # 创建类变量
  
  def initialize
    @@num += 1    # 实例方法中能访问类变量
  end
  
  def how_many 
    return @@num  # 实例方法中能访问类变量
  end
  
  def C.how_many
    return @@num  # 类方法中也能访问类变量
  end
end

c1 = C.new
c2 = C.new
p C.how_many
p c1.how_many

class D < C    # 会继承父类的类变量@@num
  puts "in D: #@@num"
end
p D.how_many
```

如果想要在class外面访问类变量，可考虑定义一个类变量的getter方法，也可以使用`class_eval`进入class上下文，还可使用Module模块提供的`class_variable_get()`方法。

```ruby
class C
  @@num = 0
end

p C.class_eval("@@num")
p C.class_variable_get "@@num"
```

类变量会被类自身以及所有子类(可能有兄弟子类)共享，相当于是一种限定在亲缘类中的全局变量。

```ruby
class A
  # 类变量
  @@class_var = "class_var in A"
  puts "in A: #{@@class_var}"
  
  # 实例方法中也能访问类变量
  def m
    puts "in A.m: #{@@class_var}"
  end
end
A.m

# 子类的类上下文和子类的实例方法中都能访问父类的类变量
class B < A
  puts "in B: #{@@class_var}"
  def mm
    puts "in B.mm: #{@@class_var}"
  end
end
```

任意一个位置修改类变量，都是全局的修改。所以，Ruby中不建议使用类变量来保存数据。


在Ruby中，更好的方式是使用类的实例变量来替代类变量，因为类的实例变量更容易被控制。关于类的实例变量，下文马上介绍。

```ruby
class C
  @num = 0
  class << self
    attr_accessor :num
  end

  def initialize
    self.class.num += 1
  end
end

c = C.new
p C.num
```

## 类(模块)方法、类(模块)的实例变量

在Ruby中，**所有的类都是Class类的对象(例如`Class.new()`可创建一个类)，所有的模块都是Module类的对象(例如`Module.new()`可创建一个模块**。

```ruby
class C;end
module M;end
C.class       #=> Class
M.class       #=> Module

C1 = Class.new
c1 = C1.new   #=> #<C1:0x00007fffc21861d0>
```

所以，**类或模块自身也是对象**，为了与类实例化后得到的对象(例如`Person.new`)进行区分，**类或模块自身对象称为类对象或模块对象**(当然，它们是Class类、Module类的实例对象)。

既然类或模块自身也是对象，它们当然也有自己的实例变量和实例方法，它们称为类的实例变量和类的实例方法。实际上，**类的实例方法，就是类方法**。

```ruby
module M
  @a = 3   # 模块M的实例变量
  puts "@a in M: #@a"
  
  def M.f   # 模块M的实例方法，即模块方法
    puts "in M.f"
  end
end
M.f      # 可直接调用模块方法，因为M是对象

class C
  include M  # mix-in时，不会mix-in模块方法
  
  @b = 4     # 类对象C的实例变量，注意不是类变量
  puts "@b in C: #@b"
  def C.g    # 类对象C的实例方法，即类方法
    puts "in C.g"
  end
end

C.g      # 可直接调用类方法，因为C是对象
# C.f    # 报错，因为Mix-In时不会扩展模块方法
```

如果了解JavaScript，会发现Ruby的类方法、类的实例变量相当于JavaScript中直接在类对象上定义属性。模块的实例变量和模块方法同理。总而言之，类或模块的实例变量和它们自身的实例方法，都属于类对象自身或模块自身。

```javascript
// javascript
class C {}
C.name = "junmajinlong"

// 等价于Ruby中类的实例变量: class C; @name = "junmajinlong"; end
```


因为**类方法和模块方法的定义方式都是`def obj.x`，这表示类方法和模块方法都是定义在类对象或模块对象的单例类空间中的**。

定义类方法或模块方法时，更常见的方式是在类上下文或模块上下文中使用self来代指类名或模块名。这有两种方式：`def self.x`或`class << self`。

```ruby
class C
  # f1是类C的类方法
  def self.f1  # 等价于def C.f1
    puts "class method: self.f1"
  end
  
  class << self     # 等价于class << C
    # f2和f3也是类C的类方法
    def f2
      puts "class method: f2"
    end
    def f3
      puts "class method: f3"
    end
  end
end

C.f1
C.f2
C.f3

module M
  # f1是模块M的模块方法
  def self.f1  # 等价于def M.f1
    puts "module method: self.f1"
  end
  
  class << self     # 等价于class << M
    # f2也是模块的模块方法
    def f2
      puts "module method: f2"
    end
  end
end

M.f1
M.f2
```

一定要注意，不要在`class << self`内部再使用`def self.x`，因为前者已经进入了类对象或模块对象的单例类空间，再在其内使用`def self.x`又打开了该单例类对象的单例空间，即打开了类对象或模块对象的单例类的单例类空间，并在其中定义了方法x。如果真的这样定义了，需使用`singleton_class`方法返回类或模块的单例类，再调用它的单例方法。

```ruby
class C
  class << self
    def f1
      puts "singleton_class of C"
    end
    def self.f2
      puts "singleton class of C's singleton_class"
    end
  end
end
C.f1
C.singleton_class.f2
```

既然类或模块自身也是对象，也可以为它们定义属性的getter和setter方法，比如使用`attr_accessor`来定义。但是要注意，如果直接在class...end内部使用`attr_xxx`来定义，它是为实例对象定义存取方法，而非为类对象定义。要为类对象定义属性的存取方法，需进入类对象或模块对象的类空间上下文，即它们的单例类空间：

```ruby
class C
  class << self
    attr_accessor :name
  end
end

module M
  class << self
    attr_accessor :name
  end
end

C.name = "junmajinlong"
p C.name
M.name = "gaoxiaofang"
p M.name
```

**子类会继承父类的类方法(即类的实例方法)，但不会继承父类的类实例变量**。

类实例变量专属于类对象，显然是不共享的，如果想要共享变量，可使用类变量。

对于类方法，它是类对象的实例方法，它们定义在类对象的单例类空间中，在逻辑上它们是类对象所独有的。但从另一个角度上来看，子类本就是衍生自父类的，父类具有的行为方法，子类也应当具备，包括父类的类方法(其本质是：类方法定义在单例类空间中，子类继承父类时，子类的单例类空间同时也会继承父类的单例类空间，所以子类也能访问父类的类方法)。

```ruby
class C
  @cls_name = "C"
  def self.f
    puts "in self.f"
  end
end

class < D
  # puts @cls_name  # 报错，无法访问类C的类实例变量
end

D.f    # 子类会继承父类的类方法
```

最后，**Class是Module的子类，所以类也是一个模块**。Module类中定义了很多直接以模块(名)为接收者的方法，它们也都适用于类(名)。例如，Module中有一个方法instance_methods用来返回模块中所有已定义的实例方法，在类名上使用也是一样允许的：

```ruby
class C
  def f1;end
  def f2;end
end

p C.instance_methods(false)
```

其实，很多时候的模块和类是『等价的』概念。例如，类空间的上下文和模块空间的上下文的性质是一样的，类中可以定义类变量、类方法、类实例变量、实例方法，模块中也可以定义类变量、模块方法、模块变量、实例方法(Mix-in后将作为类中对象的实例方法)，再例如，类和模块之间均可以互相嵌套，等等，只不过**模块不可直接被实例化、模块不被继承，而是被Mix-in**，但由于某对象Mix-in模块后，该模块会加入到该对象的祖先链中，因此该对象也可以看作是该模块的实例。

```ruby
class C
  @name = "junmajinlong"  # 类实例变量
  
  def self.f  # 类实例方法，即类方法
  end
  
  def f  # 实例方法
  end
end

module M
  @name = "gaoxiaofang"
  def self.ff  # 模块方法
  end
  def ff  # 模块中定义的实例方法
  end
end
```

## 子类会继承哪些东西

子类在继承父类时，将继承：  

1. 实例方法  
2. 类方法  
3. 类变量  
4. 方法的可见性规则  
5. 常量  

常量也可被继承。但在子类中定义父类同名常量时，意味着对常量做修改，而常量在含义上是不能修改的。但事实上，子类中定义同名常量并没有修改常量，因为常量是通过名称空间来引用的，定义在不同名称空间中的常量是不同的常量。

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end

  # 二维空间原点常量
  ORIGIN = Point.new(0, 0)

  def to_s
    "(#@x, #@y)"
  end
end

class Point3D < Point
  def initialize(x, y, z)
    super(x, y)
    @z = z
  end

  # 三维空间原点常量
  ORIGIN = Point3D.new(0, 0, 0)

  def to_s
    "(#@x, #@y, #@z)"
  end
end

puts Point::ORIGIN
puts Point3D::ORIGIN
```















