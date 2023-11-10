---
title: Ruby处理YAML和json
p: ruby/ruby_yaml_json.md
date: 2020-05-16 22:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby处理YAML

Ruby的标准库YAML基于Psych：https://ruby-doc.org/stdlib-2.6.2/libdoc/psych/rdoc/Psych.html  

![](/img/ruby/1589700330729.png)

例如：

```ruby
require 'yaml'
require 'set'

p "hello world".to_yaml
p 123.to_yaml
p %w(perl shell php).to_yaml
p ({one: 1, two: 2}).to_yaml
p Set.new([1,2,3]).to_yaml
```

得到：

```
"--- hello world\n"
"--- 123\n"
"---\n- perl\n- shell\n- php\n"
"---\n:one: 1\n:two: 2\n"
"--- !ruby/object:Set\nhash:\n  1: true\n  2: true\n  3: true\n"
```

也可以使用`YAML.dump()`方法实现和to_yaml相同的功能，它还可以写入文件。

```ruby
users = [{name: 'Bob', permissions: ['Read']},
 {name: 'Alice', permissions:['Read', 'Write']}]

File.open("/tmp/a.yml","w") { |f| YAML.dump(users, f) }
```

查看文件：

```bash
---
- :name: Bob             #=> 注意，保留了hash源数据中的符号
  :permissions:
  - Read
- :name: Alice
  :permissions:
  - Read
  - Write
```

使用YAML.load()从YAML中读取数据：

```ruby
require 'yaml'

pp YAML.load(DATA)

__END__
mysql:
  passwd: P@ssword1!
  user: root
  port: 3306
  other1: nil
  other2: false
  other3: ""
  hosts: 
    - ip: 10.10.1.1
      hostname: node1
    - ip: 10.10.1.2
      hostname: node2
```

得到：

```ruby
{"mysql"=>
  {"passwd"=>"P@ssword1!",      #=> 注意，key是String而非Symbol
   "user"=>"root",
   "port"=>3306,
   "other1"=>"nil",
   "other2"=>false,
   "other3"=>"",
   "hosts"=>
    [{"ip"=>"10.10.1.1", "hostname"=>"node1"},
     {"ip"=>"10.10.1.2", "hostname"=>"node2"}]}}
```

如果想让hash的key是符号而非字符串，可以设置选项`symbolize_names: true`：

```ruby
pp YAML.load(DATA, symbolize_names: true)
```

需要注意，YAML可以将对象进行序列化，所以有几方面注意事项：  

1. 在反序列化的时候需要也require涉及到的文件，例如对Set类型序列化后，在反序列化时如不`require 'set'`则无法还原对象  
2. 有些底层对象不能序列化，包括IO流、Ruby代码对象Proc、Binding等  
3. 不要反序列化不被信任的数据对象(比如用户输入的数据)，此时可使用safe_load()，它默认只允许加载以下几种类型的数据：  
   - TrueClass  
   - FalseClass  
   - NilClass  
   - Numeric  
   - String  
   - Array  
   - Hash  
4. 如果确实想要加载额外的数据类型，可以在safe_load()中指定参数**permitted_classes: []或permitted_symbols: []**  

# Ruby处理Json数据

## 转为json格式字符串

使用JSON.generate()可以将对象或数组转换为JSON格式的数据：

```ruby
require 'json'
p JSON.generate "abc"
p JSON.generate 123
p JSON.generate true
p JSON.generate nil
p JSON.generate [2,3,4]
p JSON.generate({name: "junmajinlong", age: 23})
require 'set'
p JSON.generate(Set.new([1,23,44]))
```

得到：

```
"\"abc\""
"123"
"true"
"null"
"[2,3,4]"
"{\"name\":\"junmajinlong\",\"age\":23}"
"\"#<Set: {1, 23, 44}>\""
```

当`require 'json'`后，很多ruby类型都具备了一个to_json的方法，可以直接将该类型的数据转换为json数据：

```ruby
p ({name: "junmajinlong", age: 23}).to_json
p (Set.new([1,23,44])).to_json
```

得到：

```
"{\"name\":\"junmajinlong\",\"age\":23}"
"\"#<Set: {1, 23, 44}>\""
```

