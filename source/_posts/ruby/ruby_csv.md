---
title: Ruby处理CSV数据
p: ruby/ruby_csv.md
date: 2020-05-16 22:53:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby处理CSV

![](/img/ruby/1589641804734.png)

可设置的项及其默认值包括：

```
col_sep: ",",                #=> 字段分隔符
row_sep: :auto,              #=> 记录分隔符
quote_char: '"',             #=> 包围字段的符号
field_size_limit: nil,       #=> 限制字段的字符数量
converters: nil,             #=> 
unconverted_fields: nil,
headers: false,              #=> 读取时忽略标题行，具体参考官方手册
return_headers: false,
write_headers: nil,
header_converters: nil,
skip_blanks: false,          #=> 忽略空行
force_quotes: false,         #=> 设置为true时，所有字段都将使用被包围
skip_lines: nil,             #=> 指定一个正则(str也会转换为正则)，
                             #=> 匹配的行将被当作注释行而忽略
liberal_parsing: false,
internal_encoding: nil,
external_encoding: nil,
encoding: nil,
nil_value: nil,             #=> 使用此处设置的值替换所有nil字段
empty_value: "",            #=> 使用此处设置的值替换所有空字符串字段
quote_empty: true,          #=> 设置为false时，空字符串字段将转换为空字段
write_converters: nil,
write_nil_value: nil,      #=> 将以此处的值替换nil字段写入文件
write_empty_value: "",
strip: false
```

## CSV类方法处理CSV数据

### 以CSV格式写入文件

要向文件中写入CSV格式的数据：

```ruby
require 'csv'

writer = CSV.open('/tmp/file.csv', 'w')
writer << ["junmajinlong", 29, 170, true]
writer << ["junma", 24, 176, false]
writer << ["jinlong", 25, 172, nil]
writer << ["majinlong", 23, 173, false]
writer.close
```

写入完成后，查看：

```
junmajinlong,29,170,true
junma,24,176,false
jinlong,25,172,
majinlong,23,173,false
```

注意其中的nil对应的写入内容为空。

可以直接在语句块中写入，这样的话可以自动关闭CSV.open()打开的IO流：

```ruby
require 'csv'

CSV.open('/tmp/file.csv', 'w') do |writer|
  writer << ["junmajinlong", 29, 170, true]
  writer << ["junma", 24, 176, false]
  writer << ["jinlong", 25, 172, nil]
  writer << ["majinlong", 23, 173, false]
end
```

CSV.open()打开的是一个封装后的IO流对象，它除了可以使用CSV单独为其提供的一些方法(比如这里的`<<`)外，还可以使用很多IO流对象的方法，比如seek()、tell()、flush()、eof?()、fsync()等等。

这里使用的`<<`方法是单独为其提供的，它涉及两个执行过程：  

1. 将数组中各元素全部转换成字符串类型并使用逗号连接  
2. 按行写入到csv打开的文件中  

### 转换为CSV格式的字符串

如果只是想执行第一个过程，即将数据转换成CSV格式的字符串而不写入，可使用类方法`generate_line()`：

```ruby
p CSV.generate_line ["junmajinlong", 29, 170, true]
p CSV.generate_line ["jun ma", 24, 176, false]
p CSV.generate_line ["jinlong", 25, 172, nil]
p CSV.generate_line ["jin, long", 23, 173, false]
=begin
"junmajinlong,29,170,true\n"
"jun ma,24,176,false\n"
"jinlong,25,172,\n"
"\"jin, long\",23,173,false\n"
=end
```

### 从CSV格式的文件中读数据

如果想要读取CSV文件，可使用类方法read()或别名readlines()：

```ruby
pp CSV.readlines('/tmp/file.csv')
=begin
[["junmajinlong", "29", "170", "true"],
 ["junma", "24", "176", "false"],
 ["jinlong", "25", "172", nil],
 ["majinlong", "23", "173", "false"]]
=end
```

注意：  

1. 读取CSV文件内容时，每行保存为一个数组，每个字段是这个数组中的一个元素  
2. 读取CSV文件内容时，除了不存在的字段转换为nil外，其它所有的数据都转换成了字符串类型。所以有时候可能需要去转换读取时的数据类型。关于类型转换，见后文  

如果要按行读取CSV文件的内容，使用类方法foreach()：

```ruby
CSV.foreach('/tmp/file.csv') do |row|
  p row
end
=begin
["junmajinlong", "29", "170", "true"]
["junma", "24", "176", "false"]
["jinlong", "25", "172", nil]
["majinlong", "23", "173", "false"]
=end
```

