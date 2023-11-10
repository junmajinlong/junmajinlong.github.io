---
title: Ruby Mysql2操作MySQL
p: ruby/ruby_mysql.md
date: 2020-05-18 22:53:06
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby操作MySQL

使用mysql2连接mysql并操作mysql。相关文档参见：  
- <https://www.rubydoc.info/gems/mysql2/0.5.2>  
- <https://github.com/brianmario/mysql2>  

```bash
gem install mysql2
```

## 连接mysql

建立连接：

```ruby
require 'mysql2'

conn = Mysql2::Client.new({ 
  host: '192.168.200.73',
  username: 'root',
  password: 'P@ssword1!'
})
```

可接受的连接选项包括：

```ruby
Mysql2::Client.new(
  :host,
  :username,
  :password,
  :port,
  :database,
  :socket = '/path/to/mysql.sock',
  :flags = REMEMBER_OPTIONS | LONG_PASSWORD | LONG_FLAG | TRANSACTIONS | PROTOCOL_41 | SECURE_CONNECTION | MULTI_STATEMENTS,
  :encoding = 'utf8',
  :read_timeout = seconds,
  :write_timeout = seconds,
  :connect_timeout = seconds,
  :connect_attrs = {:program_name => $PROGRAM_NAME, ...},
  :reconnect = true/false,
  :local_infile = true/false,
  :secure_auth = true/false,
  :ssl_mode = :disabled / :preferred / :required / :verify_ca / :verify_identity,
  :default_file = '$HOME/.my.cnf',   #=> 从文件读取连接信息
  :default_group = 'my.cfg section', #=> 选择.my.cnf中的section
  :default_auth = 'authentication_windows_client',
  :init_command => SQL_Statement  #=>主要用于设置本次连接时的某些变量
)
```

连接建立后就可以操作数据库了，比如执行SQL语句：

![](/img/ruby/1589818485439.png)

如果测试和mysql的连接是否断开，可执行ping()：

```ruby
conn.ping
```

如果连接未断开，ping()返回true，如果连接已断开但已启用auto-reconnect，则ping()会尝试依次reconnect，连接成功则返回true，否则报错。如果连接已断开，且未启用auto-reconnect，则报错。

## query()查询和结果处理

`query()`用于执行任何允许的SQL语句，比如执行查询语句。

查询结果可使用each进行迭代，迭代时传递查询到的每一行记录，可使用hash索引的方式(默认以hash类型保存每一行)查询某个字段的内容：

```ruby
conn.query("show databases").each do |row| pp row end
=begin
{"Database"=>"information_schema"}
{"Database"=>"mysql"}
{"Database"=>"mytest"}
{"Database"=>"performance_schema"}
{"Database"=>"sys"}
=end

conn.query("select * from mytest.tb").each do |row|
  pp row
  pp row["name"]
end
=begin
{"name"=>"junmajinlong", "age"=>23}
"junmajinlong"
{"name"=>"woniu", "age"=>25}
"woniu"
{"name"=>"fairy", "age"=>26}
"fairy"
=end
```

可见，查询结果中，每一行数据默认以hash格式保存。

实际上，对于增删改的SQL语句，query()的返回值为nil，对于查询类的语句，其返回值以`Mysql2::Result`对象返回

```ruby
conn.query("create table mytest.t1(id int)")
#=> nil
res = conn.query("select * from mytest.tb")
p res
```

结果：

```
#<Mysql2::Result:0x00007fffe833a230
 @query_options=
  {:as=>:hash,
   :async=>false,
   :cast_booleans=>false,
   :symbolize_keys=>false,
   :database_timezone=>:local,
   :application_timezone=>nil,
   :cache_rows=>true,
   :connect_flags=>2148540933,
   :cast=>true,
   :default_file=>nil,
   :default_group=>nil,
   :host=>"192.168.200.73",
   :username=>"root",
   :password=>"P@ssword1!"},
 @server_flags=
   {:no_good_index_used=>false, 
     :no_index_used=>true, 
     :query_was_slow=>false}>
```

`query()`各查询选项的含义以及默认的查询选项参见下文。先了解两个：
- `:as`项表示以数组方式(`:as=>:array`)还是hash方式(`:as=>:hash`)存储查询结果  
- `:symbolize_keys`表示返回hash结果时，其key是否设置为符号类型，默认为false，即key是字符串类型  

可在查询时指定这些参数，也可在each迭代时指定这些参数，例如：

```ruby
conn.query("select * from mytest.tb", symbolize_keys: true).each do |row|
  pp row
end

conn.query("select * from mytest.tb").each(symbolize_keys: true) do |row|
  pp row
end
```

结果：

```ruby
{:name=>"junmajinlong", :age=>23}
{:name=>"woniu", :age=>25}
{:name=>"fairy", :age=>26}
```

虽然大多数时候使用hash保存每一行数据更方便，但有时候也会想要以数组方式操作每一行数据，例如连接每一个字段的值：

