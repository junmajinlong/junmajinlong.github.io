---
title: Ruby中的Hash结构
p: ruby/ruby_hash.md
date: 2020-05-14 08:26:52
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby中的Hash类型

## 创建Hash对象

### 字面量创建hash

Ruby中通常使用字符串或Symbol类型作为hash的key。

以字符串作为键:

```ruby
numbers = {"one" => 1, "two" => 1, "three" => 3}
```

![](/img/ruby/1589464589903.png)

当创建的hash中换行写时，一般建议在最后一个元素后追加一个逗号，以便下次继续追加元素时不容易出错。

```ruby
numbers = {
    :one => 1,
    :two => 2,
    :three => 3,
    }
```


### Hash()

通过Kernel中提供的Hash方法来将给定数据转换成Hash类型。它是调用to_hash的方式来转换的，所以如果对象没有to_hash方法将报错。如果给定参数为nil或空数组，则返回空hash。

```ruby
Hash([])               #=> {}
Hash(nil)              #=> {}
Hash(key: :value)      #=> {:key => :value}
Hash(a: "aa", b: "bb") #=> {:a=>"aa", :b=>"bb"}
Hash([1, 2, 3])        #=> TypeError
```

### Hash[]

直接通过类方法`[]`来创建。

```ruby
Hash[ key, value, ... ] → new_hash
Hash[ [ [key, value], ... ] ] → new_hash
Hash[ object ] → new_hash
```

对于`Hash[ object ]`，要求object对象具有to_hash方法，否则报错。

例如：

```ruby
Hash["a", 100, "b", 200]             #=> {"a"=>100, "b"=>200}
Hash[ [ ["a", 100], ["b", 200] ] ]   #=> {"a"=>100, "b"=>200}
Hash["a" => 100, "b" => 200]         #=> {"a"=>100, "b"=>200}
```

### Hash.new

```ruby
new → new_hash
new(obj) → new_hash
new {|hash, key| block } → new_hash
```

对于第一种情况，创建一个空hash，且访问不存在的key时返回nil。

对于第二种情况，创建一个空hash，但访问不存在的key时，返回这里所指定的默认值obj。

对于第三种情况，创建一个空hash，每次访问不存在的key时，都自动填充根据block来创建并返回给定key。当然，也可以选择不填充不存在的key。其中`|hash, key|`中的hash代表的是这个hash结构名，key代表的是访问时的key。

```ruby
h = Hash.new("Go Fish")     # 指定默认值
h["a"] = 100
h["b"] = 200
h["a"]           #=> 100
h["c"]           #=> "Go Fish"

# The following alters the single default object
# 注意，默认值也被修改了
h["c"].upcase!   #=> "GO FISH"
h["d"]           #=> "GO FISH"
h.keys           #=> ["a", "b"]

# While this creates a new default object each time
h = Hash.new { |hash, key| hash[key] = "Go Fish: #{key}" }
h["c"]           #=> "Go Fish: c"
h["c"].upcase!   #=> "GO FISH: C"
h["d"]           #=> "Go Fish: d"
h.keys           #=> ["c", "d"]
```

## hash元素的访问和赋值

Ruby中访问Hash对象中某元素时，使用`myhash[key]`的方式。注意不是使用大括号而是中括号(理解为关联数组)。

hash也提供了一个`store()`方法来赋值。

```ruby
h={a: "aa", b: "bb"}
h[:a]
h[:b]="bbb"
h.store(:b, "bbbb")
```

`fetch()`和`fetch_values()`也可以用来获取hash中给定key的value。前者只能获取一个key的value，后者可以一次性获取多个key对应的value并返回数组，如果它们获取的某个key不存在，会直接报错，但可以指定key不存在时的返回值。它们都能使用语句块格式来指定key不存在时的返回值。