### 从CSV格式的字符串中读数据

如果想要从字符串中读取CSV格式的数据，使用parse()和parse_line()，分别用于解析多行字符串和解析单行字符串(超出一行的自动被忽略)。

- parse()不指定语句块时，返回包含解析每一行得到的数组，即一个数组的数组，它是一个csv table类型，有很多自己的方法  
- 指定语句块时，每一行对应的数组传递给语句块控制变量  

```ruby
str1=<<-eof
junmajinlong,29,170,true
jun ma,24,176,false
jinlong,25,172,
"jin, long",23,173,false
eof

# 不指定语句块时，parse返回数组
pp CSV.parse str1
=begin
[["junmajinlong", "29", "170", "true"],
 ["jun ma", "24", "176", "false"],
 ["jinlong", "25", "172", nil],
 ["jin, long", "23", "173", "false"]]
=end

# 指定语句块时，parse将每行对应的数组传递给语句块
CSV.parse(str1) {|row| p row}
=begin
["junmajinlong", "29", "170", "true"]
["jun ma", "24", "176", "false"]
["jinlong", "25", "172", nil]
["jin, long", "23", "173", "false"]
=end

str2="junmajinlong,29,170,true"
p CSV.parse_line str2
["junmajinlong", "29", "170", "true"]
```

## CSV实例方法处理CSV数据

- `CSV.new()`、`CSV.open()`可以创建csv对象(即一行一行csv格式的数据)  
- `CSV.generate()`可将字符串转换成csv对象并将该对象传递给语句块  
- `<<`、`puts()`或`add_row()`可向CSV目标中(字符串格式的CSV或CSV IO流)写入行，它们是别名关系  
- `gets()`、`shift()`、`readline()`可从csv对象中读取一行数据  
- `read()`、`readlines()`可以读取csv对象中的所有数据  
- `each()`可以从csv对象中迭代每一行  
- `eof()`或`eof?()`可以判断是否读完所有数据  
- `rewind()`可以重置当前csv对象的偏移指针  
- `line()`可以获取最近一次读取的一行数据  
- `lineno()`可以获取当前已读取的行数  
- `path()`可以获取当前读取的csv文件名  

## CSV table

CSV.parse()、CSV.read()、CSV.table()等方法返回的都是数组的数组(二维数组)，它们是CSV Table。

CSV table按照表的方式来处理csv数据，比如关注于行、关注于字段的一些操作可以采用csv table相关的方法来处理。

```ruby
# Headers are part of data
data = CSV.parse(<<~ROWS, headers: true)
  Name,Department,Salary
  Bob,Engineering,1000
  Jane,Sales,2000
  John,Management,5000
ROWS

data.class      #=> CSV::Table
data.first      #=> #<CSV::Row "Name":"Bob" "Department":"Engineering" "Salary":"1000">
data.first.to_h #=> {"Name"=>"Bob", "Department"=>"Engineering", "Salary"=>"1000"}

# Headers provided by developer
data = CSV.parse('Bob,Engineering,1000', headers: %i[name department salary])
data.first      #=> #<CSV::Row name:"Bob" department:"Engineering" salary:"1000">
```

## CSV字段类型转换

读取CSV数据时，所有的数据都会转换为字符串格式。

```ruby
# Without any converters:
CSV.parse('Bob,2018-03-01,100')
#=> [["Bob", "2018-03-01", "100"]]
```

可以在迭代每一行的语句块中对字段做必要的类型转换。

但如果类型转换方式比较简单，可以在读取数据时指定converters属性进行转换。该属性的值要么是CSV的内置类型符号，要么是符号数组，要么是一个lambda表达式。有如下内置类型：

```
Integer
Float
Numeric (Float + Integer)
Date
DateTime
All
```

当指定了类型转换后，每个字段将针对converters的值尝试做转换，转换失败则保留字段的值不变，所以如果通过lambda自定义类型转换时也一定要保证这一点。

```ruby
CSV.parse("1,2,3,4,5", converters: :numeric)
#=> [[1, 2, 3, 4, 5]]

# With built-in converters:
ct = CSV.parse('Bob,2018-03-01,100', converters: %i[numeric date])
#=> [["Bob", #<Date: 2018-03-01>, 100]]
ct.first[1] + 1  # 日期对象，加1天
#=> #<Date: 2018-03-02 ((2458180j,0s,0n),+0s,2299161j)>

# With custom converters:
CSV.parse('Bob,2018-03-01,100', converters: [->(v) { Time.parse(v) rescue v }])
#=> [["Bob", 2018-03-01 00:00:00 +0200, "100"]]
```

