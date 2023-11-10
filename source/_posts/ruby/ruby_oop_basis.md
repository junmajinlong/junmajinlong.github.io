---
title: Ruby面向对象基础
p: ruby/ruby_oop_basis.md
date: 2020-05-25 01:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby面向对象基础

Ruby是一门纯面向对象的语言，是真正的一切皆对象。

1. **通过class()方法可以查看某个对象所属的类**

    ```ruby
    3.class          #=> Integer
    "hello".class    #=> String
    nil.class        #=> NilClass
    false.class      #=> FalseClass
    /a.*b/.class     #=> Regexp
    
    # 类自身也是对象，是Class类的对象
    # Class类自身也是Class的对象
    3.class.class        #=> Class
    3.class.class.class  #=> Class
    ```

2. **通过class...end可以创建一个类，类名要求首字母大写，类名将自动被定义为常量**   

3. **通过Cls.new()可以创建Cls类的实例对象**  

4. **通过`Cls.new()`创建对象时，将自动调用`initialize()`对将要创建的对象进行初始化**  

5. **可以在initialize()指定初始化对象时的参数**  

6. **实例变量以`@`符号开头，类变量以`@@`开头，其余小写字母开头的变量都是局部变量**  

    ```ruby
    # 创建空类
    class Cls;end
    # 创建对象
    obj = Cls.new
    
    # 创建包含内容的类
    class Cls1
      # 类变量
      @@classname = "Cls1"
    
      # 初始化方法
      def initialize name,age,gender
        @name = name
        @age = age
        @gender = gender
      end
    
      # 定义对象的实例方法
      def print_name
        # s是局部变量
        s = "hello"
        puts @name
      end
    end
    
    cls = Cls1.new("junmajinlong",23,"male")
    cls.print_name
    ```

7. **通过`obj.__id__`或`obj.object_id`可查看每个对象的ID，经过一些转换，可查看对象的地址**

    ```ruby
    class Cls;end
    obj = Cls.new      #=> #<Cls:0x00007fffc99983a8>
    obj.__id__                    #=> 70368287834580
    ( obj.__id__ << 1 ).to_s(16)  #=> "7fffc99983a8"
    ```

8. **要实现类的继承关系，使用`class P < C ... end`语法，这表示子类C继承父类P。使用superclass()可查看一个类的父类**  

    ```ruby
    Class C;end
    Class C < D;end
    p D.superclass
    ```

9. **Ruby中，BasicObject类是祖先类，Object类是BasicObject的子类，其余所有的类都继承自Object类**。使用ancestors()可查看继承链(祖先链)  

    ```ruby
    class C;end
    C.superclass            #=> Object
    Object.superclass       #=> BasicObject
    BasicObject.superclass  #=> nil
    
    Integer.superclass      #=> Numeric
    Integer.superclass.superclass
    #=> Object
    
    Integer.ancestors
    #=> [Integer, Numeric, Comparable, Object, Kernel, BasicObject]
    ```

10. **Ruby中不支持多重继承，为了达到多重继承的能力，Ruby使用模块的概念，模块可被Mix-in混入到类中，使得该类也具备模块中定义的行为**。Ruby中的模块和类是一样重要的概念，两者关系非常紧密，类是模块的子类，所以类也是模块。将class关键字换成module关键字，即表示定义模块，通常使用include将模块Mix-in到类或其它模块中   

    ```ruby
    module M
      def f
        puts "f in M"
      end
      def f1;end
      def f2;end
    end
    class C
      include M
    end
    
    c = C.new
    c.f
    c.f1
    ```

11. 类和模块的区别在于，可以根据类创建出该类的实例对象(如C.new创建出C的实例对象)，而不允许根据模块直接创建出模块的实例对象，模块只能被Mix-in。尽管模块不能直接创建出实例对象，但因为Mix-in模块时，该模块也会加入到对象的祖先链中，因此这些对象也算是模块的实例对象。实际上，模块和类的共同点非常多，很多类支持的行为，模块也支持，模块支持的行为，类也支持  

12. 使用`is_a?()`、`kind_of?()`和`instance_of?()`可判断一个对象是否属于一个类  

  - `is_a?`和`kind_of?`是别名关系，它们会考虑继承链  
  - `instance_of?`不考虑继承链，只考虑当前类  

    ```ruby
    class C;end
    c = C.new
    c.is_a? C             #=> true
    c.kind_of? C          #=> true
    c.is_a? Object        #=> true
    c.instance_of? C      #=> true
    c.instance_of? Object #=> false
    ```

12. **子类继承父类时，如果想要调用父类的同名方法，可使用`super`**。super有几种用法，详细用法可参见[Ruby中的super](h/ruby/ruby_super)   

13. **除了使用常规的创建对象的方式，也可直接使用Object.new来创建一个空对象**。因为是空对象，所以默认没有实例变量，Ruby允许在类定义的外部使用`instance_variable_set`设置实例变量，使用`instance_variable_get` 获取实例变量，使用`instance_variable_defined?`判断是否定义了某实例变量  

    ```ruby
    obj = Object.new
    obj.instance_variable_set(:@name, "junmajinlong")
    p obj.instance_variable_get(:@name)
    p obj.instance_variable_defined? :@name  # true
    ```

