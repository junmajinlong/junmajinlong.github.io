---
title: Ruby Struct类型
p: ruby/ruby_struct.md
date: 2020-05-26 19:37:31
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby Struct

如果想要存储带属性的数据(key/value或类似key/value的数据)，可采用hash或对象来存储，还可以采用struct结构来存储。Ruby中并没有像C、Golang一样提供真正的Struct数据类型，Ruby是以对象的方式实现Struct的，所有Struct结果都是Struct类的子类且内部类。

## Struct和对象的区别

如果使用对象的方式存储属性固定且只需存、取方法(也有其它一些基本方法)的数据，一般需要编写如下代码：

```ruby
class D
  attr_accessor :name, :age
  def initialize(name,age)
    @name = name
    @age = age
  end
end

d1 = D.new("junmajinlong", 23)
d2 = D.new("gaoxiaofang", 24)
```

但是可以更简单：

```ruby
D = Struct.new(:name, :age)
d1 = D.new("junmajinlong", 23)
d2 = D.new("gaoxiaofang", 24)
d2.name = "xiaofang"   # setter方法修改属性
p d2.name              # getter方法获取属性
```

什么时候使用对象，什么时候使用Struct呢？首先，Struct结构也能当类使用，正如上面的D和普通类没什么区别。如果知道要在小范围内使用，或者是一次性使用的类，那么可以考虑使用Struct。

## Struct用法说明

通过`Struct.new`可以构造出一个Struct类的子类，这个子类即可当作Struct结构使用，也可当作类来使用。

```
new([class_name] [, member_name]+) → StructClass
new([class_name] [, member_name]+, keyword_init: true) → StructClass
new([class_name] [, member_name]+) {|StructClass| block } → StructClass
```

其中： 

![](/img/ruby/1590567376086.png)

因为返回子类，所以可以赋值给一个变量(任意变量名，首字母可小写)，此时就有两个名称指向这个类。

```ruby
p Struct.new("C", :name, :age)       # Struct::C
a = Struct.new("A", :name, "age")
p a == Struct::A                     # true

b = Struct.new(:name, :age)
```

如果`keyword_init`设置为true，则表示创建对象实例时，使用`key:value`格式参数。

```ruby
P = Struct.new(:name, :age, keyword_init: true)
p1 = P.new(name: "junmajinlong", age: 23)
p p1
```

如果给定语句块，则可以为Struct子类定义实例方法或其它逻辑：

```ruby
P = Struct.new(:name, :age) do
  def who_am_i
    puts "name is: #{name}"   # 注意，不是#@name
  end
end
p1 = P.new("junmajinlong", 23)
p1.who_am_i
```

建议创建Struct的匿名子类并将其赋值给变量，而不是为其指定一个名称，因为子类都保存在Struct名称空间中，子类名可能会冲突，且匿名子类方便被垃圾回收器回收。

另外，Struct存放的数据的key是固定的，不能随意增加、减少struct成员，因为Struct内部没有定义其它属性的setter/getter方法。

## Struct对象的实例方法

Struct对象存取数据很方便，既可以使用对象的点方式，也可以使用数组的数值索引、hash的key索引方式来检索。如果访问不存在的成员或索引越界，则直接报错：

```ruby
P = Struct.new(:name, :age)
p1 = P.new("junmajinlong", 23)
p p1.name
p p1[:name]
p p1["name"]
p p1[0]       #=> 顺序固定，index=0一定是name，index=1一定是age

p p1["abc"]   # 报错
```

还有以下几个方法：

```
each()        迭代所有成员的value
each_pair()   迭代所有成员的key/value
members()     返回所有成员的key
to_a()或values() 等价，返回所有成员的value
values_at()      返回指定成员(需使用数值索引来指定)的value
to_h()           返回所有成员的键值对
length()或size() 返回成员个数，即Struct对象有几个属性
==     如果是同一个Struct子类的对象，且value都相等(根据==比较)，则true
eql?   如果是同一个Struct子类的对象，且value都相等(根据eql?比较)，则true

filter()或select()
等价，筛选符合条件的成员
Lots = Struct.new(:a, :b, :c, :d, :e, :f)
l = Lots.new(11, 22, 33, 44, 55, 66)
l.select {|v| v.even? }   #=> [22, 44, 66]
```

## Struct和Hash结构的区别

1. Struct的key是固定不可更改的，无法增、减key，而hash在这方面是自由的  
2. Struct构建语法更简洁，例如`Foo = Struct.new(:a,:b,:c)`之后，可以不用指定key的名称直接创建struct对象`Foo.new(1,2,3)`  
3. 默认情况下，Struct访问不存在的元素会报错，而hash是返回nil，尽管可以设置默认值或fetch()抛错  
4. Struct对象在遍历时，顺序是固定可保证的  
5. 可为Struct对象提供能访问成员属性的实例方法，而想要为一部分Hash对象单独提供方法(如果只是单个hash对象，可设置单例方法)，稍嫌麻烦  

```ruby
Person = Struct.new(:name, :age) do
  def who_am_i
    puts "name is: #{name}"
  end
end

p1 = Person.new("junmajinlong", 23)
p1.who_am_i
```


## struct、hash、class和openstruct的对比

Ruby中可以使用这四种方式存储数据，但它们的性能有所区别。下面是创建这四种结构存储数据的性能对比：

参考：<https://palexander.posthaven.com/ruby-data-object-comparison-or-why-you-should-never-ever-use-openstruct>。

```ruby
require 'ostruct'    # 提供openstruct
require 'benchmark'

COUNT = 10_000_000
NAME = "Test Name"
EMAIL = "test@example.org"

class Person
  attr_accessor :name, :email
end

Benchmark.bm(13) do |x|
  x.report("hash:") do
    COUNT.times do
      p = {name: NAME, email: EMAIL}
    end
  end

  x.report("struct:") do
    PersonStruct = Struct.new(:name, :email)
    COUNT.times do
      p = PersonStruct.new
      p.name = NAME
      p.email = EMAIL
    end
  end

  x.report("class:") do
    COUNT.times do
      p = Person.new
      p.name = NAME
      p.email = EMAIL
    end
  end
  
  x.report("openstruct:") do
    COUNT.times do
     p = OpenStruct.new
     p.name = NAME
     p.email = EMAIL
   end
  end
  
end
```

测试结果：

```
                    user     system      total        real
hash:           0.836551   0.000639   0.837190 (  0.837230)
struct:         1.713467   0.000065   1.713532 (  1.713540)
class:          1.130315   0.000000   1.130315 (  1.130318)
openstruct:    51.333243   0.000012  51.333255 ( 51.333304)
```

所以，hash最快、class稍慢一点，struct比class又更慢一点，openstruct最慢(非常非常慢，因此不要用它)。
