---
title: Ruby观察者模式(Observer)
p: ruby/ruby_observer.md
date: 2020-06-01 19:37:33
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby观察者模式(Observer)

![](/img/ruby/1591023476319.png)

这类示例很多。比如，股民可以通过一个软件帮忙盯着股价，这个软件允许股民关注某些股票的价格(或其它信息)，每当软件发现股民关注的股价达到了股民设置的高警戒线(可能要抛售)或低警戒线(可能要跳楼)，软件都会通知股民。

再比如流量监控软件，它允许用户指定要监控哪些应用程序并设置警戒线，当被监控的应用程序使用的流量超出了警戒线，便通知用户或者禁止它的流量。

## 手动实现观察者模式代码

在[Observer Pattern in RUBY](https://medium.com/@mitchocail/observer-pattern-in-ruby-e80ac3c1dac7)中给出了很好的示例，这里简单重复一遍。

教授准备设置期中考试，但是时间未定。学生、教授助理和系主任都关注教授确定期中考试的这个时间：学生要准备开始复习，助理要预定考试教室，系主任要准备审核。

最初实现的版本是比较丑陋的版本：

```ruby
class Professor
  attr_reader :name, :exam_date
  def initialize(name, subject, stu, assistant, dean)
    @name = name
    @subject = subject
    @stu = stu
    @assistant = assistant
    @dean = dean
  end

  def set_midterm(midterm_date)
    @exam_date = midterm_date

    # 确定时间后，调用(通知)学生、助理和系主任
    @stu.update(self)
    @assistant.update(self)
    @dean.update(self)
  end
end

class Student
  def update(prof)
    puts "start studying for Prof. #{prof.name} exam on the #{prof.exam_date}"
  end
end
class Assistant
  def update(prof)
    puts "find a classroom for Prof. #{prof.name} exam on the #{prof.exam_date}"
  end
end
class Dean
  def update(prof)
    puts "go over for Prof. #{prof.name} exam on the #{prof.exam_date}"
  end
end

stu = Student.new
ass = Assistant.new
dean = Dean.new
prof = Professor.new("Jeff", "Software", stu, ass, dean)
prof.set_midterm("2020-03-22")
```

上面的实现很丑陋，因为Professor负责了太多东西，如果同一个教授有多个课程要期中考试，这种实现会变得糟糕。如果关注教授期中考试时间的除了学生、助理、系主任，还有其它人也关注，这会变得更糟糕。

一种改善方式是将关注教授其中考试时间的人都收集到一个数组，谁关注就添加到数组。

```ruby
class Professor
  attr_reader :name, :exam_date

  def initialize(name, subject)
    @name = name
    @subject = subject

    @observers = []
  end

  def add_observer(observer)
    @observers << observer
  end

  def set_midterm(midterm_date)
    @exam_date = midterm_date

    # 确定时间后，调用(通知)学生、助理和系主任
    notify_observers
  end

  def notify_observers
    @observers.each do |observer|
      observer.update(self)
    end
  end
end
```

现在，谁关注，谁就加入observers数组：

```ruby
prof = Professor.new("Jeff", "Software")
stu = Student.new
prof.add_observer(stu)

ass = Assistant.new
prof.add_observer(ass)

dean = Dean.new
prof.add_observer(dean)

prof.set_midterm("2020-03-22")
```

因为一个教授可能有多个课程要期中考试，所以还可以继续完善，按学科设置期中考试的时间关注，并将其抽离。

```ruby
module Subject
  def add_observer(observer)
    @observers = [] unless defined? @observers
    @observers << observer
  end

  def notify_observers
    @observers = [] unless defined? @observers
    @observers.each do |observer|
      observer.update(self)
    end
  end
end

class Professor
  include Subject
  attr_reader :name, :exam_date

  def initialize(name, subject)
    @name = name
    @subject = subject
  end
  def set_midterm(midterm_date)
    @exam_date = midterm_date

    # 确定时间后，调用(通知)学生、助理和系主任
    notify_observers
  end
end
```

## Ruby Observable模块实现观察者模式

observer中定义的`Observable`模块提供了几个方法：

```
1.add_observer(obj, func=:update)
将对象obj添加为订阅者，func为notify_observers时调用的方法，
默认为update方法即obj.func必须是可响应的(respond_to)

2.changed(state=true)
设置状态。当changed为true时，意味着应使用notify_observers通知订阅者

3.changed?
返回changed状态

4.count_observers()
返回当前订阅者的数量

5.delete_observer(observer)
删除某个订阅者

6.delete_observers()
删除所有订阅者

7.notify_observers(*arg)
通知订阅者，即调用订阅者对象的update方法(默认为该方法)，
同时传递arg参数给update。通知完成后，将自动设置changed=false。
注意，该方法会检查changed状态，只有为true时才会通知
```

注意，因为`notify_observers`时会调用订阅者的update()方法，所以订阅者必须能响应update方法，且update方法的参数必须能符合`notify_observers(*arg)`的arg。

例如：

```ruby
require 'observer'

class Professor
  include Observable
  attr_reader :name, :exam_date

  def initialize(name, subject)
    @name = name
    @subject = subject
  end
  def set_midterm(midterm_date)
    @exam_date = midterm_date

    # 确定时间后，调用(通知)学生、助理和系主任
    changed
    notify_observers(self)
  end
end
```

## 分析Ruby Observer的源码

Observable模块的源码非常简单。当某个类Mix-in该模块后，会为每个实例对象创建`@observer_peers`和` @observer_state`两个实例变量，后者保存changed状态，前者是一个hash结构，保存的是每个observer(key)和该通知该observer时调用的函数(value)，默认的调用函数是`:update`。

```ruby
module Observable
  def add_observer(observer, func=:update)
    @observer_peers = {} unless defined? @observer_peers
    # 指定的调用函数必须能响应
    unless observer.respond_to? func
      raise NoMethodError, "observer does not respond to `#{func}'"
    end
    @observer_peers[observer] = func
  end

  # 从@observer_peers中删除observer
  def delete_observer(observer)
    @observer_peers.delete observer if defined? @observer_peers
  end

  # 删除@observer_peers中所有observer
  def delete_observers
    @observer_peers.clear if defined? @observer_peers
  end

  # 统计observer的个数
  def count_observers
    if defined? @observer_peers
      @observer_peers.size
    else
      0
    end
  end

  # 设置changed状态，默认状态为true
  def changed(state=true)
    @observer_state = state
  end

  def changed?
    if defined? @observer_state and @observer_state
      true
    else
      false
    end
  end

  # 通过遍历@observer_peers，通知所有的observer并调用它们的调用函数
  # 要求changed状态为true，同时设置changed状态为false
  def notify_observers(*arg)
    if defined? @observer_state and @observer_state
      if defined? @observer_peers
        @observer_peers.each do |k, v|
          k.send v, *arg
        end
      end
      @observer_state = false
    end
  end
end
```

模块倒是很简单，但是它有一个不足：Observable模块只提供了状态改变时通知所有对象的行为。这意味着它只能为所有Observer监控一个共同的指标并通知。

但也有很多应用场景是只通知一个指定的对象，比如多个对象的监控标准不一样，当满足某个对象的监控要求时，只通知关注该标准的对象即可，而不能遍历通知所有对象。

比如，流量监控软件允许用户监控多个应用程序，并允许为每个应用程序设置不同的流量上限，当某个应用程序的流量超出上限后，只对该应用程序做出处理即可。

## 多观察标准

怎么实现多标准观察呢？加一个方法通知单个Observer就行了。显然，通知单个Observer而不是根据某个监控标准的变化去通知所有Observer，所以不应该去依赖于changed状态。

```ruby
module Observable
  # 区分名称：notify_observer()和notify_observers()
  def notify_observer(observer, *arg)
    if defined? @observer_peers
      raise "#{observer} is not a observer" unless @observer_peers.include? observer
        observer.send @observer_peers[observer], *arg
    end
  end
end
```

现在，使用这个通知单个observer的方法notify_observer()去通知学生、助理和系主任。这里使用遍历的方式去通知：

```ruby
def set_midterm(midterm_date)
  @exam_date = midterm_date

  @observer_peers.each do |k, _|
    notify_observer(k, self)
  end
end
```