```ruby
h = { a: 100, b: 200, c: 300 }
h.fetch(:a)                #=> 100
h.fetch(:z)                #=> 报错KeyError
h.fetch(:z, "go fish")     #=> "go fish"，在参数中指定返回值
h.fetch(:z) { |el| "go fish, #{el}"}
      #=> "go fish, z"，将key作为变量传递给语句块作为返回值

h.fetch_values(:a)       #=> [100]
h.fetch_values(:a,:b)    #=> [100, 200]
h.fetch_values(:a,:z)    #=> 报错KeyError
h.fetch_values(:a,:d) {|x| "d_value 400"}
      #=> [100, "d_value 400"],:d不存在，所以返回指定的返回值
```

对于访问hash的key或value来说，还有其它一些比较好用的方法。

- `keys`：返回所有key组成的数组
- `values`：返回所有value组成的数组
- `key(VALUE)`：根据给定的值返回它的key，如果有多个VALUE，则返回第一个出现的key，不存在则返回nil
- `values_at(key1,key2...)`：返回指定key(可以指定多个key)的value组成的数组
- `slice(key1,key2...)`：类似于value_at，只不过slice返回的是hash结果
- `has_value?()`和`value?(VALUE)`：它们等价，测试给定value是否存在于hash中
- `has_key?(KEY)`和`key?(KEY)`和`include?(KEY)`和`member?(KEY)`：它们是等价的别名，测试给定key是否存在于hash中
- `empty?`：判断是否为空hash  

```ruby
# Hash#slice
h = { a: 100, b: 200, c: 300 }
h.slice(:a)           #=> {:a=>100}
h.slice(:b, :c, :d)   #=> {:b=>200, :c=>300}

# Hash#values_at
h.values_at(:a, :b)   #=> [100, 200]
```

## hash检索key的方式

默认情况下，搜索Hash中元素时是将给定key和hash中的key做`eql?`比较的，但是这有可能会出现hash键冲突问题，如果键冲突较多，hash的效率会急剧下降。

如果要作为key的对象没有定义hash()方法，将直接采用它的对象id(object_id)作为比较依据，因为它默认会继承`Object#hash()`，而它返回的是object_id。这造成的结果是尽管key的内容相同，但由于不是同一对象，仍然可能访问不到对应的value。

其实，Hash有个实例方法称为`compare_by_identity`，可以手动设置hash搜索元素时以`equal?`来比较key，使得在访问hash比较key时以其唯一标识(即地址或id)作为筛选依据，也就是key的对象完全相同才能访问到hash中的元素。

也就是说，`compare_by_identity`是一个筛选方式的开关，是hash对象的一个属性字段。Ruby还提供了一个检查这个开关是否设置的方法`compare_by_identity?`，如果已设置，则返回true。

例如：

```ruby
h={a: "aa", b: "bb", "c" => "cc"}
h.compare_by_identity     # 先打开这个设置
h.compare_by_identity?    #=> true
h["c"]   #=> nil，因为此时"c"和hash中的"c"不是同一个对象
h[:a]    #=> "aa"，因为Symbol是同一对象
```

虽然这样的设置能保证key的唯一性。但设置了`compare_by_identity`限制之后，**字符串形式的key将没办法直接通过字符串key去访问**，只能遍历访问。

```ruby
h.each_key { |key| puts h[key] }
=begin
aa
bb
cc
=end
```

该设置(compare_by_identity)无法取消，只能重建hash。

```ruby
h={:a=>"aa", :b=>"bb"}
h.compare_by_identity
h.compare_by_identity?    #=> true

hh={}
h.each { |key,value| hh[key]=value }
hh.compare_by_identity?    #=> false
```

## hash的默认值

默认情况下，当检索Hash中不存在的键时，将返回nil。但可以设置hash的默认值，当检索到不存在的键时，将返回这个默认值。

设置默认值的方式有两种：
1. 通过hash的default()方法设置hash的默认值
2. 通过Hash.new()方法创建hash的时候，指定默认值

例如：

