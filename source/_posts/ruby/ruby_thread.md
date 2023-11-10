---
title: 使用Ruby线程库(Thread)
p: ruby/ruby_thread.md
date: 2020-06-04 19:14:36
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# 使用Ruby线程库(Thread)

Thread是Ruby的线程库，Thread库已经内置在Ruby中，但如果想要使用线程安全的Queue、Mutex以及条件变量等，则需要手动`require 'thread'`。

## 主线程main

默认情况下，每个Ruby进程都具备一个主线程main，如果没有创建新的线程，所有的代码都将在这个主线程分支中执行。

使用`Thread.main()`类方法可获取当前线程组的主线程，使用`Thread.current()`可以获取当前正在执行的线程分支。使用`Thread.list()`可获取当前进程组中所有存活的线程。

```ruby
p Thread.main
p Thread.current
p Thread.main == Thread.current
=begin
#<Thread:0x0000000001d9ae58 run>
#<Thread:0x0000000001d9ae58 run>
true
=end
```

可见，线程其实是一个Thread类的实例对象。

## 创建Ruby线程

使用Thread库的new()、start()、fork()可创建线程，它们几乎等价，且后两者是别名关系。

![](/img/ruby/1591366438333.png)

如果主线程先执行完成，主线程将直接退出，主线程的退出将会终止进程，使得其它线程也会退出。

```ruby
Thread.new {puts "hello"}
puts "world"
```

上述代码几乎总是会输出`world`，然后退出，主线程的退出使得子线程不会输出`"hello"`。之所以总是会输出world而不是输出hello，这和Ruby的线程调度有关，在后面的文章中会详细解释Ruby中的线程调度。

## join()和value()等待线程

如果想要等待某个线程先执行完成，可使用`t.join()`，如果线程t尚未退出，则join()会阻塞。可以在任意线程中调用`t.join()`，谁调用谁等待。

```ruby
t = Thread.new { puts "I am Child" }
t.join  # 等待子线程执行完成
puts "I am Parent"
```

还可以将多个线程对象放进数组，然后执行遍历join，另一种常见的做法是使用`map{}.each(&:join)`的方式：

```ruby
threads = []
3.times do |i|
  # 将多个线程加入到数组中
  threads << Thread.new { puts "Thread #{i}" }
end

# 在main线程中join每个线程，
# 因此只有3个线程全都完成后，main线程才会继续，即退出
threads.each(&:join)
=begin
Thread 1
Thread 0
Thread 2
=end

# 另一种常见方式
3.times.map {|i| Thread.new { puts "Thread #{i}" } }.each(&:join)
Array.new(3) {|i| Thread.new { puts "Thread #{i}" } }.each(&:join)
```

`t.value()`和`t.join()`类似，不同之处在于`t.value()`在内部调用`t.join()`等待线程t之后，还会在等待成功时取得该线程的返回值。

```ruby
a = Thread.new { 2 + 2 }
p a.value   #=> 4
```

注意，对于Ruby来说，无论是否执行join()操作，任务执行完成的线程都会马上被操作系统回收(从OS线程表中删除)，但被回收的线程仍然能够使用`value()`方法来获取被回收线程的返回值。之所以会这样，我个人猜想，也许是因为Ruby内部已经帮我们执行了join操作并将线程返回值保存在Ruby内部，这样对于用户来说就更加安全，而且用户执行join()或value()操作，可能是在等待Ruby内部的这个值的出现。

## 线程的异常处理

默认情况下，当某个非main线程中抛出异常后，该线程将因异常而终止，但是它的终止不会影响其它线程。

```ruby
t = Thread.new {raise "hello"}    # 抛出异常
sleep 1    # 仍然睡眠1秒后退出
```

如果使用了`t.join()`或`t.value()`去等待抛出异常的线程t，异常将会传播给调用这两个方法的线程。例如主线程调用`t.join`，如果t会抛出一次异常，那么主线程在等待过程中还会抛出一次异常。

```ruby
t = Thread.new {raise "hello"}    # 抛出异常
t.join()    # 子线程抛异常后，main线程也抛异常
```

如果想要让任意线程出现异常时终止整个程序，可设置类方法`Thread.abort_on_exception`为true，它会在任意子线程抛出异常后自动传播给main线程，从而终止进程：

```ruby
Thread.abort_on_exception = true
Thread.new { raise "Error" }
sleep 1   # 不会睡眠完1秒，而是子线程异常后立即异常退出
```

如果想要让某个特定的线程出现异常时终止整个程序，可设置同名的实例方法`t.abort_on_exception`为true，只有t线程异常时才会终止程序。

```ruby
t1 = Thread.new { raise "Error from t1" }
t1.abort_on_exception = true
sleep 1
```

另外，线程实例方法`t.raise()`可以直接在线程t抛出异常。

需注意，Ruby线程有一个巨大的缺点：无论是raise抛出异常还是各种终止(比如kill、exit)，都不会执行ensure子句。

## 线程的状态和生命周期

Ruby中的线程具有5种状态，可通过`t.status()`查看，该方法有5种对应的返回值：

```
- run: 线程正在运行(running)或可运行(runnable)  
- sleep: 线程处于睡眠态，比如阻塞(如sleep,mutex,io block)  
- false: 线程正常退出后的状态，包括执行完流程、手动退出(t.exit)、信号终止(t.kill)  
- nil: 线程因抛出异常(比如raise)而退出的状态  
- aborting: 线程被完全kill之前的过渡状态，不考虑这种状态的存在  
```

