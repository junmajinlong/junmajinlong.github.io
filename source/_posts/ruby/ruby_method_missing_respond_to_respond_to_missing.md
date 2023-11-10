---
title: Ruby的method_missing、respond_to?以及respond_to_missing?
p: ruby/ruby_method_missing_respond_to_respond_to_missing.md
date: 2020-05-25 23:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby的method_missing、respond_to?以及respond_to_missing?

## 关于method_missing

默认情况下，当调用`obj.f()`时，将从它的单例类、继承链中查找方法f，如果最终没有找到f方法，则抛出错误：undefined method f。

```ruby
class C
end
C.new.f
```

![](/img/ruby/1590461817766.png)

例如，将`method_missing`作为对象的实例方法定义在类中：

```ruby
class C
  def method_missing(m, *args, &block)
    puts "can't find method: #{m}"
    super
  end
end
C.new.f
```

重载`method_missing`时，它的第一个参数是查找失败的方法名称(上例中为`:f`)，另外两个参数分别用于接收调用方法时给定的参数和语句块。

一般来说，会在重载的`method_missing`中使用super继续上抛方法缺失错误，但非必须。

因为可以自定义`method_missing`，使得很多方法看上去是存在的，只不过这些方法都被`method_missing`处理了。比如，只要是以`_name`结尾的方法，都认为是存在的方法：

```ruby
class C
  def method_missing(m, *args, &block)
    if m =~ /_name$/
      puts "finding method '*_name': #{m}"
    else
      super
    end
  end

end
C.new.my_name    # 输出finding method '*_name': my_name
C.new.good_name  # 输出finding method '*_name': good_name
C.new.abc        # 报错
```

## 关于respond_to?和respond_to_missing?

`method_missing`用于定义方法查找失败时的处理方式，而`respond_to?`则判断方法是否能查找成功，即接收者对象能否响应所调用的方法。默认只查找public方法，可指定相关参数使其也查找private和protected方法。

例如：

```ruby
class C
  def method_missing(m, *args, &block)
    if m =~ /_name$/
      puts "finding method '*_name': #{m}"
    else
      super
    end
  end

end
C.new.my_name    # 输出finding method '*_name': my_name
p C.new.respond_to? :helloword   #=> false
p C.new.respond_to? :my_name     #=> false
```

从上面最后一行代码的结果中可知，`respond_to?`无法响应那些能匹配`method_missing`的方法名，但是这些方法既然定义在`method_missing`中，可能就是想表达该方法已经存在。

为了能respond那些能匹配`method_missing`的方法名，重载该方法即可：

```ruby
class C
  def method_missing(m, *args, &block)
    if m =~ /_name$/
      puts "finding method '*_name': #{m}"
    else
      super
    end
  end

  def respond_to?(m, *)
    !!(m =~ /_name$/) || super
  end
end
C.new.my_name
p C.new.respond_to? :helloword    #=> false
p C.new.respond_to? :my_name      #=> true
```

虽然现在所有以`_name`为后缀的方法已经『存在』了，可以让`respond_to?`返回true，也可以使用`obj.xyz_name`的方式来调用该类方法。但是还不够，在不是使用`obj.xyz_name`的方式调用时，它仍然是不存在的，它毕竟不是真实存在的。比如method方法、public_method方法等将其进行转换时会报错，提示找不到方法，也即无法转换：

```ruby
class C
  def method_missing(m, *args, &block)
    if m =~ /_name$/
      puts "finding method '*_name': #{m}"
    else
      super
    end
  end

  # def respond_to?(m, *)
  #   !!(m =~ /_name$/) || super
  # end
end
C.new.my_name
p C.new.respond_to? :helloword
p C.new.respond_to? :my_name

# p C.new.method :my_name         # 报错
# p C.new.public_method :my_name  # 报错
```

Ruby为了让不存在但却能匹配`method_missing`的方法更像已存在的方法，还加入了一个`respond_to_missing?`替代`respond_to?`，重载前者而非后者，使得在真正操作那些缺失但能匹配`method_missing`的方法时认为其是存在的。

```ruby
class C
  def method_missing(m, *args, &block)
    if m =~ /_name$/
      puts "finding method '*_name': #{m}"
    else
      super
    end
  end

  # def respond_to?(m, *)
  #   !!(m =~ /_name$/) || super
  # end

  def respond_to_missing?(m, *)
    !!(m =~ /_name$/) || super
  end
end

C.new.method(:my_name).call   # 输出finding method '*_name': my_name
```

## 使用建议

大家都建议，在重载方法`method_missing`时，也总是重载`respond_to_missing?`，这是合理的建议。