14. **Ruby中的类(和模块)也是对象，每个类都是Class类的实例对象，每个模块都是Module的实例对象，所以通过Class.new直接创建一个类，使用Module.new直接创建一个模块**   

    ```ruby
    C = Class.new
    C1 = Class.new do 
      def f
        puts "in f"
      end
    end
    
    c = C.new
    c1 = C1.new
    c1.f
    ```

15. **类和模块的定义代码，在定义时就被执行**  

    ```ruby
    class C
      puts "www.junmajinlong.com"    # 直接输出
    end
    puts "WWW.JUNMAJINLONG.COM"
    ```

16. **Ruby中可以通过`def obj.f`的方式在类的外部定义只属于对象obj的方法f，这样的方法称为对象obj的单例方法**。其实，**每当使用`def obj.x`语法时，都是在打开obj对象的单例类空间，并在该空间内定义方法x。单例类是特殊的匿名类，它优先于当前对象所在类被查找**，所以单例方法可覆盖类中定义的实例方法  

    ```ruby
    class C
      def f
        puts "in f"
      end
    end
    c = C.new
    c.f  # 调用c对象的f方法，来自于类中实例方法
    
    def c.f  # 定义c对象的单例方法f，将覆盖类中的实例方法
      puts "in c.f"
    end
    
    c.f  # 调用c对象的f方法，只不过来自c的单例空间
    p c.singleton_methods    # 输出[:f]
    ```

17. **除了`def obj.x`会打开对象obj的单例类空间，`class << obj`也会打开并进入对象obj的单例类空间，它们的作用都是进入单例类上下文的词法作用域**。注意obj是对象，class后的东西并非一定得是类名(当然，类自身也是对象)。两者的区别：前者只能做单次定义，后者像是进入了语句块，可在单例空间中执行多条语句 

    ```ruby
    class C;end
    c = C.new
    
    def c.f1
      puts "in c.f1"
    end
    
    class << c
      def f2
        puts "in f2"
      end
      
      def f3
        puts "in f3"
      end
    end
    ```

18. **更为常见的打开单例类空间的方式是`def self.f`和`class << self`**。关于Ruby中的self，参见下文给出的链接  

19. **Ruby中，实例属性被隐藏在对象的『黑匣子』中** 

    - 外界不可通过obj.name的方式访问obj对象的实例属性`@name`  
    - 实例方法内部可以通过`@name`访问自身的属性name  
    - 可通过定义属性的getter方法来访问默认被隐藏的对象属性  

    ```ruby
    # 创建包含内容的类
    class Cls
    
      def initialize name,age
        @name = name
        @age = age
      end
    
      def get_name
        @name
      end
    end
    
    cls = Cls.new("junmajinlong",23)
    #cls.name  # 报错
    puts cls.get_name
    ```

20. Ruby提供了三个特殊的属性getter/setter方法：  

    - **attr_reader：自动定义了指定属性的getter方法**  
    - **attr_writer：自动定义了指定属性的setter方法**  
    - **attr_accessor：自动定义了指定属性的getter和setter方法**  
    - 它们可接受字符串类型的属性名，也可以接受符号类型的属性名  
    - 例如`attr_reader :name`等价于`def name; @name end`  
    - `attr_writer :name`等价于`def name=(name); @name=name end`  
    - 如果想让某属性只读，只设置getter不设置setter即可  
    - 设置getter方法后，在类定义的外部，可通过`obj.name`的方式访问obj对象的name属性，实际上是调用了`obj.name()`  
    - 设置getter方法后，在实例方法内部，要访问实例变量name，可以通过`self.name`的方式来访问，实际上是调用了`self.name()`  

    ```ruby
    class Cls
      # 为name和age两个实例变量设置getter方法
      attr_reader :name, :age
      
      def initialize name,age
        @name = name
        @age = age
      end
      
      # 实例方法内部，使用self.name访问name属性
      def print_name
        puts "my name is: #{self.name}"
      end
    end
    
    obj = Cls.new("junmajinlong",23)
    
    # 等价于obj.name()，通过getter方法访问实例属性
    puts obj.name
    ```

21. **在类定义和模块定义的内部，可使用`self`来表示当前对象**。关于self具体含义，参见[Ruby中self的含义](/ruby/ruby_self)  

22. **在对象内部，所有『无点引用』的方式其实都是省略了self的引用**  

23. **使用public、protected和private可设置实例方法、类方法、常量的可见性**。关于方法的可见性规则，参见[Ruby方法可见性规则：private、public和protected](/ruby/ruby_private_protect_public)  

24. **Ruby中的类或模块也是对象，是对象就具有实例方法、实例变量等**。在类作为对象时相关的行为，具体参见：[厘清Ruby中类和对象的各种方法、各种变量](/ruby/ruby_class_obj_meth_var)  

25. **Ruby中的对象除了属于一个具有名称的具名类外，还属于一个更高查找优先级的匿名类：单例类**。关于单例类的详细说明，参见[搞懂Ruby的单例类空间](/ruby/ruby_singleton_class)  
