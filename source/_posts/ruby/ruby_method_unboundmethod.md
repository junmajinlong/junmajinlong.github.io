---
title: Ruby方法对象和UnboundMethod
p: ruby/ruby_method_unboundmethod.md
date: 2020-05-27 17:46:16
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby方法对象和UnboundMethod

## 将方法转换成方法对象

方法自身不是对象，但可以转换成对象。将一个非对象转换成对象，这称为对象化(objectify)操作。

转换后的方法对象是Method类的实例。

![](/img/ruby/1590573077745.png)

这里的meth就是一个Method对象，这个对象可以像Proc一样被调用。

注意上面调用的method()方法，它以`obj.method(Sym)`的方式将Sym对应的方法转换成一个方法对象(Method类的实例对象)。这个方法对象在内部已经绑定在obj上，这使得以后调用meth时可省略obj这个接收者。

自定义一个类和一个实例方法来理解更容易：

```ruby
class C
  attr_accessor :name
  def talk
    puts "hello: #{self.name}"
  end
end

c = C.new
c.name = "cccc"
c.talk      # 通过对象名显式调用它的方法talk

meth = c.method(:talk)  # meth绑定在c上，对应c.talk
meth.call   # 通过Method对象的方式直接调用，其内部等价于c.talk
```

输出：
```
hello: cccc
```

obj.method()返回的方法对象meth是绑定在obj上的，所以通过meth.call执行时，talk内部的self也仍然对应于对象c。

所以，obj.method()的作用是将一个对象的方法封装成一个Method对象，这个Method对象中有它绑定的对象obj。

此外，Object还提供了：  
- `public_method`：是method的public版本，只允许转换public属性的方法(method可以转换private、protected以及public属性的方法)  
- `singleton_method`：是method的singleton版本，只允许转换单例方法  

## Method对象的to_proc效果

Method对象实现了to_proc，所以可以使用`&meth_obj`这种方式替代语句块进行迭代。

```ruby
[1, 2, 3].each(&method(:puts)) #=> prints 3 lines to stdout
# [1,2,3].each {|x| method(:puts).call(x)}

out = File.open('test.txt', 'w')
[1, 2, 3].each(&out.method(:puts)) #=> prints 3 lines to file

require 'date'
%w[2017-03-01 2017-03-02].collect(&Date.method(:parse))
#=> [#<Date: 2017-03-01 ((2457814j,0s,0n),+0s,2299161j)>,
     #<Date: 2017-03-02 ((2457815j,0s,0n),+0s,2299161j)>]
```

## Method对象和Proc重组成新的可执行整体

Method对象还有两个比较有趣的功能：

```
meth << g → a_proc     (1)
meth >> g → a_proc     (2)
```

其中meth是Method对象，g是一个Proc对象。

- 方式(1)：先调用Proc对象g，得到返回值，再将返回值作为参数传递给meth，并执行meth  
- 方式(2)：先调用meth，得到返回值，再将返回值作为参数传递给Proc对象g，并执行g  

注意，它们都返回Proc对象，所以返回时是不执行的，调用时才按如上所述顺序执行。

换句话说，这两种方式是将两段可调用的代码组合在一起，并推迟到需要的时候一块执行。

如果不考虑它们的惰性推迟，其实它们在行为以及结果上分别等价于：

```ruby
meth.call(g.call())
g.call(meth.call())
```

例如：

```ruby
def f(x)
  x * x
end

f = self.method(:f)
g = proc {|x| x + x}
p (f << g).call(2)     # 16，等价于f.call(g.call(2))
p (f >> g).call(2)     # 8， 等价于g.call(f.call(2))
```

## 解绑和重绑

可以使用`unbound()`对一个Method对象进行解绑，然后再用`bind(obj1)`绑定到另一个对象obj1上，obj1可以是同类对象或子类对象。
```ruby
class C
  attr_accessor :name

  def talk
    puts "say hello: #{self.name}"
  end
end

c = C.new
d = C.new
c.name = "cccc"
d.name = "dddd"

c_meth = c.method(:talk) # 绑定在对象c上
p c_meth     # #<Method: C#talk>

unmeth = c_meth.unbind  # 从对象c上解绑
p unmeth     # #<UnboundMethod: C#talk>

d_meth = unmeth.bind(d)  # 绑定到对象d上
p d_meth     # #<Method: C#talk>
d_meth.call  # say hello: dddd
```

输出结果：
```
#<Method: C#talk>
#<UnboundMethod: C#talk>
#<Method: C#talk>
say hello: dddd
```

Module模块提供了`instance_method()`方法和`public_instance_method()`方法，可以直接在类上或模块上使用这两个方法返回该类或该模块的一个未绑定的Method对象。
```ruby
unmeth1 = C.instance_method(:talk)
p unmeth1    # #<UnboundMethod: C#talk>
```


## 调用指定父类的方法

得到一个未绑定的方法有时候是有奇效的。例如，在子类对象上直接调用某方法时，它只会执行第一个查找到的方法，如果想要明确执行父类或某个层次的祖先类中的同名方法，这时只能通过未绑定方法来实现。

例如：

```ruby
class A
  def talk
    puts "I am A"
  end
end

class B < A
  def talk
    puts "I am B"
  end
end

class C < B
end

c = C.new
c.talk  # 调用的父类B中的talk
```

这时想要通过对象c来执行祖先类A中的talk方法，可以获取祖先类A的一个未绑定talk方法，然后再将其绑定到c上。
```ruby
meth = A.instance_method(:talk)
meth.bind(c).call  # 输出"I am A"
```

如果想要在C类中定义一个调用父类的方法，可以将上面的过程封装到C类中：
```ruby
class C<B
  def call_super
    A.instance_method(:talk).bind(self).call
  end
end
```