此外，JSON.dump()也可以将对象转换为JSON格式的字符串，而且它还支持写入文件：

```ruby
hsh = {name: "junmajinlong", age: 23}
File.open("/tmp/a.json", "w") {|f| JSON.dump(hsh, f)}
```

## json格式字符串转为Ruby对象

要从json格式字符串转为ruby对象，有一些选项可设置，参考https://ruby-doc.org/stdlib-2.7.1/libdoc/json/rdoc/JSON.html#method-i-parse>，比如*symbolize_names*选项表示是否将json object中的key解析为符号类型的key，如果设置为false，则解析为字符串的key。

要将json格式的字符串解析为Ruby数据类型(Hash)，使用JSON.parse()，

```ruby
require 'json'

hsh = '{"name": "junmajinlong", "age": 23}'

p JSON.parse(hsh)
p JSON.parse(hsh, symbolize_names: true)
```

注意，上面的json字符串必须是合理的json数据，比如key必须使用双引号包围而不能使用单引号，字符串必须使用双引号包围，等等。比如`"{'name': 'junmajinlong', 'age': 23}"`就不是合理的json字符串。

要从json文件中读取json数据并转换为Ruby数据，使用load()：

```ruby
data = File.open("/tmp/a.json") do |f|
  JSON.load(f)
end

pp data
#=> {"name"=>"junmajinlong", "age"=>23}
```

## 自定义对象的转换方式

json支持的数据类型有：  

- 字符串  
- 数值  
- 对象  
- 数组  
- 布尔  
- Null  

从一种语言的数据转换为Json数据时，如果数据类型也是JSON所支持的，可直接转换，但如果包含了JSON不支持的类型，则可能报错，也可能以一种对象字符串的方式保存，这取决于对应的实现。

可以在对象中定义`as_json`实例方法来决定对象如何转换为json字符串，再定义类方法`from_json()`来决定如何从json字符串中恢复为一个对象。

例如，

```ruby
require 'json'
require 'date'

class Person
  attr_accessor :name, :birthday
  def initialize name, birthday
    @name = name
    @birthday = DateTime.parse(birthday)
  end
end

File.open("/tmp/p.json", "w") do |f|
  JSON.dump(Person.new("junmajinlong", "1999-10-11"), f)
end
```

查看保存的json数据：

```bash
$ cat /tmp/p.json
"#<Person:0x00007fffc7e575d0>"
```

定义`as_json`和`frmo_json`：

```ruby
require 'json'
require 'date'

class Person
  attr_accessor :name, :birthday
  
  def initialize name, birthday
    @name = name
    @birthday = DateTime.parse(birthday)
  end
  
  def as_json
    {
      name: @name,
      birthday: @birthday.strftime("%F")
    }
  end

  def self.from_json json
    data = JSON.parse(json)
    new(data["name"], data["birthday"])
  end
end
```

之后要序列化、反序列化该对象，可：

```ruby
data = Person.new("junmajinlong", "1999-10-11").as_json
p data

p1=Person.from_json(JSON.dump data)
p p1.birthday
```

如果是读写json文件，可：

```ruby
person1 = Person.new("junmajinlong", "1999-10-11")
File.open("/tmp/p.json", "w") do |f|
  JSON.dump(person1.as_json, f)
end

p1 = File.open("/tmp/p.json") do |f|
  Person.from_json(f.read)
  # Person.from_json(JSON.load(f).to_json)
end
p p1
```


## 几种JSON解析工具的性能测试

测试了json标准库、oj和fast_josnparser解析json的性能，测试项包括：

- 从文件中加载并解析json字符串为ruby对象
- 从内存json字符串中解析json字符串为ruby对象
- 带有symbolize_keys/symbolize_names转换时的解析 
- json标准库和oj将ruby对象dump为json字符串
- json标准库和oj将ruby对象dump为json字符串保存到文件

注:

- fast_jsonparser没有dump功能，只有解析json字符串功能
- oj在将对象转换为json字符串时，可能会丢失数据的精度，比如浮点数的精度

测试的json字符串数量大约50M。

测试了ruby 2.7.1和ruby 3.0.1两个版本，gem包的版本信息如下：

