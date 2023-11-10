---
title: Ruby 3多线程并行：Ractor
p: ruby/ruby_ractor.md
date: 2021-05-04 19:14:36
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby 3多线程并行：Ractor

> Ruby 3 Ractor官方手册：<https://github.com/ruby/ruby/blob/master/doc/ractor.md>

在Ruby 3之前，使用Thread来创建新的线程，但这种方式创建的多线程是并发而非并行的，MRI有一个全局解释器锁GIL来控制同一时刻只能有一个线程在执行：

```ruby
# main Thread

t1 = Thread.new do 
  # new Thread
  sleep 3
end
t1.join
```

Ruby 3通过Ractor(Ruby Actor，Actor模型通过消息传递的方式来修改状态)支持真正的多线程并行，多个Ractor之间可并行独立运行。

```ruby
# main Ractor

# 创建一个可与main Ractor并行运行的Ractor
r = Ractor.new do
  sleep 2
  Ractor.yield "hello"
end

puts r.take
```

需注意，每个Ractor中至少有一个原生Ruby线程，但每个Ractor内部都拥有独立的GIL，使得Ractor内部在同一时刻最多只能有一个线程在运行。从这个角度来看，Ractor实际上是解释器线程，每个解释器线程拥有一个全局解释器锁。

如果main Ractor退出，则其他Ractor也会收到退出信号，就像main Thread退出时，其他Thread也会退出一样。

## 创建Ractor

使用`Ractor.new`创建一个Ractor实例，创建实例时需指定一个语句块，该语句块中的代码会在该Ractor中运行。

```ruby
r = Ractor.new do
  puts "new Ractor"
end
```

可在new方法的参数上为该Ractor实例指定名称：

```ruby
r = Ractor.new(name: "ractor1") do
  puts "new Ractor"
end

puts r.name  # ractor 1
```

new方法也可指定其他参数，这些参数必须在name参数之前，且这些参数将直接原样传递给语句块参数：

```ruby
arr = [11, 22, 33]
r = Ractor.new(arr, name: "r1") do |arr|
  puts "arr"
end
sleep 1
```

关于new的参数，稍后还会有解释。

可使用`Ractor.current`获取当前的Ractor实例，使用`Ractor.count`获取当前存活的Ractor实例数量。

## Ractor之间传递消息

Ractor传递消息的方式分两种：  

- Push方式：向某个特定的Ractor实例推送消息，可使用`r.send(Msg)`或别名`r << Msg`向该Ractor实例传送消息，并在该Ractor实例内部使用`Ractor.receive`或别名`Ractor.recv`或它们的同名私有方法来接收推送进来的消息  
  - Ractor还提供了`Ractor.receive_if {expr}`方法，表示只在expr为true时才接收消息，`receive`等价于`receive_if {true}`  
- Pull方式：从某个特定的Ractor实例拉取消息，可在该Ractor实例内部使用`Ractor.yield`向外传送消息，并在需要的地方使用`r.take`获取传输出来的消息  
  - `Ractor.new`的语句块返回值，相当于`Ractor.yield`，它也可被`r.take`接收  

因此，对于Push方式，要求知道消息传递的目标Ractor，对于Pull方式，要求知道消息的来源Ractor。

```ruby
# yield + take
r = Ractor.new {Ractor.yield "hello"}
puts r.take

# send + receive
r1 = Ractor.new do
  # Ractor.receive或Ractor.recv
  # 或同名私有方法：receive、recv
  puts Ractor.receive
end
r1.send("hello")
r1.take    # 本次take取得r1语句块的返回值，即puts的返回值nil
```

使用new方法创建Ractor实例时，可指定new的参数，这些参数会被原样传递给Ractor的语句块参数。

```ruby
arr = [11, 22, 33]
r = Ractor.new(arr) { |arr| ...}
```

实际上，new的参数等价于在Ractor语句块的开头使用了`Ractor.receive`接收消息：

```ruby
r = Ractor.new 'ok' { |msg| msg }
r.take #=> 'ok'

# 基本等价于
r = Ractor.new do
  msg = Ractor.receive
  msg
end
r.send 'ok'
r.take #=> 'ok'
```

### 消息端口

Ractor之间传递消息时，实际上是通过Ractor的消息端口进行传递的。

每个Ractor都有自己的incoming port和outgoing port：  

- incoming port：是该Ractor接收消息的端口，`r.send`和`Ractor.receive`使用该端口  
  - 每个incoming port都连接到一个大小不限的队列上  
  - `r.send`传入的消息都会写入该队列，由于该队列大小不限，因此`r.send`从不阻塞  
  - `Ractor.receive`从该队列弹出消息，当队列为空时，`Ractor.receive`被阻塞直到新消息出现  
  - 可使用`r.close_incoming`关闭incoming port，关闭该端口后，`r.send`将直接报错，`Ractor.receive`将先从队列中取数据，当队列为空后，再调用`Ractor.receive`将报错  
