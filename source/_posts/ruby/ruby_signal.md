---
title: Ruby信号处理
p: ruby/ruby_signal.md
date: 2021-07-01 19:14:36
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby信号处理

## Ruby使用Process.kill发送信号

```
Process.kill(signal, pid, ...) → integer
```

`Process.kill`发送指定的信号给一个或多个进程或进程组：   
- 如果目标`pid>0`，表示发送信号给指定PID的进程  
- 如果目标`pid=0`，表示发送信号给调用kill的进程所在进程组的所有进程  
- 如果目标`pid<0`，表示按照操作系统的规则发送信号。对于Linux来说：  
  - 如果`pid=-1`，表示发送信号给除pid=1的init进程外的所有进程，当然，没有权限的进程将不受影响  
  - 如果`pid<-1`，表示发送信号给`-pid`所在进程组的所有进程，例如`-3000`表示发送信号给pid=3000的进程所在进程组的所有进程  

`Process.kill`的第一个参数是要发送的信号：  

- 信号可以是字符串格式的信号名或数值格式的信号ID，INT或SIGINT或1都是有效的信号  
- 如果信号带有负号(如`-2`或`-INT`)，表示发送信号给进程所在进程组而非指定的进程(Linux不支持带负号的信号)  
- 如果信号为0，表示探测是否能发送信号给目标进程，可探测是否能管理目标进程或者探测目标进程是否存活  

```ruby
pid = fork do
  sleep 300
end
# ...
Process.kill("HUP", pid)
Process.wait
```

## Ruby使用trap()设置信号处理程序

Ruby中使用`Kernel.trap`或`Signal.trap`捕获信号并设置信号处理程序，这两个trap等价。

可设置多个trap来监控多个信号。

```ruby
Signal.trap(0, proc { puts "Terminating: #{$$}" })
Signal.trap("CLD")  { puts "Child died" }
fork && Process.wait
=begin
Terminating: 27461
Child died
Terminating: 27460
=end
```

trap的第一个参数是监控的信号名称，可以是字符串的信号名称(如SIGINT)，可以是省略SIG前缀的信号名称(如INT)，可以是信号对应的数值(如2)。

Ruby支持一个特殊的信号0(对应的字符串信号名为EXIT或SIGEXIT)，表示进程退出时会触发的信号。

trap的第二个参数或语句块是捕获到信号后执行的代码。第二个参数有几种特殊情况：  
- 如果第二个参数为字符串`IGNORE`或`SIG_IGN`，表示忽略本次捕获的信号  
- 如果第二个参数为字符串`DEFAULT`或`SIG_DFL`，表示按照Ruby的默认处理规则来处理  
- 如果第二个参数为字符串`EXIT`，表示以退出状态码0退出当前进程  
- 如果第二个参数为字符串`SYSTEM_DEFAULT`，表示按照系统的默认信号处理规则来处理，即以退出状态码141退出进程

## 避免信号覆盖

使用第三方包的时候，有时候不知道这个包是否定义了某个信号的信号处理程序，或者知道它定义了某信号信号处理程序，但自己定义这个信号的信号处理程序时，不想覆盖第三方包中所定义的处理程序。

这时，应该利用好trap的返回值。每一次trap设置信号处理程序时，都返回本信号之前已经定义的信号处理程序(是一个Proc对象)。只是需要注意，有些信号的初始处理程序是一个字符串值`DEFAULT`而不是一个Proc对象，因此，应该进行类型判断：

```ruby
# 第一次定义INT的信号处理程序
first_trap = trap('INT') { 
  first_trap.call if first_trap.is_a? Proc
  puts "first_trap" 
}

# 第二次定义INT的信号处理程序
old_trap = trap('INT') { 
  old_trap.call if old_trap.is_a? Proc   # 调用第一次定义的信号处理程序
  puts "old trap"  # 本次trap时执行的逻辑
}
# 定义好之后，old_trap为第一次定义的信号处理程序

# 之后按下CTRL+C触发INT信号的信号处理程序
```

## 多线程信号注册问题

如果是在多线程中注册信号处理程序，该**信号处理程序将总是注册在所在进程的main线程中**(即使是在其它线程中设置`trap()`)。

```ruby
pid = fork do
  puts "main Thread: #{Thread.current}"
  Thread.new {
    puts "new Thread: #{Thread.current}"
    trap("TERM", proc { puts "Signal: #{Thread.current}" })
    sleep 2
  }
  sleep 2
end

sleep 1
Process.kill 'SIGTERM', pid
=begin
main Thread: #<Thread:0x00007fffd6ed4c10 run>
new Thread: #<Thread:0x00007fffd714f2b0@a.rb:4 run>
Signal: #<Thread:0x00007fffd6ed4c10 run>
=end
```

## 子进程继承信号处理程序

子进程会从父进程继承信号处理程序。
```ruby
trap 'TERM', proc { puts "Signal: #{Process.pid}" }
puts "Parent: #{Process.pid}"

pid = fork do
  sleep 30
end

puts "Child: #{pid}"
Process.kill 'TERM', pid
=begin
Parent: 2872
Child: 2901
Signal: 2901
=end
```