```ruby
sql = 'select * from mytest.tb'
res = conn.query(sql)
res.each(as: :array) do |row|
  p row.join(",")
end
```

结果：

```
"junmajinlong,23"
"woniu,25"
"fairy,26"
```

`Mysql2::Result`自身具备的方法不多：

```
#count ⇒ Object (also: #size)
#each(*args) ⇒ Object
#fields ⇒ Object
#free ⇒ Object
```

另外，`Mysql2::Result`已经mix-in了Enumerable模块，所以可直接使用该模块中的方法，例如first表示取第一行记录。

```ruby
sql = 'select * from mytest.tb'
res = conn.query(sql)
res.first
#=>{"name"=>"junmajinlong", "age"=>23}
```

需注意，所有SQL语句中涉及到的值都是未经转义的，有时候需要也建议在执行它们之前先对它们进行转义。

```
escaped = conn.escape("gi'thu\"bbe\0r's")
results = conn.query("SELECT * FROM users WHERE group='#{escaped}'")
```

## 查询选项含义以及默认查询选项

query()默认的查询选项可以通过`Mysql2::Client.default_query_options`获取，它是一个hash结果：

```ruby
Mysql2::Client.default_query_options
=begin
{:as=>:hash,
 :async=>false,
 :cast_booleans=>false,
 :symbolize_keys=>false,
 :database_timezone=>:local,
 :application_timezone=>nil,
 :cache_rows=>true,
 :connect_flags=>2148540933,
 :cast=>true,
 :default_file=>nil,
 :default_group=>nil}
=end
```

其中(重要)：  
- `:as`项表示以数组方式(`:as=>:array`)还是hash方式(`:as=>:hash`)存储查询结果  
- `:symbolize_keys`表示返回hash结果时，其key是否设置为符号类型，默认为false，即key是字符串类型  
- `:async`表示查询是否异步模式，即是否非阻塞的查询，参考<https://github.com/brianmario/mysql2#async>  
- `:cast`指示MySQL的查询结果转换为Ruby数据时是否进行类型转换，如果确定本次查询的字段类型和Ruby的类型完全对应，可禁用casting功能提升效率  
- `:database_timezone`指示Ruby接收MySQL返回的日期时间数据时的时区，mysql2将先以该时区创建日期时间对象来保存对应字段的值。仅支持`:local`和`:utc`两个值 
- `:application_timezone`指示最终`Mysql2::Result`中的日期时间的时区，即程序端的时区。因此，mysql2先以"无损"的时区从MySQL获取日期时间数据，并根据application_timezone将其转换成程序端时区的日期时间对象  
- `:cache_rows`指示是否缓存构建出来的hash行或array行  
- Mysql2处理查询结果的流程：  
  - Mysql2的MySQL C api从MySQL服务端查询数据，并保存在Ruby的查询结果集(结果集属于C)  
  - `Mysql2::Result`和C端结果集是关联的，当释放`Mysql2::Result`，也会对C结果集进行GC  
  - Mysql2在需要取得结果集中的数据时(比如each迭代)，才从结果集中根据查询选项构建所需行并返回，比如构建hash结构的行，构建数组结构的行，构建hash结构时将key转换为Symbol类型等  
  - 默认情况下，从结果集中查询并构建出来的hash行或array行会缓存在Ruby中，使得下次再次请求这一行时(比如再次迭代)，可用直接从缓存中取得hash行或array行  
  - 比如从MySQL服务端查询了100行数据保存在C的结果集中，第一次以hash方式请求其中4行，这4行hash数据会缓存起来，如果下次再从头开始以hash方式请求15行，则前4行来自于缓存，后11行来自于结果集的临时构建  
  - 如果`:cache_rows`未禁用，当结果集中的所有行都被缓存，`Mysql2::Result`将会去释放C端的结果集  
  - 如果能确保查询的结果集只使用一次，可禁用`:cache_rows`，这会提升效率  
- `:stream: true`表示以Stream的方式处理查询结果。有时候查询结果数据量非常大，Ruby端不方便存放所有结果，可采用stream的方式去处理本次查询：一边从MySQL服务端取数据放进结果集，一边从结果集中取数据进行处理(比如迭代)。使用stream时，会自动关闭cache_rows，因为它们是互相冲突的概念。此外，使用stream模式要求必须迭代完所有数据集才会执行下一条查询，因为一个mysql连接在某一时刻只能执行一个操作，在迭代完之前，本次查询操作还尚未完成。

修改`Mysql2::Client.default_query_options`可以设置默认query()的查询选项。如果想要设置其中某选项，可以通过hash合并的方式来设置该选项。

```ruby
Mysql2::Client.default_query_options
#=> {:as=>:hash, ...}

Mysql2::Client.default_query_options.merge!(:as => :array)
#=> {:as=>:array, ...}
```

## prepare()+execute()

除了直接使用query()执行SQL语句查询数据库，也可以使用prepare()方法将字符串准备成一个待执行的SQL语句，其中可以使用`?`充当占位符。

