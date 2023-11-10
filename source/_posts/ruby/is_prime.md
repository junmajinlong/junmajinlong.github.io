---
title: 效率比较：几种方法判断一个数是否是素数
p: ruby/is_prime.md
date: 2020-04-04 17:37:29
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------


# 法一：穷举判断一个数是否是素数

给定一个数n，从2开始自增x，判断n是否被x整除。

x最多自增到n的平方根即可，因为如果有大于n的平方根的值可整除n时，那么其商必定是小于n的平方根的值。

代码如下：

```ruby
def prime?(n)
  raise TypeError unless n.kind_of?(Integer)
  2.upto(Integer.sqrt(n)) {|x|
    return false if n % x == 0
  }
  return true
end
```

这里可以稍作改进，可以在自增时自动跳过偶数部分，这可以通过每次自增2实现：

```ruby
def prime?(n)
  raise TypeError unless n.kind_of?(Integer)
  return true if n == 2
  return false if n % 2 == 0
  3.step(Integer.sqrt(n), 2) {|x|
    return false if n % x == 0
  }
  return true
end
```
# 给定一个数求出所有小于它的素数

例如给定一个数20，返回所有小于它的素数，即`[2, 3, 5, 7, 11, 13, 17]`。

```ruby
def primes(n)
  arr = []
  2.upto(n) do |x|
    arr << x if prime?(x)
  end
  arr
end
```

# 法二：改进的求素数算法(6n+-1)

观察一下100以下的所有素数：

```
2,3,5,7,11,13,17,19,23,29,31,37,41,43,47,53,59,61,67,71,73,79,83,89,97
```

从5开始，所有的素数都是6的倍数的邻居：

```
5  = 6 * 1 - 1
7  = 6 * 1 + 1
11 = 6 * 2 - 1
13 = 6 * 2 + 1
17 = 6 * 3 - 1
19 = 6 * 3 + 1
23 = 6 * 4 - 1   # 25 = 6 * 4 + 1 不是素数
29 = 6 * 5 + 1
31 = 6 * 5 + 1
...
```

所以有以下结论：  
1. 表达式`(6n +/- 1)`之外的(2,3)是素数  
2. 从5开始的所有素数都满足表达式`(6n +/- 1)`，所有不满足该表达式的都不是素数  
3. 满足该表达式的不一定是素数  

改进的代码：

```ruby
def prime?(n)
  raise TypeError unless n.is_a? Integer
  # 2 或 3 是素数
  return true if n == 2 or n == 3
  
  # 不是6n +/- 1的数不是素数
  return false if n % 6 != 1 and n % 6 != 5
  
  # 满足6n +/- 1的数，还需进一步判断6n两边的数是否是小素数的倍数
  5.step(Integer.sqrt(n), 6) do |x|
    return false if n % x == 0 or n % (x+2) == 0
  end
  true
end
```

# 法三：继续改进的求素数算法

还可以继续对每次跳6步进行改进。

根据前面的描述，所有的素数都符合`6n +/- 1`模式，但其实也满足`30n +/- 1`或`30n +/- 7`或`30n +/- 11`或`30n +/- 13`模式，简记为`30n +/- (1,7,11,13)`。

例如对于如下素数，从7开始：

```
2,3,5,7,11,13,17,19,23,29,31,37,41,43,47,53,59,61,67,71,73,79,83,89,97

 7 = 30 * 0 + 7
11 = 30 * 0 + 11
13 = 30 * 0 + 13
17 = 30 * 1 - 13
19 = 30 * 1 - 11
23 = 30 * 1 - 7
29 = 30 * 1 - 1

31 = 30 * 1 + 1
37 = 30 * 1 + 7
41 = 30 * 1 + 11
43 = 30 * 1 + 13
47 = 30 * 2 - 13
49 = 30 * 2 - 11
53 = 30 * 2 - 7
59 = 30 * 2 - 1
...
```

所以：  
1. 表达式之外的`2,3,5`也是素数  
2. 不满足表达式`30n +/- (1,7,11,13)`的都不是素数  
3. 满足表达式`30n +/- (1,7,11,13)`的不一定是素数，还需判断该数是否能被这些小素数整除  

Ruby代码如下：  

![](/img/ruby/733013-20200405004020640-2043832703.jpg)

# 法四：Ruby官方的素数库

Ruby官方提供了一个名为`Prime`的库：https://ruby-doc.org/stdlib-2.6.5/libdoc/prime/rdoc/Prime.html

该库使用的是惰性生成器生成素数，所以效率并不高。至少，比上面介绍的两种改进方法要慢的多。

# 各种方法的效率测试

对法一、法二、法三、法四进行效率测试，比如生成400W以下的所有素数并加入到各自的数组中，比较其时间。

```ruby
# 每次跳过2
def prime2?(n)
  raise TypeError unless n.kind_of?(Integer)
  return true if n == 2
  3.step(Integer.sqrt(n), 2) {|x|
    return false if n % x == 0
  }
  return true
end

# 每次跳过6
def prime6?(n)
  raise TypeError unless n.is_a? Integer
  return true if n == 2 or n == 3
  return false if n % 6 != 1 or n % 6 != 5
  5.step(Integer.sqrt(n), 6) do |x|
    return false if n % x == 0 or n % (x+2) == 0
  end
  true
end

# 每次跳过30
def prime30?(n)
  raise TypeError unless n.is_a? Integer
  return true if n == 2 or n == 3 or n == 5
  case n % 30
  when 1, 7, 11, 13, 17, 19, 23, 29
    7.step(Integer.sqrt(n), 30) do |x|
      return false if n % x == 0 or n % (x + 4) == 0 or
                      n % (x + 6) == 0 or n % (x + 10) == 0 or
                      n % (x + 12) == 0 or n % (x + 16) == 0 or
                      n % (x + 22) == 0 or n % (x + 24) == 0
    end
    return true
  else
    false
  end
end

# 官方Prime库的prime?()
require 'prime'

require 'benchmark'

prime_official_arr = []
prime2_arr = []
prime6_arr = []
prime30_arr = []
n = 4_000_000

Benchmark.bm(15) do |bm|
  bm.report("official") do
    2.upto(n) do |x|
      prime_official_arr << x if Prime.prime?(x)
    end
  end
  
  bm.report("2n") do
    2.upto(n) do |x|
      prime2_arr << x if prime2?(x)
    end
  end
  
  bm.report("6n") do
    2.upto(n) do |x|
      prime6_arr << x if prime6?(x)
    end
  end
  
  bm.report("30n") do
    2.upto(n) do |x|
      prime30_arr << x if prime30?(x)
    end
  end
end

p prime_official_arr == prime2_arr 
p prime_official_arr == prime6_arr 
p prime_official_arr == prime30_arr
```

测试结果：

```
                      user     system      total        real
official         24.656250   0.000000  24.656250 ( 24.711801)
2n               10.515625   0.000000  10.515625 ( 10.526535)
6n                5.562500   0.000000   5.562500 (  5.578110)
30n               3.703125   0.015625   3.718750 (  3.713207)
true
true
true
```

可见，效率最差的是官方库提供的惰性生成模式的`Prime.prime?`，效率最好的是每次跳30步的。







