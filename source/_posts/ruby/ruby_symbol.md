---
title: Ruby中的Symbol类型
p: ruby/ruby_symbol.md
date: 2020-05-11 09:37:34
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

# Symbol类型

Symbol是Ruby中特有的，称为『符号』，它是**不可变**对象。对于Symbol，理解它的意义比掌握它的具体用法更重要。

在展现形式上，Symbol的第一个字符使用一个冒号，其它和字符串形式完全相同。事实上String可以转换成Symbol(`intern()`方法或`to_sym()`方法)，Symbol也可以转换成String(`to_s()`方法或`id2name()`方法)，唯一不同的是Symbol前面多了一个冒号。

```ruby
# Symbol字面量
:name          #=> :name
:"last name"   #=> :"last name"
a=:name        #=> :name
a.class        #=> Symbol
a="last name"  #=> "last name"
:"#{a}"        #=> :"last name"

# %s构造Symbol字面量
%s(last name)  #=> :"last name"

# String --> Symbol
"last name".to_sym  #=> :"last name"

# Symbol --> String
:"last name".to_s     #=> "last name"
:"last name".id2name  #=> "last name"
```

由于Symbol和String的特性不同，它们的使用场景也有很大不同。但仍然有很多场景下它们可以互换使用。

## Symbol在Ruby内部的维护

**Ruby中的常量名、方法名、变量名、类名等等都是Symbol类型**。

![](/img/ruby/1589465254220.png)

由于**Symbol是解释器内部维护的全局性的表**，所以不同上下文中对名称相同的Symbol对象的引用是同一个对象。例如，在某处定义了常量名GOOD，它是一个Symbol，那么在另一个上下文中如果定义了一个名为GOOD的方法，那么它们引用的对象都是Symbol表中的同一个GOOD对象，只不过它们的意义不一样，一个是常量，一个是方法。


## 关于Symbol的等同性

对于字符串来说，内容相同的两个字符串可能是不同的字符串对象。

但对于Symbol来说，只要内容相同，那么一定是同一个Symbol对象。反之，内容不同的两个Symbol一定是不同的对象。因为在创建Symbol对象时，首先会根据其内容做一番处理然后存储到Ruby解释器内部维护的Symbol表中。所以，Symbol能保证了唯一性。

## Symbol能用来干什么

通常会使用id作为标识符来表示某个东西(对象)的唯一性，id通常也是数值形式的。但是在Ruby中，Symbol可以看作是字符串形式的标识符，只不过这个『字符串』是不可变的。

正如Ruby中的类、方法等都是使用Symbol作为名称标识符的，即能保证唯一性，又能保证检索的高效性。

当使用字符串的目的仅仅只是一种标识而不考虑其内容的变化性，那么可以使用Symbol来取代字符串。

比如，Hash类型的key经常使用Symbol类型。

再例如，某方法中接收一个参数，参数可能是『AM』或『PM』，它们是不变的，就这两种情况，那么可以将该参数定义为Symbol类型。

## Symbol的一些常用API

`Symbol::all_symbols -> arr`  
返回Ruby解释器内部维护的Symbol表中所有已定义的Symbol。

```ruby
Symbol.all_symbols.size  #=> 4680

Symbol.all_symbols.select {|elm|  elm == :class}
      #=> [:class]

Symbol.all_symbols.select {|elm|  elm == :Kernel}
      #=> [:Kernel]
```

`to_s`  
`id2name`  
将Symbol转换成字符串。

`sym1 <=> sym2`  
比较两个Symbol对象的大小，比较的方式是to_s转换成字符串，再对字符串使用`<=>`做比较。

`casecmp(other_symbol) → -1, 0, +1, or nil`  
不区分大小写的比较Symbol，而比较是转换成字符串比较的。它是不区分大小写版的`<=>`方法。

`sym1 == sym2`  
`sym1 === sym2`  
sym1和sym2是同一Symbol对象时返回true，即它们在字符串上的内容相同时为true。

`sym1 =~ OBJ`  
将sym1先转换成字符串，再做正则匹配。即等价于`sym1.to_s =~ OBJ`。一般这个时候的OBJ是正则表达式。

`sym[idx]`  
`sym[idx,length]`  
`sym[rng]`  
`slice[idx]`  
`slice[idx,length]`  
`slice[rng]`  
sym先转换成字符串，然后再索引字符串。例如：

```ruby
:hello[1..3]  #=> "ell"
:hello[1,3]   #=> "ell"
:hello[1]     #=> "e"
```

`capitalize → symbol`  
`capitalize([options]) → symbol`  
将symbol转换成字符串后改其首字母大小，然后再转换成Symbol。即等价于`sym1.to_s.capitalize.to_sym`。

```ruby
:"last name".capitalize  #=> :"Last name"
```