```ruby
# 1.默认时，搜索不存在的键时返回nil
numbers = {one: 1, two: 2, three: 3,}
numbers[:four]           #=> nil

# 2.default()
numbers.default = "nnn"
numbers[:four]        #=> "nnn"

# 3.Hash.new()
h = Hash.new("mmm")
h[:four]              #=> "mmm"
```

注意，**默认值可能会在访问不存在键时被修改**。

```ruby
numbers = {one: 1, two: 2, three: 3,}
numbers.default = "nnn"   #=> "nnn"

>> numbers["five"]         #=> "nnn"
>> numbers["six"]          #=> "nnn"
>> numbers["five"].upcase! #=> "NNN"
>> numbers["six"]          #=> "NNN" 默认值已被改
```

另外，关于hash默认值有以下重要的两点：
- 只要访问的是不存在的键，即使指定了其返回默认值，它在hash中也仍然是不存在的
- 但可以通过Hash.new的代码块逻辑，设置当访问不存在的键时，自动设置该不存在的键，且还可以加上默认不存在键的返回值。它的副作用是，即便只是检测键是否存在，也会以默认值创建这个键元素，建议的方式参考下面的示例 

例如，下面通过Hash.new代码块将所有访问的不存在的键的值设置为0，且可以单独指定其默认返回值。
```ruby
h = {}
h.default = "unknown key"
h["a"]      #=> "unknown key"
h           #=> {}，仍然为空

# 使用默认返回值0
h1 = Hash.new {|hash,key| hash[key] = 0}
h1["a"]    #=> 0
h1         #=> {"a"=>0}

# 手动指定返回值
h2 = Hash.new {|hash,key| hash[key] = 0;"unknown key"}
h2["a"]    #=> "unknown key"
h2         #=> {"a"=>0}


# 即便只是检测，也会创建元素
h3 = Hash.new {|hash,key| hash[key] = []}
h3["a"]   #=> []
h3        #=> {"a"=>[]}

# 可以考虑在访问的时候设置不存在元素的初始化值
h4 = {}
(h4[:a] ||= []).push("hello")  # 不存在时设置为[]，即等价于设置了默认值[]
```

访问不存在的键并将其添加到hash结构中，这是非常实用的一个技巧。

## 访问多层嵌套的hash

Hash提供了一个非常好用的方法dig()，在数组中也存在这个方法，都用于访问多层次的容器对象。

在Hash中更方便，不断指定key即可，而且还可以混合array一起dig。

```ruby
dig(key, ...) → object
```

例如：

```ruby
h = { foo: {bar: {baz: 1}}}

h.dig(:foo, :bar, :baz)     #=> 1
h.dig(:foo, :zot, :xyz)     #=> nil

g = { foo: [10, 11, 12] }
g.dig(:foo, 1)              #=> 11，混合Array
g.dig(:foo, 1, 0)           #=> TypeError
g.dig(:foo, :bar)           #=> TypeError
```

## hash的迭代遍历

each、each_key、each_value、each_pair，其中each和each_pair等价。

```ruby
myhash = {positive: "negative", up: "down", right: "left"}

myhash.each_key { |key| puts key }
myhash.each_value { |value| puts value }
myhash.each { |key, value| puts "Opposite of #{key}: #{value}" }
```

## hash有关的转换操作

- `try_convert`调用`to_hash`将给定参数转换成hash，转成功则返回转换后的hash对象，不成功则返回nil
- `to_hash`
- `to_h`
- `to_a`
- `to_s`
- `inspect`：对于Hash类型来说，等价于`to_s`
- `to_proc`：返回proc，该proc将hash的key映射为对应的value，不存在的元素对应于nil

```ruby
h = {a:1, b:2}
hp = h.to_proc
hp.call(:a)          #=> 1
hp.call(:b)          #=> 2
hp.call(:c)          #=> nil
[:a, :b, :c].map(&h) #=> [1, 2, nil]

h = {"c" => 300, "a" => 100}
h.to_s   #=> "{\"c\"=>300, \"a\"=>100}"

h={a: "aa",b: "bb"}  #=> {:a=>"aa", :b=>"bb"}
h.to_s               #=> "{:a=>\"aa\", :b=>\"bb\"}"
```

