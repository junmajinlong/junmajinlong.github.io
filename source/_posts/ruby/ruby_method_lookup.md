---
title: Ruby方法查找路径：祖先链
p: ruby/ruby_method_lookup.md
date: 2020-05-25 22:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby方法查找路径：祖先链

先说结论：  

![](/img/ruby/1590419526070.png)

## 实例方法的查找路径

实例方法的查找路径非常简单，就是按照所属类的继承链来查找实例方法，一经找到，立即停止。

定义类C和类D，D继承于C：

```ruby
class C;end
class D < C;end
```

那么D的继承链为：

```ruby
D.ancestors()
# [D, C, Object, Kernel, BasicObject]
```

所以，如果调用`D.new.f()`，将首先从D类中寻找实例方法f，没有找到将继续从父类C中寻找实例方法f，仍未找到，于是再依次找Object、Kernel模块以及BasicObject中是否有实例方法f。

因为mix-in模块有include、prepend以及extend三种方式：关于模块的include、prepend和extend详细的用法和细节，参见[Ruby include、prepend和extend用法分析](/ruby/ruby_include_prepend_extend/)。  

- include模块时，模块添加在继承链中当前类的后面   
- prepend模块时，模块添加在继承链中当前类的前面  
- extend模块时，如果是obj.extend(M)，则模块M中的实例方法以prepend的方式隐式添加在当前类的前面(实际上是对象obj的单例类空间中)  
- extend模块时，如果是Mod.extend(M)，则模块M中的实例方法被添加为类方法或模块方法，且以include的方式添加在当前类的后面(实际上是该类对象的单例空间中)  

所以，对于如下示例：

```ruby
module M1;end
module M2;end
module M3;end
class C
end
class D < C
  include M1
  prepend M2
end
d = D.new
d.extend(M3)
```

如果调用`d.f()`，将首先从M3中查找，再从继承链`[M2, D, M1, C, Object, Kernel, BasicObject]`查找。

## 类方法的查找路径

参见：[搞懂Ruby的单例类空间：单例类的祖先链](/ruby/ruby_singleton_class#singleton_ancestors)。

## 路径查找图示

![](/img/ruby/path_lookup.jpg)