```
fast_jsonparser (0.5.0)
json (default: 2.5.1)
oj (3.11.7)
```

测试代码：

```ruby
require 'benchmark'
require 'json'
require 'oj'
require 'fast_jsonparser'

# warm
json_file='test'  # 文件大小大约50M
str = File.read(json_file)

######## JSON

puts " load file ".center(80, '-')
Benchmark.bm(30) do |x|
  x.report("JSON.load:") { File.open(json_file){ |f| JSON.load(f) } }
  x.report("Oj.load_file:") { Oj.load_file(json_file) }
  x.report("FastJsonparser.load:") { FastJsonparser.load(json_file) }
end

puts
puts " load file with symbolize_keys ".center(80, '-')
Benchmark.bm(30) do |x|
  x.report("JSON.load:") { File.open(json_file){ |f| JSON.load(f, nil, symbolize_names: true, create_additions: false) } }
  x.report("Oj.load_file:") { Oj.load_file(json_file, symbol_keys: true) }
  x.report("FastJsonparser.load:") { FastJsonparser.load(json_file, symbolize_keys: true) }
end

puts
puts " parse str ".center(80, '-')
Benchmark.bm(30) do |x|
  x.report("JSON.parse:") { JSON.parse(str) }
  x.report("Oj.load:") { Oj.load(str) }
  x.report("FastJsonparser.parse:") { FastJsonparser.parse(str) }
end

puts
puts " parse str with symbolize_keys ".center(80, '-')
Benchmark.bm(30) do |x|
  x.report("JSON.parse:") { JSON.parse(str, symbolize_names: true) }
  x.report("Oj.load:") { Oj.load(str, symbol_keys: true) }
  x.report("FastJsonparser.parse:") { FastJsonparser.parse(str, symbolize_keys: true) }
end

obj = JSON.parse(str, symbolize_names: true)

puts 
puts " dump JSON to str ".center(80, '-')
Benchmark.bm(30) do |x|
  x.report("JSON.dump:") { JSON.dump(obj) }
  x.report("Oj.dump:") { Oj.dump(obj) }
end

puts 
puts " dump JSON to file ".center(80, '-')
Benchmark.bm(30) do |x|
  x.report("JSON.dump:") { File.open('0_json_dump', 'w') {|f| JSON.dump(obj, f) } }
  x.report("Oj.to_file:") { Oj.to_file('0_oj_dump', obj) }
end
```

测试结果：

Ruby 2.7.1中：

```
---------------------------------- load file -----------------------------------
                                     user     system      total        real
JSON.load:                       1.591831   0.058021   1.649852 (  1.738119)
Oj.load_file:                    1.350385   0.057684   1.408069 (  2.434268) <-慢
FastJsonparser.load:             0.653968   0.103258   0.757226 (  0.848913) <-快

------------------------ load file with symbolize_keys -------------------------
                                     user     system      total        real
JSON.load:                       1.212617   0.039052   1.251669 (  1.349545)
Oj.load_file:                    1.432059   0.098950   1.531009 (  2.679610) <-慢
FastJsonparser.load:             0.695538   0.008384   0.703922 (  0.797081) <-快

---------------------------------- parse str -----------------------------------
                                     user     system      total        real
JSON.parse:                      1.343596   0.000000   1.343596 (  1.350368)
Oj.load:                         1.133612   0.000000   1.133612 (  1.140939)
FastJsonparser.parse:            0.701701   0.012340   0.714041 (  0.720296) <-快

------------------------ parse str with symbolize_keys -------------------------
                                     user     system      total        real
JSON.parse:                      1.250775   0.000000   1.250775 (  1.258796)
Oj.load:                         1.131296   0.000000   1.131296 (  1.138020)
FastJsonparser.parse:            0.697433   0.015962   0.713395 (  0.719439) <-快

------------------------------- dump JSON to str -------------------------------
                                     user     system      total        real
JSON.dump:                       1.374611   0.028454   1.403065 (  1.403081)
Oj.dump:                         1.025049   0.040184   1.065233 (  1.065246) <-快

------------------------------ dump JSON to file -------------------------------
                                     user     system      total        real
JSON.dump:                       1.234362   0.040246   1.274608 (  1.369214)
Oj.to_file:                      1.168707   0.000000   1.168707 (  1.270957)
```