## 比较hash的大小

Hash类提供了几个比较好用的用来比较hash对象大小的方法。

下面的比较操作，均不考虑元素的前后顺序。

- `hash1 < hash2`：hash1是hash2的真子集时返回true
- `hash1 <= hash2`：hash1是hash2的子集时返回true
- `hash1 == hash2`：hash1等于hash2(每对key/value使用`==`比较后相等，其数量相同)时返回true
- `hash1 > hash2`：hash2是hash1的真子集时返回true
- `hash1 >= hash2`：hash2是hash1的子集时返回true

```ruby
h1 = {a:1, b:2}
h2 = {a:1, b:2, c:3}
h1 < h2    #=> true
h2 < h1    #=> false
h1 < h1    #=> false

h1 = {a:1, b:2}
h2 = {a:1, b:2, c:3}
h1 <= h2   #=> true
h2 <= h1   #=> false
h1 <= h1   #=> true

h1 = {a:1, b:2}
h2 = {a:1, b:2.0}
h1 == h2      #=> true
```

但是Hash并没有直接定义`<=>`，而是从Object中继承`<=>`，而Object中的`<=>`只做`==`比较，为真时返回0，为假时也就是大于或小于均返回nil。


## 筛选hash中的元素

`filter()`、`filter!()`、`select()`、`select!()`以及`keep_if()`都可以用来筛选hash中满足指定条件的元素。还有`reject()`、`reject!()`以及`delete_if()`是筛选不满足条件的元素。看起来很多，但它们都是等价的。

- filter和select等价
- filter!和select!和keep_if等价
- reject!和delete_if等价

```ruby
h = { "a" => 100, "b" => 200, "c" => 300 }
h.select {|k,v| k > "a"}  #=> {"b" => 200, "c" => 300}
h.select {|k,v| v < 200}  #=> {"a" => 100}

h.reject {|k,v| k < "b"}  #=> {"b" => 200, "c" => 300}
h.reject {|k,v| v > 100}  #=> {"a" => 100}
```

## 删除hash中的元素

- `delete`：删除指定key的元素
- `delete_if、reject()`：删除满足语句块中的元素
- `clear()`方法清空hash
- `compact()`和`compact!()`移除hash中所有**value**为nil的元素
- `shift()`移除第一个key/value对

### delete()

```ruby
delete(key) → value
delete(key) {| key | block } → value
```

删除hash中指定的key对应元素。如果给定key不存在，则返回nil。使用语句块时，如果key不存在，则传递key作为块的变量并返回块的返回值。

```ruby
h = { "a" => 100, "b" => 200 }
h.delete("a")                              #=> 100
h.delete("z")                              #=> nil
h.delete("z") { |el| "#{el} not found" }   #=> "z not found"
```

### delete_if()

```ruby
delete_if {| key, value | block } → hsh
delete_if → an_enumerator
```

删除满足语句块条件的元素。

```ruby
h = { "a" => 100, "b" => 200, "c" => 300 }
h.delete_if {|key, value| key >= "b" }   #=> {"a"=>100}
```

### shift()

从hash中移除key/value对，当已经没元素可以shift之后，将返回hash的默认值。

```ruby
h={a:"aa",b:"bb"}
h.default="defalut_value"

h.shift     #=> [:a, "aa"]
h.shift     #=>[:b, "bb"]
h.shift     #=> "defalut_value"
```

## merge和update和replace

`merge()`可以将其它一个或多个hash合并到某hash对象中。如果不指定任何参数，则直接返回一个hash的拷贝。

`merge!()`和`update()`等价，都是原处修改hash。

显然在merge时可能会出现key重复的问题。对于此，merge提供了两种方式去解决key重复的问题：