prepare后的语句是一个`Mysql2::Statement`对象，该对象有一个execute()方法，可以用来执行这个准备好的语句，它可指定查询选项，且其返回值同query()一样：对于增删改操作，返回值为nil，对于查询类操作，返回`Mysql2::Result`结果对象。

```ruby
res_sql = conn.prepare('select * from mytest.tb where age >= ? and name like ?')
res = res_sql.execute(20, '%junma%', as: :array)
res.first
```

## 处理多结果集

有些存储过程中可能包含多个查询结果集，或者有时候会在一行SQL中包含多个select语句而同时返回多个结果集，Mysql2能很好地处理多结果集问题。

要处理多结果集，连接mysql时必须指定一个flag：

```ruby
conn = Mysql2::Client.new({
  host: "192.168.200.73",
  username: "root",
  password: "P@ssword1!",
  flags: Mysql2::Client::MULTI_STATEMENTS
})
```

然后执行多结果集的多个查询语句：

```ruby
res = conn.query('select 1;select 2;select 3')
```

虽然本次query()涉及了多个select语句，Mysql2也已经保存了这三个select的查询结果集(保存在结果集队列中)，但本次query()方法的返回值仅是第一个结果集，所以res中保存的是第一个结果集的内容。

```ruby
res.first   #=> {"1"=>1}
```

要获取剩余的结果集，可通过`conn.next_result`将结果集偏移指针移到下一个结果集，然后通过`conn.store_result`获取下一个结果集，依次类推，直到没有剩余结果集后，`conn.next_result`返回false。可通过`more_results?()`方法判断是否还有剩余的结果集。

```ruby
conn.next_result
res = conn.store_result
res.first #=> {"2"=>2}

conn.next_result
res = conn.store_result
res.first #=> {"3"=>3}

conn.next_result  #=> false
```

所以，可遍历多个结果集：

```ruby
res = conn.query('select 1;select 2;select 3')
loop do
  p res.first
  break unless conn.next_result
  res = conn.store_result
end

# 或者
p res.first
while conn.next_result
  res = conn.store_result
  p res.first
end
```

输出结果：

```
{"1"=>1}
{"2"=>2}
{"3"=>3}
```

需注意，开启多行语句(即多结果集)功能后，所查询得到的所有结果集必须已经处理完成(严格来说，是存放结果集的队列已经为空)，才能继续执行后续的SQL语句(事实上，经测试，结果集队列未空的情况下执行其它SQL语句会导致直接断开mysql连接)。可使用`abandon_results!()`方法强行丢弃所有剩余结果集，使得Mysql2马上回归正常状态：可向MySQL服务端发送SQL语句。

```ruby
# res和res1都只保存第一个结果集
# 但结果集队列中保留的是select 5和select 6的结果集
res = conn.query('select 1;select 2;select 3')
conn.abandon_results!  # 丢弃所有剩余结果集
res1 = conn.query('select 4;select 5;select 6')
```

另外，如果多个查询语句中间的某个查询语句报错，它将影响其后面的语句不会执行，所以无法获取后面的结果集。

```ruby
res = conn.query('select 1;select 2;select A;select 3')
loop do
  p res.first
  break unless conn.next_result
  res = conn.store_result
end
```

结果：

```
{"1"=>1}
{"2"=>2}
Mysql2::Error: Unknown column 'A' in 'field list'
```

## Mysql2的EventMachine

Mysql2支持EM，可以执行异步的query()，此外，可以指定当query()查询成功或失败时的回调语句块：

```ruby
require 'mysql2/em'

EM.run do
  client1 = Mysql2::EM::Client.new
  defer1 = client1.query "SELECT sleep(3) as first_query"
  defer1.callback do |result|
    puts "Result: #{result.to_a.inspect}"
  end

  client2 = Mysql2::EM::Client.new
  defer2 = client2.query "SELECT sleep(1) second_query"
  defer2.callback do |result|
    puts "Result: #{result.to_a.inspect}"
  end
end
```

## ORM之：Sequel

`Active:Record`应该是最为人熟知的orm，其功能极其丰富。

另一个轻量级的ORM是Sequel，它支持ADO, JDBC, MySQL, Mysql2, ODBC, Oracle, PostgreSQL,  SQLite3等等。

Sequel官方手册：  
- <https://github.com/jeremyevans/sequel>  
- <http://sequel.jeremyevans.net/rdoc/>  

例如：

```ruby
require 'sequel'

# 创建数据库实例
DB = Sequel.connect(
  adapter: :mysql2,
  user: 'root',
  password: 'P@ssword1!',
  host: '192.168.200.73',
  port: 3306,
  database: 'mytest'
)

# 创建数据集，数据集表示的是一张表或表部分数据
# 此时不会去查询数据，会推迟到需要数据时才查询
dataset = DB[:tb]

# 迭代表数据
dataset.each do |row|
  pp row
end

# 条件查询
pp dataset.where(name: 'junmajinlong', age: 23).first
pp dataset.where { name =~ "junmajinlong" and age =~ 23 }.first
```

