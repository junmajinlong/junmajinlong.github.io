---
title: Ruby循环语句
p: ruby/ruby_loop.md
date: 2020-05-11 10:37:43
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby中的循环语句

![](/img/ruby/1589464826174.png)

## times方法循环

times是Integer类的一个方法，它后面跟一个语句块，如果没有给定语句块则返回一个迭代器对象。

```
times {|i| block } → self
times → an_enumerator
```

当明确循环次数的时候，使用times方法进行循环将变得非常实用，代码可读性也非常强。

例如，要循环10次：

```ruby
10.times {
  puts "循环输出10次"
}
```

每次循环的时候，可以将次数作为语句块的变量使用(变量的值从0开始)：

```ruby
10.times do |i|
  puts "这是第#{i}次循环"
end
```

## for..in循环

这种for..in循环是迭代测试性循环，从一个容器中检测是否存在下一个元素，存在则迭代。for内部使用的是each方法，for是一个语法糖类型的语法式循环语句：

```ruby
for i in 1..5
  puts i
end

langs = %w(perl php shell python)
for lang in langs do
  puts lang
end
```

for和while/until的关键字do可以替换为换行符或分号，例如下面是等价的：

```ruby
for i in 1..5
  puts i
end

for i in 1..5 do
  puts i
end

for i in 1..5; puts i; end
for i in 1..5 do puts i; end
```

## while和until循环

```ruby
# while循环
a = 0
while a < 10 do
  p a
  a += 1
end

p a        # 输出10

# until循环
a=0
until a>10 do
  p a         # 输出0到10
  a+=1
end

p a    # 输出11
```

## while和until作为修饰符的循环

将while和until作为修饰符，简化简单的循环代码：

```ruby
a = 0
a += 1 while a < 10
p a # prints 10

a = 0
a += 1 until a > 10
p a # prints 11
```

如果while或until前面有多条语句，则可以使用begin...end语句块：

```ruby
a=0
begin
  p a
  a+=1
end while a<10
```

只是需要注意的是，while和until作为修饰符的时候，当使用了`begin...end`语句块后，循环体一定会执行一次，然后才开始第一次的条件判断。但如果不使用`begin...end`，则不会出现这样的问题。

```ruby
x = 10
puts "hello" while x < 10  # 不输出

y = 10
begin puts "world" end while y < 10  # 输出
```


## loop无限循环

loop方法是Kernel模块中提供的一个用来做无限循环的方法：

```ruby
loop {
  p "hello world"
}
```

可以使用break来控制loop循环的退出。

## each迭代循环

each方法用来迭代任何可迭代的对象，它的根在Enumerable模块，只要某个类实现了each方法用来yield元素，那么只要mix-in Enumerable之后就可以自动获得Enumerable中一大堆的each变体方法，如`each_with_index`迭代方法。

```ruby
sum=0
(1..5).each do |i|
  sum += i
end
p sum
```

## break、next、redo

它们可以用在各种迭代、循环语句块中用来控制迭代、循环的流程。

- break退出当前层次的迭代结构、循环结构  
- next立即进入下一轮迭代或循环(也可以认为是跳到当前循环的尾部从而进入下一轮循环)  
- redo立即回到当前迭代/循环的头部，再次执行一次迭代/循环语句块  

需要注意的是，break、next、redo除了可用于循环式的语句块，还可用于普通的语句块：  

- break：等价于return，跳出当前语句块且跳出语句块所在函数  
- next：跳过语句块，但不跳出函数  
- redo：重新执行语句块  

例如：

```ruby
def f
  yield
  puts "after block"
end

f { puts "in block: before"; break; puts "in block: after" } 
#=> 输出"in block: before"
f { puts "in block: before"; next; puts "in block: after" }
#=> 输出"in block: before"和"after block"
f { puts "in block: before"; redo; puts "in block: after" }
#=> 无限输出"in block: before"
```


## 几种循环的效率比较

只考虑速度不考虑内存的话。经过比较，while(until)是最快的。

```bash
# while条件循环
$ time ruby -e 'a=0;i=0;while(i<100000000) do a = a+1;i = i+1; end;p a'
100000000

real    0m1.770s
user    0m1.563s
sys     0m0.219s

# for i in range
$ time ruby -e 'a=0;for i in 1..100000000 do a = a+1 end;p a'
100000000

real    0m3.404s
user    0m3.395s
sys     0m0.012s

# N.times
$ time ruby -e 'a=0;100000000.times do a = a+1 end;p a'
100000000

real    0m3.230s
user    0m2.984s
sys     0m0.266s

# range.each
$ time ruby -e 'a=0;(1..100000000).each do a = a+1 end;p a'
100000000

real    0m3.180s
user    0m2.953s
sys     0m0.219s
```

## Ruby中的循环优化

在循环次数多时，循环内部的代码优化对循环效率的提升是有益的。

**1.循环中尽量不要重复创建重复的对象**。

在Ruby中，数值、Symbol、true、false、nil都是不可变对象，循环中使用它们的常量形式其实每次都在引用同一个对象。

```ruby
$ ruby -e '5.times {puts 144444444423.__id__}'
288888888847
288888888847
288888888847
288888888847
288888888847
```

而Ruby中的字符串、数组、hash是可变对象，在循环中每次使用它们的常量形式都会重新创建新对象，这会使得效率降低。此时应将常量字符串、数组、hash定义成变量保存在循环的外部。

```ruby
$ ruby -e '5.times {puts "a".__id__}'
70368305866240
70368305866120
70368305866060
70368305866000
70368305865940

$ ruby -e 'a="a"; 5.times {puts a.__id__}'
70368545646060
70368545646060
70368545646060
70368545646060
70368545646060
```

此外，正则表达式的**常量形式**在循环中默认是不会重新编译的对象(除非使用了变量内插方式来构建常量的正则表达式对象，此时它是动态的，但可以使用『o』修饰符来强制缓存第一次的编译结果从而不再重新编译)，所以循环中也可以使用正则的常量形式。但如果是**Regexp类方法构建**的正则表达式对象，则每次都会重新构建。

例如，下面两种循环，实现的需求是一样的，但是用了两种不同的方法：

```ruby
words.each {|w| next if w.include?("abc")}
words.each {|w| next if w =~ /abc/}
```

上面两个循环语句都是从一个words数组中匹配是否包含abc字符串。但是在第一个循环中，『abc』以常量的方式存在，每次迭代时都重新创建、销毁这个字符串对象，而第二个循环中只构建了一次正则表达式对象。所以，虽然`include?()`的检测比正则表达式的匹配性能更好，但是结果却是第二个循环表现更佳，特别是循环次数多的时候。