另外，还有两种统称状态：  

- alive：存活的线程，等价于run + sleep  
- stop：已停止的线程，等价于sleep + dead(false+nil)  

可分别使用`alive?()`和`stop?()`来判断线程是否属于这两种统称状态。

此外：

```
Kernel.sleep：让当前线程睡眠指定时长，无参数则永久睡眠，线程将进入睡眠队列
Thread.stop：让当前线程睡眠，进入睡眠队列，等价于无参数的sleep  
Thread.pass：转让CPU，当前线程进入就绪队列而不是睡眠队列  
t.run：唤醒线程t使其进入就绪队列，同时让当前线程放弃CPU，调度程序将重新调度  
t.wakeup：唤醒线程t使其进入就绪队列，但不会让当前线程放弃CPU，调度程序将不会立即重新调度  

Thread.kill：终止指定线程，它将不再被调度  
Thread.exit：终止当前线程，它将不再被调度  
t.exit,t.kill,t.terminate：终止线程t，t将不再被调度  
```

几个注意事项：  

- 这里5个终止线程的方式效果上是完全等价的，三个实例方法是别名关系，而两个类方法的内部也都是调用线程对象的kill  
- 最好要不加区分地看待run和wakeup  
- 对于Thread.pass，除了知道它转让CPU的行为是确定的，不要对它假设任何额外的行为，比如不要认为出让CPU后一定会调度到其它Ruby线程，很有可能会在调度其它一些非Ruby线程后再次先调度到本线程而非其它Ruby线程  
- 需注意，无论是raise抛出异常还是各种终止(比如kill、exit)，都不会执行ensure子句  

## 线程私有变量和局部变量

Ruby进程内的所有线程共享进程的虚拟地址空间，所以共享了一些数据。

但线程是语句块或者Proc对象，所以语句块内部创建的变量是在当前线程栈内部的，是每个线程私有的变量。

```ruby
# 主线程中的变量
a = 1

# 子线程
t1 = Thread.new(3) do |x|
  a += 1
  b=3
  x=4
end

# 主线程
t1.join
p a   # 2
#p b  # 报错，b不存在
#p x  # 报错，x不存在
```

Ruby为线程提供了局部变量共享的概念，每个线程对象都可以有自己的局部数据空间(即线程本地变量)，线程对象的局部空间互不影响，比如两个线程中同时进行正则匹配，两个线程的`$~`是不一样且互不影响的。

线程对象`t`的局部数据空间是`t[key]=value`，即一个名为t的hash结构，因为对象t是可以共享的，所以它的局部空间也是共享的。

```ruby
t1 = Thread.new do
  t = Thread.current
  t[:name] = "junmajinlong"
  t[:age] = 23
end

t1.join

p t1.keys          # [:name, :age]
p t1.key? :gender  # false
p t1[:name]        # "junmajinlong"
t1[:age] = 24  
p t1[:age]         # 24
```

所以，有这么几个方法：

```
t[key]
t[key]=
t.keys
t.key?
```

此外还有一个fetch()方法，类似于Hash的fetch()，默认情况下访问不存在的key会异常，可指定默认值或通过语句块返回默认值。

严格来说，从Ruby 1.9出现Fiber之后，`t[]`不再是线程本地变量(thread-local)，而是纤程(Fiber)本地变量(fiber-local)。但也支持使用线程本地变量：

```ruby
t.thread_variables
t.thread_variable?
t.thread_variable_get
t.thread_variable_set
```

## 线程组

默认情况下，所有线程都在默认的线程组中，这个默认线程组是Ruby程序启动时创建的。可使用`ThreadGroup::Default`获取默认线程组。

```ruby
t1 = Thread.new do
  Thread.stop
end

p t1.group
p Thread.current.group
p ThreadGroup::Default
=begin
#<ThreadGroup:0x00000000019bcb60>
#<ThreadGroup:0x00000000019bcb60>
#<ThreadGroup:0x00000000019bcb60>
=end
```

- 使用`ThreadGroup.new`可创建一个自定义的线程组  
- 使用`tg.add(t)`可将线程t加入线程组tg，这将会从原来的线程组移除t再加入新组tg  
- 使用`tg.list`可列出线程组tg中的所有线程  
- 使用`t.group`可获取线程t所属的线程组  
- 子线程会继承父线程的线程组，即子线程也会加入父线程所在的线程组  

```ruby
tg = ThreadGroup.new
t1 = Thread.new { Thread.stop }
t2 = Thread.new { Thread.stop }
tg.add t1
tg.add t2
pp tg.list
pp t1.group
=begin
[#<Thread:0x000000000196c480 a.rb:4 sleep_forever>,
 #<Thread:0x000000000196c3b8 a.rb:5 sleep_forever>]
#<ThreadGroup:0x000000000196c520>
=end
```

线程组还有一个功能：可使用`tg.enclose`封闭线程组tg，封闭后的线程组将不允许内部线程移出加入其它组，也不允许外界线程加入该组，只允许在该组中创建新线程。使用`tg.enclosed?`测试线程组tg是否已封闭。

其实，使用线程组可以将多个线程分类统一管理，线程组本质是一个线程数组加一些额外属性。比如，可以为线程组定义一些额外的针对线程组中所有线程的功能：wakeup组中的所有线程、join所有线程、kill所有线程。

```ruby
class ThreadGroup
  def wakeup
    list.each(&:wakeup)
  end
  def join
    list.each { |th| th.join if th != Thread.current }
  end
  def kill
    list.each(&:kill)
  end
end
```