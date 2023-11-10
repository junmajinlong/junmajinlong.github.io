---
title: Ruby序列化和持久化存储：Marshal和Pstore
p: ruby/ruby_marshal_pstore.md
date: 2020-05-16 22:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby Marshal序列化

Marshal是Ruby的核心库，可以将一些对象以二进制的方式序列化保存到文件中，需要时再从文件中加载重新构建成对象，即反序列化。

Marshal对数值、字符串、数组、布尔值等基础数据的序列化保存没有任何问题。

![](/img/ruby/1589647146161.png)

序列化和反序列化的过程非常简单：

```ruby
# 一个嵌套数组
arr = [
  %w(Perl Python PHP),
  %w(C C++ Java Golang),
  %w(Shell Powershell Cmdline)
  ]

# 将arr对象序列化保存到文件中
File.open('/tmp/data.dat', "w") do |file|
  Marshal.dump(arr, file)
end

# 反序列化
File.open('/tmp/data.dat') do |file|
  data = Marshal.load(file)
end

p data
```

Marshal.dump()还可以通过第三个参数指定最多允许序列化多少个嵌套的对象层次，即深度，超出了深度将报错。其默认值为-1，此时表示不检查深度，即dump所有层次。例如：

```ruby
arr = [
  %w(Perl Python PHP),
  [ %w(C C++), %(Java Golang) ],   #=> 3层
  %w(Shell Powershell Cmdline)
  ]

# 将arr对象序列化保存到文件中
File.open('/tmp/data.dat', "w") do |file|
  Marshal.dump(arr, file, 4)      #=> 小于4将报错
end
```

如果想要指定对象中要序列化的内容，或者指定序列化成什么类型，可以在类中编写`marshal_dump`和`marshal_load`方法。例如，只dump一部分数据并以数组的方式保存：

```ruby
class Klass
  def initialize name, age, height 
    @name = name
    @age = age
    @height = height
  end
  
  def marshal_dump
    [@name, @age]
  end
  
  # 反序列化，arr是序列化时的数组
  # 最终它返回一个Klass的实例对象
  def marshal_load arr
    @name, @age = arr
  end
end

# 序列化Klass的一个对象，但只会包含name和age两个属性
obj = Klass.new("junmajinlong", 23, 170)
File.open('/tmp/me.dat','w') do |file|
  Marshal.dump(obj, file)
end

# 反序列化，得到一个Klass的实例对象，并设置name和age属性
obj1 = File.open("/tmp/me.dat") do |file|
  Marshal.load file
end

p obj1
#=> #<Klass:0x00007fffcc0119f8 @name="junmajinlong", @age=23>
```

显然，上面反序列化的过程中缺少了一个height属性。为了让对象完整，在反序列化的时候需要根据反序列化得到的结果合理构建新对象。例如，使用instance_eval()构建新对象：

```ruby
def marshal_load arr
  self.instance_eval do
    initialize(*arr, 170)
  end
end
```

# Pstore存储

Pstore(persistence store)是Ruby的一个持久化存储的标准库，它以基于Hash数据类型的方式将数据以key-value的方式存储在文件中(二进制的)，其中value是想要存储的数据，key是这部分数据的一个名称。

在Pstore中，key称为root，每个key都是一个root。

Pstore是基于事务的，所以多次增删改数据的过程是原子的，可统一提交(commit)、统一回滚(abort)。commit()和abort()时都会立即终止本次事务，所以它们后面的代码不会执行，如果没有指定commit()或abort()，则在退出transaction的时候自动保存。

因为pstore每次读都要先加载文件部分内容到内存(直到找到对应的key)，所以读效率不高。再者，每次写入都需要拷贝文件的绝大部分数据，所以效率更低。因此，Pstore只适用于少量数据、少量读写的数据存储场景。

例如，持久化保存到文件：

```ruby
require 'pstore'

s = PStore.new('/tmp/pstore.dat')

s.transaction do
  s["p1"] = {name: "junmajinlong", age: 23, height: 170 }
  s["p2"] = {name: "junma", age: 22, height: 180}
  s.commit
  s["p3"] = {name: "jinlong", age: 24}
end

s.transaction do
  # 覆盖p2
  s["p2"] = {name: "jinlong", age: 24, height: 170 } 
end   #=> 自动commit
```

从pstore文件中读取数据：

```ruby
require 'pstore'

s = PStore.new("/tmp/pstore.dat")

p2 = s.transaction do
  s["p2"]
end
p p2
puts p2.name
```

`transaction(read_only=false)`还可以指定参数设置该事务是否只读，如果设置了只读，则事务内对pstore做任何修改都会抛出错误。

Pstore还有其它一些辅助方法：  

```
[KEY]     ：获取元素的值，如果元素不存在则返回nil
delete()  ：删除元素，可指定元素不存在时的默认值参数
fetch()   ：获取元素，如果元素不存在，默认报错，可指定默认返回值  
path()    ：返回pstore文件的路径
root?()   ：检查key是否存在  
roots()   ：返回所有的key
```