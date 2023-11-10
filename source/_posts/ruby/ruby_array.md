---
title: Ruby中的数组
p: ruby/ruby_array.md
date: 2020-05-11 10:37:45
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------


# Ruby数组

Ruby中的数组是一个容器，可以存放任意数据类型的元素，因为**存放在数组中的不是这些元素本身，而是这些元素的引用**。当然，那些小对象(如Fixnum、nil对象、false/true对象)除外，因为它们比一个指针还要小，它们在Ruby中是直接存储数据本身而不是使用额外的指针去引用的。

注意，Array类中包含了Enumerable模块，所以Enumerable中的方法也都能使用，例如Enumerable中非常好用的reduce()方法。

## 创建数组

### 字面常量创建

```ruby
# 1.使用[xxx]方式创建
arr1 = ["Perl", "Python", "Ruby"]

# 2.空数组
arr = []

# 3.使用%w或%W可以省略引号和逗号，而使用空格分隔
# %w(小写)不解释内插表达式，%W(大写)解释内插
# 此时中括号可以换成其它符号，如%{}、%<>、%##
# 如果元素中要保留空格，使用反斜线转义
arr2 = %w[Perl Python Ruby]   # 3个元素
arr3 = %w<Perl Python\ Ruby>  # 2个元素

a = "Perl"
arr4 = %w(#{a} Python Ruby)   # #{a}不做变量替换，第一个元素为`\#{a}`
arr5 = %W(#{a} Python Ruby)   # #{a}会做变量替换

# 4.使用%i创建符号symbol数组
arr6 = %i(Perl Python Ruby)  # [:Perl, :Python, :Ruby]
```

创建arr时，允许解析表达式。例如，范围表达式、变量等。

```ruby
[-10...0, 0..10]
[-5...0, 0..5][0].to_a #=> [-5, -4, -3, -2, -1]

a=10;b=20
c=[a, b]
```

其中上面表示Range的`-5...0`和`0..5`区别在于前者不包含尾边界，即它是左闭右开的，而后者是左右都闭。

### Array.new()创建

Array类的new()方法可以创建数组。这样可以一次性划分好数组的内存空间，一定程度上避免了后续数组自动扩容时引起的内存空间划分开销。

```ruby
ary = Array.new    #=> []
Array.new(3)       #=> [nil, nil, nil]
Array.new(3, true) #=> [true, true, true]
```

当new()指定了第二个参数时，各初始化的元素将**指向同一个对象**。

```Ruby
arr=Array.new(3, "abc")
arr[1][0]="A"   #=> ["Abc", "Abc", "Abc"]
```

![](/img/ruby/1589464100634.png)

例如，初始化互不影响的3个『abc』元素数组。

```ruby
arr = Array.new(3) {"abc"}
arr[0][1]="A"
p arr          # ["aAc", "abc", "abc"]
```

多维数组：

```ruby
Array.new(3){Array.new(2)} #=> [[nil, nil], [nil, nil], [nil, nil]]
```

### Array()创建

通过Kernel模块提供的`Array(arg)`方法创建数组。这实际上是将对象转换成Array对象，它的内部逻辑如下：
- 如果arg对象定义了to_ary，则调用该方法取得数组
- 如果没定义to_ary，但定义了to_a，则通过to_a取得数组
- 如果to_ary和to_a都没有定义，则直接将arg当成整体封装到数组的一个元素中

```ruby
{:a => "a", :b => "b"}.to_a   #=> [[:a, "a"], [:b, "b"]]
Array({:a => "a", :b => "b"}) #=> [[:a, "a"], [:b, "b"]]
Array(1..5)  #=> [1, 2, 3, 4, 5]
```

关于它的内部逻辑，稍做演示。例如，String类既没有to_ary，也没有to_a方法：
```ruby
"a".methods.grep(/to_a|to_ary/)  #=> []

mystr = "hello world"
Array(mystr)    #=> ["hello world"]


def mystr.to_a
  split(/ /)
end
Array(mystr)  #=> ["hello", "world"]
```

### to_a()创建

如果某个类定义了to_a()方法，可以将对象转换成数组结构。例如，hash类有to_a()方法，所以可以将hash结构转换成数组。

```ruby
my_hash = {Perl: "Larry Wall", Ruby: "Matz"}
arr = my_hash.to_a()  # [[:Perl, "Larry Wall"],[:Ruby, "Matz"]]
```

### split()从字符串创建

字符串对象有split()方法，可以按照指定的分隔符将字符串切割成数组。

```ruby
p "perl python ruby".split  # ["perl", "python", "ruby"]
```

## 访问数组元素

### 索引/slice取元素

获取数组的方式有多种。列出了以下几种常见的(返回多个元素的方式将以新数组的方式返回)：

- arr[1]或arr.at(n)或arr.slice(n)：获取index=1的元素
- arr[1..3]或arr.slice(1..3)：获取index=1、2、3的3个元素
- arr[1...3]或arr.slice(1...3)：获取index=1、2的两个元素
- arr[1..]：Range中的`1..`或`1...`或`1..nil`等均表示无限，即取index=1开始的所有数组元素
- arr[2, 3]或arr.slice(2, 3)：获取从index=2开始的3个元素，即index=2、3、4的元素
- arr[-1]：获取最后一个元素
- arr[-3]：获取倒数第三个元素

由于`arr[]`提供的功能已经足够丰富，所以at()方法和slice()方法很少用。

如果索引越界，则返回nil。但是有一个特殊情况，当使用slice的操作时，因为要返回的是数组，所以有如下"异常"情况：

```ruby
a = %w(a b c d e)
a[4]       #=>"e"
a[4,0]     #=>[]
a[4,1]     #=>["e"]
a[4..100]  #=>["e"]

a[5]       #=>nil
a[5,0]     #=>[]
a[5,1]     #=>[]
a[5..100]  #=>[]

a[6]       #=>nil
a[6,0]     #=>nil
a[6,1]     #=>nil
```

上面a[5]返回nil，但a[5,x]返回空数组，而a[6]以及a[6,x]则直接返回nil。首先a[6]以及a[6,x]都是索引越界，所以返回nil。而a[5]的5是索引越界，所以返回nil。问题是a[5,x]返回的是空数组，而不是nil。这是因为a[5,x]是一个slice操作，它要求返回新数组。从下面的slice操作可以理解为什么a[5,1]返回空数组。

```ruby
p a[0, 5]   # ["a", "b", "c", "d", "e"]
p a[1, 4]   # ["b", "c", "d", "e"]
p a[2, 3]   # ["c", "d", "e"]
p a[3, 2]   # ["d", "e"]
p a[4, 1]   # ["e"]
p a[5, 0]   # []
p a[5, 1]   # []
```

也就是说，对于slice操作，arr[len, x]的arr[len]可以认为是最边缘的元素，尽管arr[len]已经越界了。

其实`slice()`是按规则删除元素，只不过它不影响原始数组。而`slice!()`则是删除一些元素并直接影响原始数组，也就是在原处修改数组。

```ruby
a = [1, 2, 3, 4]
a.slice!(1,2)  # 返回:[2, 3]，a变成[1, 4]
```

因为通过索引方式取值时，索引越界时默认会返回nil。因此还定义了另一个方法fetch，它会在索引越界时直接报错或指定默认返回信息：
```ruby
arr = [11,22,33,44]
arr.fetch(-1)          #=> 44
arr.fetch(3)           #=> 44
arr.fetch(4)           # 报错
arr.fetch(4,"cat")     #=> "cat"
arr.fetch(4) {|idx| puts "index #{idx} out of bounds"}
#=>index 4 out of bounds
```

### Array.values_at()取分散元素

Array类提供了values_at()方法，可以取得每个数组对象中指定索引处的多个分散元素。

例如：

```ruby
a = %w(a b c d e)
p a.values_at(1)          # ["b"]
p a.values_at(0, 0, 2, 4) # ["a", "a", "c", "e"]
p a.values_at(1, 2, 10)   # ["b", "c", nil]
p a.values_at(1..3)       # ["b", "c", "d"]
```

除了这种方式可以取得分散元素，还可以使用map()函数进行操作。例如，等价于`values_at(0,0,2,4,10)`的写法为：

```ruby
p [0, 0, 2, 4, 10].map{|i| a[i]}
```

### 其它一些方法

比如，默认情况下索引越界时将返回nil，使用fetch()方法可以获取指定索引的元素，且可以指定越界时的默认值，否则直接报错。

```ruby
a=%w(a b c d e)
p a.fetch(10)  # 报错：索引越界IndexError
p a.fetch(10, "ten") # 指定默认值
```

使用first()和last()可以取得数组中的第一个元素和最后一个元素，它们会取得元素后立即退出。

```ruby
a = %w(a b c d e)
p a.first     # "a"
p a.last      # "e"
```

使用take()可以取得数组中的前n个元素，使用drop()可以取得除了数组前n个元素外剩下的元素。它们都不会删除元素。因为不改变原数组，所以它们的时间复杂度都是O(n)，其中n为所返回元素个数。

```ruby
a = %w(a b c d e f)
p a.take(2)  # ["a", "b"]
p a.drop(2)  # ["c", "d", "e", "f"]
p a          # ["a", "b", "c", "d", "e", "f"]
```

## 检索数组

有这么几个方法：filter()、filter!()、select()、select!()、reject()、reject!()、keep_if()、delete_if()。看起来很多，其实都有等价关系：
- filter()、select()等价：筛选满足条件的
- filter!()、select!()、keep_if()等价：留下满足条件的，不满足条件的删掉
- reject!()、delete_if()等价：满足条件的删掉，留下不满足条件的

它们返回的都是数组，只不过一些是原地修改，一些是返回新的数组对象。

> 复杂的三角恋
> 在array中，filter和select等价，在Enumerable中也有filter、select和find_all，它们等价。 
> 此外，Enumerable中的find是find_all的单元素版，等价于Enumerable的detect。  


```ruby
arr = [1, 2, 3, 4, 5, 6]
new_arr = arr.select {|a| a > 3}
p new_arr      # [4, 5, 6]
p arr          # [1, 2, 3, 4, 5, 6]

new_arr = arr.select! {|a| a > 3}
p new_arr      # [4, 5, 6]
p arr          # [4, 5, 6]
```

```
arr = [1, 2, 3, 4, 5, 6]
new_arr = arr.reject {|a| a > 3}
p new_arr      # [1, 2, 3]
p arr          # [1, 2, 3, 4, 5, 6]

new_arr = arr.reject! {|a| a > 3}
p new_arr      # [1, 2, 3]
p arr          # [1, 2, 3]
```


## 为数组元素赋值

`[] at slice`方法都可以为数组中指定位置处元素赋值。唯一需要注意的是正负数索引位置的区别：
- 正索引：表示index=X(X>=0)元素后面的位置
- 负索引：表示index=Y(Y<0)元素前面的位置

```ruby
a = %w(a b c d e)

# 单元素赋值
a[1] = "bb"  # 为第2个元素赋值

# 范围赋值
a[1, 2] = ["bb", "cc"]  # a=%w(a bb cc d e)
a[1, 2] = %w(bb)  # a=%w(a bb d e)
a[1, 2] = %w(bb cc dd)  # a=%w(a bb cc dd d e)

# 完全插入元素：指定len为0
a[1, 0] = %w(B C) # a=%w(a B C b c d e)
a[-1,0] = "f"     # 不是插入在数组尾部，而是倒数第二个位置

# 范围赋值替换
a[1..1] = "e"     # 替换第二个元素
a[-1..-1] = "e"   # 替换尾部元素

# 完全删除元素：将右边对象设置为空数组
a[1, 2] = []  # a=%w(a d e)
```

赋值语句的返回值是所赋值内容。例如`a[1,0] = %w[B C]`的返回值是`%w[B C]`，于此同时原始数组被修改。

需要注意，当使用数组子集赋值(即使用范围索引`arr[x..y]`或`arr[x,y]`)时，等号右边的内容将展开对各个范围内的元素进行赋值。但如果是对单个元素赋值，则是将等号右边的整个对象赋值给该元素。从数组元素保存的是引用去考虑这方面，很容易理解。
```ruby
arr = [1,2,3,4,5]      #=> [1, 2, 3, 4, 5]
arr[1..3] = [22,33,44]
arr                    #=> [1, 22, 33, 44, 5]

arr[1,3] = [2,3,4]
arr                    #=> [1, 2, 3, 4, 5]
arr[1,1] = [22,33]
arr                    #=> [1, 22, 33, 3, 4, 5]

# 注意，引用了整个数组
arr[1] = [222,333]     #=> [222, 333, 444]
arr                    #=> [1, [222, 333], 33, 4, 5]
```

## 扩充数组对象

数组可以执行`+ - * & |`操作，不过`- & |`是将数组作为集合进行操作的，而使用`+`和`*`可以扩展数组。

因为ruby中的一元运算符操作`x += y`和二元运算符操作`x = x + y`完全等价，都会创建新对象x。所以，涉及到大数组时，效率会比较低下。

`+`可以将两数组加在一起返回一个新数组：

```ruby
arr = [1, 2, 3]
arr1 = arr + [3, 4, 5]  # arr不变，arr1=[1,2,3,3,4,5]
```

`concat()`方法也能将0到多个数组扩展到一个数组上。它是原地修改的，所以时间复杂度是O(n)，n为追加部分的元素个数。注意，concat()会将参数数组全都"压平"然后追加到原数组尾部：

```ruby
["a", "b"].concat(["c", "d"])   # ["a", "b", "c", "d"]
["a"].concat(["b"], ["c", "d"]) # ["a", "b", "c", "d"]
["a"].concat     # ["a"]

a = [1, 2, 3]
a.concat([4, 5])  # a=[1, 2, 3, 4, 5]

a = [1, 2]
a.concat(a, a)   # a=[1, 2, 1, 2, 1, 2]
```


`*`对于数组有两种用法：

```ruby
# 1.数组乘一个数值对象，返回新数组
# 数值必须在数组后面，不能放前面
arr = [1, 2, 3]
arr1 = arr * 2  # arr不变，arr1=[1,2,3,1,2,3]

# 2.数组乘一个字符串，将返回字符串而非新数组
# 相当于join方法，所以可用于分隔各元素
arr = [1, 2, 3]
arr1 = arr * ","   # arr不变，arr1="1,2,3"
arr2 = arr * ",+"  # arr不变，arr2="1,+2,+3"
```

如果只想分隔除最后一个元素外的其它元素，并单独处理最后一个元素。可参考如下：
```ruby
arr = %w[perl php shell ruby]
arr[0..-2].join(", ") + " and " + arr[-1]
```


## 数组信息和数组测试

获取数组长度，可以使用count()、length()或size()方法，后两者等价。而count()方法通过参数或语句块的方式还能获取满足条件的元素个数。

```ruby
a = %w(a b c d e)
p a.length      # 5
p a.count       # 5
p a.size        # 5

ary = [1, 2, 4, 2]
ary.count                  # 4，数组元素个数
ary.count(2)               # 2，等于2的元素个数
ary.count {|x| x%2 == 0}   # 3，偶元素个数
```

检查数组是否为空数组，使用`empty?()`方法：

```ruby
p a.empty?      # false
```

检查数组是否包含某元素，使用`include?()`方法：

```ruby
p a.include?('c')  # true
p a.include?('x')  # false
```

## 向数组中增、删元素

### 插入元素

push()/append()、unshift()/prepend()、insert()或特殊符号`<<`，注意它们都是原处修改数组对象的：

```ruby
# push()向尾部插入元素
# push()返回数组自身，可以链式插入
# append()等价于push()
arr = [1, 2, 3, 4]
arr.push(5)   # arr = [1, 2, 3, 4, 5]
arr.push(6).push(7) #  arr = [1,2,3,4,5,6,7]

# <<向尾部插入元素
# <<返回数组自身，所以可以进行链式插入
arr = [1, 2, 3, 4]
arr << 5          # arr = [1, 2, 3, 4, 5]
arr << 6 << 7     # arr = [1,2,3,4,5,6,7]
arr <<8 <<[9, 10] # [1,2,3,4,5,6,7,8,[9,10]]

# unshift()向头部插入元素
# prepend()等价于unshift()
arr = [1, 2, 3, 4]
arr.unshift(0).unshift(-1) # arr=[-1,0,1,2,3,4]

# insert()向给定索引位置处插入元素，可一次性插入多个
arr = [0, 1, 2, 3, 4]
arr.insert(3, "Ruby") # [0, 1, 2, 'Ruby', 3, 4]
arr.insert(3,"Perl","Shell") # # [0,1,2,'Perl','Shell','Ruby',3,4]

# 通过范围赋值的方式插入元素
arr = [0, 1, 2, 3, 4]
arr[1,0] = [11, 22]  # [0, 11, 22, 1, 2, 3, 4]
arr[1..1] = [11, 22]  # 与上等价
```

对于大数组(或未来可能会变大的数组)来说，不要使用insert和unshift插入元素，因为它们会伴随着元素的迁移。而应该使用`<<`或`push`或`append`在尾部插入元素。

### 删除元素

**按位置删除元素**

pop()、shift()、delete_at()：

```ruby
# pop()移除数组尾部元素并返回该元素
arr = [1, 2, 3, 4, 5, 6]
arr.pop   # 返回6， arr变成[1, 2, 3, 4, 5]

# shift()从数组头部移除一个元素并返回该元素
arr.shift     # 返回1，arr变成[2, 3, 4, 5]

# delete_at()移除给定索引位置处的元素并返回该元素
arr.delete_at(2)  # 返回4，arr变成[2, 3, 5]

# 通过范围赋值的方式删除元素
arr = [1, 2, 3, 4, 5, 6]
arr[1, 2] = []   # arr = [1, 4, 5, 6]
```

对于大数组(或未来可能会变大的数组)来说，不要使用delete_at和shift删除元素，因为它们会伴随着元素的迁移。而应该使用pop删除尾部元素。

**按给定值删除元素**

delete()：找到元素则删除，找不到元素则返回nil，如果给了语句块，则在且仅在找不到元素时执行语句块  

```ruby
# delete()删除数组中等于某个值的元素
arr1=[1, 2, 2, 4]
arr2 = %w(Perl Shell Python Ruby)
arr1.delete(2)       # 返回2，arr1变成[1,4]
arr2.delete "Shell"  # 返回"Shell"，arr2变成%w(Perl Python Ruby)
arr2.delete("PHP") {"element not found"}
```

**删除重复元素**

uniq()、compact()以及成对的带感叹号后缀的uniq!()、compact!()：

```ruby
# uniq()删除重复元素，不是原处修改对象的
arr=%w(a b c x y a c b y)
uniq_arr = arr.uniq   # uniq_arr=%w(a b c x y)，arr不变

# uniq!()删除重复元素，直接修改原始对象且返回修改后的数组
arr=%w(a b c x y a c b y)
uniq_arr = arr.uniq!   # uniq_arr和arr都变成%w(a b c x y)
```

```ruby
# compact()删除所有nil元素，不是原处修改对象
arr = ["a", "b", "c", nil, "a", "c", nil, "b"]
compact_arr = arr.compact
p compact_arr    # ["a", "b", "c", "a", "c", "b"]
p arr            # ["a","b","c",nil,"a","c",nil,"b"]

# compact!()删除所有nil元素，原处修改对象
arr = ["a", "b", "c", nil, "a", "c", nil, "b"]
compact_arr = arr.compact!
p compact_arr    # ["a", "b", "c", "a", "c", "b"]
p arr            # ["a", "b", "c", "a", "c", "b"]
```

**清空数组**

clear()直接清空数组：

```ruby
a = [1,2,3]
a.clear()    # a=[]
```

clear()的内部行为是将数组的start元素索引和元素数量total设置为0。

直接将空数组赋值给变量`arr=[]`也可以清空数组，但其内部过程是创建一个空数组对象并让变量指向该空数组。

## 迭代数组

迭代方式有很多，通过for、while、times、数组自带的`each、each_index、reverse_each`，还有Mix-in Enumerator后获取的迭代方式：`each_cons、each_slice、each_entry、each_with_index、with_index、each_with_object`等。

此处仅简单介绍for/while/times迭代和数组自身几个迭代方法，关于从Enumerator获取的相关的迭代方式，参见对应文章：[Enumerator各种迭代方法](/ruby/ruby_iterators_in_enum)。

- [for迭代](#blogfor)
- [while迭代](#blogwhile)
- [times迭代](#blogtimes)

**数组自身迭代方法**：

- [each](#blogeach)：迭代各元素  
- [each_index](#blogeachindex)：只迭代各元素索引  
- [reverse_each](#blogreverseeach)：反序迭代各元素  

<a name="blogfor"></a>
<a name="blogwhile"></a>
<a name="blogtimes"></a>

### for、while、times迭代

```ruby
# for
x = %w(a b c d e)

for i in x
  puts "element: #{i}"
end

# while
x = %w(a b c d e)

i=0
while i < x.length
  puts "element: #{x[i]}"
  i += 1
end

# times
x = %w(a b c d e)

x.length.times do |n|
  puts "element: #{x[n]}"
end
```

<a name="blogeach"></a>

### each()

```ruby
each {|item| block} → ary
each → Enumerator
```

迭代数组中每个元素，并将每个元素传递给语句块中的变量。最后返回数组自身。

```ruby
arr = [1, 2, 3, 4, 5]
arr.each {|a| print a -= 10, " "}
## 输出：-9 -8 -7 -6 -5
## arr=[1, 2, 3, 4, 5]
```

注意，Ruby中是按引用赋值的，所以each迭代(也包括for，for的本质是each)时控制变量总是引用数组原元素，也因此，修改控制变量的值会直接修改原数组元素，除非这个元素是不可变对象。

例如，上面示例中数组的元素全部是数值，数值是不可变对象，所以修改控制变量a后不会影响原数组元素。但如果元素是可变对象，则会直接修改原数据：

```ruby
a=%w[perl shell python php ruby]
a.each {|x| x[1]="L" }
puts a
# 输出：
=begin
pLrl
sLell
pLthon
pLp
rLby
=end
```

<a name="blogeachindex"></a>

### each_index()

```ruby
each_index {|index| block} → ary
each_index → Enumerator
```

迭代数组中的每个元素，并将每个元素的索引传递给语句块中的变量。最后返回数组自身。

```ruby
a = ["a", "b", "c", "d"]

a.each_index do |x|
  puts "index: #{x}"
end

## 输出结果：
=begin
index: 0
index: 1
index: 2
index: 3
=end
```

<a name="blogreverseeach"></a>

### reverse_each()

```ruby
# reverse_each反序迭代数组
arr = [1, 2, 3, 4, 5]
arr.reverse_each {|a| print "#{a}-"}
## 输出5-4-3-2-1-
```

## 数组转换和测试

类方法：
- Array.try_convert(arg)：尝试将arg转换成数组，如果能转换则返回转换后的数组对象，否则返回nil。可以用它来测试参数对象是否是数组
- to_a()：返回self，对于数组自身来说，和to_ary()基本没区别
- to_ary()：返回self，对于数组自身来说，和to_a()基本没区别
- to_s()：将数组转换成字符串格式
- to_h()：将数组转换成hash类型
- inspect()：(对于数组来说)等价于to_s()

```ruby
# try_convert(arg)将转换成数组
arr = Array.try_convert([1])    # [1]
arr1 = Array.try_convert(1)     # nil
arr2 = Array.try_convert("1")   # nil

# 测试元素是否是数组、字符串
if tmp=Array.try_convert(arg)
  # arg is an array
elsif tmp = String.try_convert(arg)
  # arg is a string
end

# 更标准的测试数据类型
case arr
when String
  puts "String"
when Array
  puts "Array"
end

# 或者
arr.is_a? Array
arr.kind_of? Array
Array === arr
```

```ruby
# to_a()和to_ary()，都返回self
# 表示返回当前数组的引用
# 对于数组自身来说，这两者没差别
arr = ["foo", "bar"]
arr1 = arr.to_a

p arr.eql?(arr1)    # true
p arr.object_id     # 42285320
p arr1.object_id    # 42285320
```
```ruby
# to_s()等价于inspect()
# 将数组转换成字符串
arr = ["foo", "bar"]
arr.to_s   # [\"foo\", \"bar\"]
```

```ruby
# to_h()将数组转换成hash
h1 = [["foo",1], ["bar", 2]].to_h

# to_h的语句块形式，每个元素都将作为块中的变量
h2 = [["foo", 1],["bar", 2]].to_h {|x| x}
h3 =  ["foo", "bar"].to_h {|s| [s, s * 2] }

p h1   # {"foo"=>1, "bar"=>2}
p h2   # {"foo"=>1, "bar"=>2}
p h3   # {"foo"=>"foofoo", "bar"=>"barbar"}
```

## 数组大小、等同的比较

四种比较方式：等值比较`==`、hash值比较`eql?`、大小比较`<=>`和是否同一数组对象的比较`equal?()`。

数组还继承了Object类的`===`，对于数组而言，它等价于`==`。

### ==

只有两数组长度相同、两数组对应索引位置的元素使用`==`测试也相等，才判定两数组相等。

返回true/false。注意，如果含有nil元素，则nil与nil的比较结果为true。

```ruby
[ "a", "c" ]    == [ "a", "c", 7 ]     # false
[ "a", "c", 7 ] == [ "a", "d", "f" ]   # false
[ "a", "c", 7 ] == [ "a", "c", 7 ]     # true
["a", 1, 1.2] == ["a", 1.0, 1.2]       # true
```

### eql?()

只有两数组对象的内容完全相同时返回true，否则返回false。

它是根据各对象计算出的hash值比较的，一般来说比`==`要严格一点。

```ruby
[1, 2] == [1, 2.0]      # true
[1, 2].eql?([1, 2.0])   # false
[1, 2].eql?([1, 2])     # true

[1, 2].hash    # 2027605168499122706
[1, 2.0].hash  # 3393919734812826294
```

### equal?()

继承自Object类，它比较的是两者是否是同一对象。这个方法基本上不会被子类重写。

这个方法比`==`和`eql?()`都要严格。

```ruby
[1,2].equal?([1,2])  # false

a = [1, 2]
b = a
a.equal?(b)  # true
```

### <=>

这个比较符号，当执行的是`obj1 <=> obj2`时：

- obj1大于obj2时，该比较表达式返回1
- obj1等于obj2时，该比较表达式返回0
- obj1小于obj2时，该比较表达式返回-1

对于数组而言，对数组中的每个元素都一一对应的比较，直到找到某个数组中的元素不等于另一个数组中对应索引处的元素就返回结果。如果数组长短不一致，且较短数组的所有元素都和另一数组对应元素相等，则长数组更大。于是可以推断，只有两数组长度、内容完全相等(通过`==`号比较，例如1等于1.0)，数组才相等。

```ruby
arr = [1, 3, 5, 7]
arr1 = [1, 3.0, 5, 7]
arr2 = [1, 3, 5]
arr3 = [1, 3, 5, 7, 9]
arr4 = [1, 3, 7, 5]
arr5 = [1, 3, 7, 5, nil]

p arr <=> arr1   # 0
p arr <=> arr2   # 1
p arr <=> arr3   # -1
p arr <=> arr4   # -1
p arr <=> arr5   # -1
```

## 数组元素排序

数组Mix-in了Enumerable模块，它具备`sort`和`sort_by`两个方法，且还具备对应的原地修改的排序方法。

```ruby
arr = %w[shell python php c ruby go perl]

# 按字符数量比较
arr.sort {|x,y| x.size <=> y.size}
#=> ["c", "go", "php", "ruby", "perl", "shell", "python"]

# 按字符数量比较
arr.sort_by {|x| x.size}
#=> ["c", "go", "php", "ruby", "perl", "shell", "python"]

# 多个排序依据
arr.sort_by {|x| [x.size, x]}
#=> ["c", "go", "php", "ruby", "shell", "python"]
```

## 打乱数组

array对象可以使用`shuffle`和`shuffle!`方法打乱数组顺序：

```ruby
a = [1,2,3,4,5]
a.shuffle      #=> [1, 5, 4, 3, 2]
```

也可以自己定义一个方法：
```ruby
class Array
  def random_array()
    self.sort_by { rand }
  end
end
```