Ruby 3.0.1中：

```
---------------------------------- load file -----------------------------------
                                     user     system      total        real
JSON.load:                       1.362151   0.083610   1.445761 (  1.569754)
Oj.load_file:                    1.343601   0.182046   1.525647 (  2.684472) <-慢
FastJsonparser.load:             2.634435   0.052734   2.687169 (  2.776105) <-慢

------------------------ load file with symbolize_keys -------------------------
                                     user     system      total        real
JSON.load:                       1.287954   0.018572   1.306526 (  1.409770)
Oj.load_file:                    1.478750   0.043847   1.522597 (  2.668882) <-慢
FastJsonparser.load:             2.717857   0.006164   2.724021 (  2.822728) <-慢

---------------------------------- parse str -----------------------------------
                                     user     system      total        real
JSON.parse:                      1.242225   0.008661   1.250886 (  1.304554)
Oj.load:                         1.097922   0.000000   1.097922 (  1.110031)
FastJsonparser.parse:            2.602679   0.017232   2.619911 (  2.634604) <-慢

------------------------ parse str with symbolize_keys -------------------------
                                     user     system      total        real
JSON.parse:                      1.368262   0.000000   1.368262 (  1.380730)
Oj.load:                         1.332349   0.000000   1.332349 (  1.346331)
FastJsonparser.parse:            2.706804   0.007238   2.714042 (  2.726935) <-慢

------------------------------- dump JSON to str -------------------------------
                                     user     system      total        real
JSON.dump:                       1.724653   0.009250   1.733903 (  1.733912)
Oj.dump:                         1.298235   0.030041   1.328276 (  1.328279) <-快

------------------------------ dump JSON to file -------------------------------
                                     user     system      total        real
JSON.dump:                       1.765664   0.040595   1.806259 (  1.905785)
Oj.to_file:                      1.228744   0.020309   1.249053 (  1.349684) <-快
=end
```

性能测试结论：
- (1).ruby 3之前，fast_jsonparser非常快，但是Ruby 3中的fast_jsonparser很慢
- (2).OJ解析本地json字符串的性能比标准库json性能稍好，但oj从文件中加载并解析json的速度很慢
- (3).OJ将ruby对象解析为json字符串的效率比json标准库性能好

即：

```
dump：
Oj.dump > JSON.dump

ruby 3之前：
FastJsonparser.load > JSON.load > Oj.load_file
FastJsonparser.parse > Oj.load > JSON.parse

ruby 3之后：
JSON.load > Oj.load_file > FastJsonparser.load
Oj.load > JSON.parse > FastJsonparser.parse
```

## multi_json

有一个名为multi_json的gem包，它提供多种json包的功能，默认采用OJ作为json的适配引擎。它支持下面几种json适配器：

- [Oj](https://github.com/ohler55/oj) Optimized JSON by Peter Ohler
- [Yajl](https://github.com/brianmario/yajl-ruby) Yet Another JSON Library by Brian Lopez
- [JSON](https://github.com/flori/json) The default JSON gem with C-extensions (ships with Ruby 1.9+)
- [JSON Pure](https://github.com/flori/json) A Ruby variant of the JSON gem
- [NSJSONSerialization](https://developer.apple.com/library/ios/#documentation/Foundation/Reference/NSJSONSerialization_Class/Reference/Reference.html) Wrapper for Apple's NSJSONSerialization in the Cocoa Framework (MacRuby only)
- [gson.rb](https://github.com/avsej/gson.rb) A Ruby wrapper for google-gson library (JRuby only)
- [JrJackson](https://github.com/guyboertje/jrjackson) JRuby wrapper for Jackson (JRuby only)
- [OkJson](https://github.com/kr/okjson) A simple, vendorable JSON parser

如果oj已被require，则默认采用oj处理json，如果oj没有被require，而是require了yajl，则采用yajl处理json，依次类推。

它提供了load()和dump()方法：

```
load(json_str, options = {})
  options: 
    symbolize_keys: true, false
    adapter:  oj, json_gem, yajl, json_pure, ok_json

dump(object, options = {})    
```