- outgoing port：是该Ractor向外传出消息的端口，`Ractor.yield`和`r.take`使用该端口  
  - `Ractor.yield`或Ractor语句块返回时，消息从outgoing port流出  
  - 当没有`r.take`接收消息时，r内部的`Ractor.yield`将被阻塞  
  - 当r内部没有`Ractor.yield`时，`r.take`将被阻塞  
  - `Ractor.yield`从outgoing port传出的消息可被任意多个`r.take`等待，但只有一个`r.take`可获取到该消息  
  - 可使用`r.close_outgoing`关闭outgoing port，关闭该端口后，再调用`r.take`和`Ractor.yield`将直接报错。如果`r.take`正被阻塞(等待`Ractor.yield`传出消息)，关闭outgoing port操作将取消所有等待中的take并报错  

### Ractor.select等待消息

可使用`Ractor.select(r1,r2,r3...)`等待一个或多个Ractor实例outgoing port上的消息(因此，select主要用于等待`Ractor.yield`的消息)，等待到第一个消息后立即返回。

`Ractor.select`的返回值格式为`[r, obj]`，其中：  

- r表示等待到的那个Ractor实例  
- obj表示接收到的消息对象  

例如：

```ruby
r1 = Ractor.new{'r1'}
r2 = Ractor.new{'r2'}
rs = [r1, r2]
as = []

# Wait for r1 or r2's Ractor.yield
r, obj = Ractor.select(*rs)
rs.delete(r)
as << obj

# Second try (rs only contain not-closed ractors)
r, obj = Ractor.select(*rs)
rs.delete(r)
as << obj
as.sort == ['r1', 'r2'] #=> true
```

通常来说，会使用`Ractor.select`来轮询等待多个Ractor实例的消息，通用化的处理流程参考如下：

```ruby
# 充当管道功能的Ractor：接收消息并发送出去，并不断循环
pipe = Ractor.new do
  loop do
    Ractor.yield Ractor.receive
  end
end

RN = 10
# rs变量保存了10个Ractor实例
# 每个Ractor实例都从管道pipe中取一次消息然后由本Ractor发送出去
rs = RN.times.map{|i|
  Ractor.new pipe, i do |pipe, i|
    msg = pipe.take
    msg # ping-pong
  end
}
# 向管道中发送10个数据
RN.times{|i| pipe << i}

# 轮询等待10个Ractor实例的outgoing port
# 每等待成功一次，从rs中删除所等待到的Ractor实例，
# 然后继续等待剩下的Ractor实例
RN.times.map{
  r, n = Ractor.select(*rs)
  rs.delete r
  n
}.sort #=> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

此外，`Ractor.select`除了可等待消息外，也可以用来yield传递消息，更多用法参考官方手册：[Ractor.select](https://ruby-doc.org/core-3.0.1/Ractor.html#method-c-select)。

## Ractor并行时如何避免竞态

多个Ractor之间是可并行运行的，为了避免Ractor之间传递数据时出现竞态问题，Ractor采取了一些措施：

- 对于不可变对象，它们可直接在Ractor之间共享，此时传递它们的引用  
- 对于可变对象，它们不可直接在Ractor之间共享，此时传递数据时，默认先按字节逐字节拷贝，然后后传递副本  
- 也可以显式指定移动数据，将某份数据从Ractor1移动到另一个Ractor2中，即转移数据的所有权(参考Rust的所有权规则)，转移所有权后，原始所有者Ractor中将无法再访问该数据  

### 传递可共享对象：传递引用

可共享的对象：自动传递它们的引用，效率高  
- 不可变对象可在Ractor之间直接共享(如Integer、symbol、true/false、nil)，如：  
  - `i=123`：i是可共享的  
  - `s="str".freeze`：s是可共享的  
  - `h={c: Object}.freeze`：h是可共享的，因为Object是一个类对象，类对象是可共享的
  - `a=[1,[2],3].freeze`：a不可共享，因为冻结后仍然包含可变的`[2]`  
- Class/Module对象，即类对象自身和模块对象自身是可共享的  
- Ractor对象自身是可共享的  

例如：

```ruby
i = 33
r = Ractor.new do
  m = recv
  puts m.object_id
end