- 不使用语句块时，重复时将采用覆盖key的方式  
- 使用语句块的时候，将采用语句块中的处理逻辑来定义这个key的value值  

```ruby
h1 = { a: 100, b: 200 }
h2 = { b: 246, c: 300 }
h3 = { b: 357, d: 400 }
h1.merge          #=>{:a=>100, :b=>200}
h1.merge(h2)      #=>{:a=>100, :b=>246, :c=>300}
h1.merge(h2, h3)  #=>{:a=>100, :b=>357, :c=>300, :d=>400}
h1.merge(h2) {|key, oldval, newval| newval - oldval}
                  #=>{:a=>100, :b=>46, :c=>300}
h1.merge(h2, h3) {|key, oldval, newval| newval - oldval}
                  #=>{:a=>100, :b=>311, :c=>300, :d=>400}
```

`replace()`方法将某hash的所有元素直接替换成另一个hash的元素。既然是元素的替换，那么自然是原处修改的，且hash对象自身的id是不变的。

```ruby
h = { "a" => 100, "b" => 200 }
h.object_id    #=> 70368443261440
h.replace({ "c" => 300, "d" => 400 })   #=> {"c"=>300, "d"=>400}
h.object_id    #=> 70368443261440
```

## 更新key或value: transform_XXX

```ruby
transform_keys {|key| block } → new_hash
transform_keys → an_enumerator
transform_keys! {|key| block } → new_hash
transform_keys! → an_enumerator

transform_values {|value| block } → new_hash
transform_values → an_enumerator
transform_values! {|value| block } → new_hash
transform_values! → an_enumerator
```

`transform_{keys|values}`用来生成一个新的hash，这个新的hash的`{key|value}`根据语句块来生成。

```ruby
h={a:"aa",b:"bb"}  #=> {:a=>"aa", :b=>"bb"}
h.transform_keys { |key| key.to_s.upcase }
                   #=> {"A"=>"aa", "B"=>"bb"}
h.transform_values { |value| value.to_s.upcase }
                   #=> {:a=>"AA", :b=>"BB"}
```

## key/value倒转

Hash提供了一个方法`invert()`可以将hash中每对key/value倒转，即key当作value，而value当作key。

```ruby
h={:a=>100, :b=>200, :c=>300}
h.invert   #=> {100=>:a, 200=>:b, 300=>:c}
h          #=> {:a=>100, :b=>200, :c=>300}
```

但是，如果有多个key的value都相同，那么倒转后将出现key重复的问题，invert的处理方式是只取最后一个，前面的丢弃。

```ruby
h={:a=>100, :b=>300, :c=>300}
h.invert        #=> {100=>:a, 300=>:c}
```

假设hash对象中没有重复的value，那么倒转操作是对合的。

```ruby
h={a:"aa",b:"bb",c:"cc"}
h.invert            #=> {"aa"=>:a, "bb"=>:b, "cc"=>:c}
h.invert.invert     #=> {:a=>"aa", :b=>"bb", :c=>"cc"}
h.invert.invert == h   # true
```

所以，可以比较倒转后的hash的size来轻松判断一个hash中是否存在多个key的value相同。

```ruby
h1={a:"aa",b:"bb",c:"cc"}
h2={a:"aa",b:"bb",c:"bb"}

h1.size == h1.invert.size   #=> true，所有value都唯一
h2.size == h2.invert.size   #=> false，存在重复的value
```

## 压平hash

类似于`Array#flatten`，Hash也有一个flatten方法，可以将hash压平成一个Array。

默认不会递归压平，而是只压最外层，但可以指定压的层次。

```
faltten(level=1)
```

例如：

```ruby
a =  {1=> "one", 2 => [2,"two"], 3 => "three"}
a.flatten    # => [1, "one", 2, [2, "two"], 3, "three"]
a.flatten(2) # => [1, "one", 2, 2, "two", 3, "three"]
```
