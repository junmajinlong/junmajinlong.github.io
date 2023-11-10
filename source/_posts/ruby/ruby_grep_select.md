---
title: Ruby grep和select筛选元素
p: ruby/ruby_grep_select.md
date: 2020-05-29 22:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

在Enumerable模块中定义了grep()方法和select()方法，都可以用于筛选元素，还定义了它们元素取反的方法grep_v()和reject()。此外，select()还有别名方法find_all()和filter()。

但是grep()和select()是不同的，如果只考虑筛选元素，那么grep可以看作是select()的特例，grep采用`===`进行筛选，而select()根据给定的条件进行筛选。

例如：

```ruby
obj = ["a", "b", "c", 1, 2, 3, nil]
```
筛选出其中的字符串元素：

![](/img/ruby/1590848866418.png)

筛选出非空元素：

```ruby
obj.grep_v(NilClass)  #=> ["a", "b", "c", 1, 2, 3]
obj.select {|x| x!= nil}
```

可以根据正则表达式进行筛选，因为正则表达式定义了`===`运算：

```ruby
obj = %w(perl shell php python)
obj.grep(/^p/)     #=> ["perl", "php", "python"]
obj.select {|x| x =~ /^p/}
```

可以筛选在范围内的元素：

```ruby
nums = [9, 10, 11, 20]
nums.grep(5..10)      #=> [9, 10]
nums.select {|x| (5..10) === x}
```

grep()除了单纯的筛选，还允许跟一个语句块对筛选出的数据做处理，并返回处理后的结果。这等价于select + map的效果。

```ruby
nums = [9, 10, 11, 20]
nums.grep(5..10) {|n| n * 2 }  #=> [18, 20]
nums.select {|x| (5..10) === x}.map {|x| x * 2}
```

Ruby 2.7提供了filter_map()方法：

```ruby
nums.filter_map {|x| x * 2 if (5..10) === x }
#=> [18, 20]
```