r.send(i)  # 传递i
r.take     # 等待Ractor执行结束(语句块返回)
puts i.object_id  # i传递后仍然可用
=begin
67
67
=end
```

值得注意的是，Ractor对象是可共享的，因此可将某个Ractor实例传递给另一个Ractor实例。例如：

```ruby
pipe = Ractor.new do
  loop do
    Ractor.yield Ractor.receive
  end
end

RN = 10
rs = RN.times.map{|i|
  # pipe是一个Ractor实例，这里作为参数传递给其他的Ractor实例
  Ractor.new pipe, i do |pipe, i|
    pipe << i
  end
}

RN.times.map{
  pipe.take
}.sort #=> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### 传递不可共享对象：传递副本

绝大多数对象不是可直接共享的。在Ractor之间传递不可共享的对象时，默认会传递deep-copy后的副本，即按字节拷贝的方式拷贝该对象的每一个字节。这种方式效率较低。

例如：

```ruby
arr = [11, 22, 33]  # 数组是可变的，不可共享
r = Ractor.new do
  m = recv
  puts "copied: #{m.object_id}"
end

r.send(arr)  # 传递数组，此时将逐字节拷贝数组
r.take
puts "origin: #{arr.object_id}"

=begin
copied: 60
origin: 80
=end
```

从结果看，两个Ractor内的arr不是同一个对象。

需注意，对于全局唯一的对象来说(比如数值、nil、false、true、symbol)，逐字节拷贝时并不会拷贝它们。例如：

```ruby
arr = %i[lang action sub]
r = Ractor.new do
  m = recv
  puts "copied: #{m.object_id}, #{m[0].object_id}, #{m[1].object_id}"
end

r.send(arr)
r.take
puts "origin: #{arr.object_id}, #{arr[0].object_id}, #{arr[1].object_id}"

=begin
copied: 60, 80, 1046748
origin: 100, 80, 1046748
=end
```

注意，Thread对象无法拷贝，因此无法在Ractor之间传递。

### 转移数据所有权

还可以让`r.send(msg, move: true)`和`Ractor.yield(msg, move: true)`传递数据时，明确表示要移动而非拷贝数据，即转移数据的所有权(从原来的所有者Ractor实例转移到目标Ractor实例)。

无论是可共享还是不可共享的对象，都可以转移所有权，只不过转移可共享对象的所有权没有意义，因为转移之后，原所有者仍然拥有所有权。

因此，通常只对不可共享的数据来转移所有权，转移所有权后，原所有者将无法访问该数据。

```ruby
str = "hello"
puts str.object_id
r = Ractor.new do
  m = recv
  puts m.object_id
end

r.send(str, move: true)  # 转移str的所有权
r.take
#puts str.object_id  # 转移所有权后再访问str，将报错

=begin
60
80
=end
```

**值得注意的是，移动的本质是内存拷贝，它底层也一样是逐字节拷贝原始数据的过程，所以移动传递数据的效率和传递副本数据的效率是类似的。移动传递和传递副本的区别之处在于所有权，移动传递后，原所有者Ractor实例将无法访问该数据，而拷贝传递方式则允许原所有者访问**。

注意，Thread对象无法转移所有权，因此无法在Ractor之间传递。

### 不可共享变成可共享：Ractor.make_shareable

对于不可共享的数据obj，可通过`Ractor.make_shareable(obj)`方法将其转变为可共享的数据，默认转变的方式是逐层次地递归冻结obj。也可指定额外的参数`Ractor.make_shareable(obj, copy: true)`，此时将深拷贝obj得其副本，再让副本(逐层递归冻结)转变为可共享数据。

例如：

```ruby
arr = %w[lang action sub]
puts arr.object_id
r = Ractor.new do
  m = recv
  puts m.object_id
end

r.send(Ractor.make_shareable(arr))
r.take
puts arr.object_id
puts arr.frozen?
```

输出：

```
60
60
60
true
```

## 示例

工作者线程池：

```ruby
require 'prime'

pipe = Ractor.new do
  loop do
    Ractor.yield Ractor.receive
  end
end

N = 1000
RN = 10
workers = (1..RN).map do
  Ractor.new pipe do |pipe|
    while n = pipe.take
      Ractor.yield [n, n.prime?]
    end
  end
end

(1..N).each{|i|
  pipe << i
}

pp (1..N).map{
  _r, (n, b) = Ractor.select(*workers)
  [n, b]
}.sort_by{|(n, b)| n}

```

Pipeline：

```ruby
# pipeline with yield/take
r1 = Ractor.new do
  'r1'
end

r2 = Ractor.new r1 do |r1|
  r1.take + 'r2'
end

r3 = Ractor.new r2 do |r2|
  r2.take + 'r3'
end

p r3.take #=> 'r1r2r3'
```