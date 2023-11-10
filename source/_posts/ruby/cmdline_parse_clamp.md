---
title: Ruby命令行解析工具Clamp用法详解
p: ruby/cmdline_parse_clamp.md
date: 2020-11-04 17:33:29
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby命令行解析工具Clamp用法详解

Ruby中可用于解析命令行选项参数的gem非常多，本文介绍其中一种比较好用的`Clamp`工具。

项目地址：<https://github.com/mdub/clamp>。

Clamp有一个比较严重的缺点：不能多选项互斥。例如某命令接受`--running`或`--stop`选项，但两者不能同时出现(即`cmd <--running|--stop>`)，Clamp自身无法实现这一点。

## 关于命令行解析的基本背景知识

对于下面的命令行来说：
```bash
cmd [-d]       -c <CONFIG_PATH>    -p <PID_PATH> TIMEOUT [USER]
cmd [-d] --config <CONFIG_PATH> --pid <PID_PATH> TIMEOUT [USER]

cmd -d -c /path/to/config --pid /path/to/pid 20 junmajinlong
cmd    -c /path/to/config --pid /path/to/pid 20 junmajinlong
cmd -d -c /path/to/config --pid /path/to/pid 20 
```

其中：  
- `-d`是不带参数的选项，即无参数型选项，也称为开关型选项  
- `-c /path/to/config`是带参数的短选项，其中`-c`是选项，`/path/to/config`是选项所需的参数  
- `--pid /path/to/pid`是带参数的长选项，其中`--pid`是选项，`/path/to/pid`是选项所需参数  
- `20`和`junmajinlong`是非选项型参数，也称为位置参数，它们不属于某个选项  
- `-d`选项是可选的，可以指定也可以省略  
- 第二个位置参数USER也是可选的  

多数命令都遵守GNU的命令行解析模式，这意味着选项和位置参数的位置可以随意变换。例如，上面命令的等价写法是：
```bash
cmd -c /path/to/config 20 --pid /path/to/pid junmajinlong -d
```

还有些命令可以使用子命令，例如git、ip命令等。
```bash
cmd config ...
cmd clone ...
cmd -o subcmd -s
cmd subcmd1 subcmd2
```

其中：  

![](/img/ruby/1604580563045.png)

## 选项(option)

clamp在`Clamp do...end`语句块中编写命令行的解析逻辑，也可以用定义类的方式编写命令行解析逻辑，`Clamp do...end`方式是后者的语法糖。
```ruby
require 'clamp'

Clamp do
  ...
end
```

在Clamp的语句块中，可使用`option`方法定义要解析的命令行选项，定义`execute()`方法表示执行命令。

Clamp会自动推导`-h, --help`的帮助信息，无需手动编写类似Usage的帮助信息。

### 带参数选项

a.rb中编写如下代码：
```ruby
require 'clamp'

Clamp do
  option "--config", "CONFIG", "specify config file"

  def execute
    puts "config file: #{config}"
    p config.class
  end
end
```

上面的`option`方法定义了如何解析长选项`--config`，第二个参数`CONFIG`和第三个参数用于在帮助信息中显示选项的值和选项的描述信息。

例如，执行该程序，并带上`--help`或`-h`选项，将自动输出帮助信息。
```bash
$ ruby a.rb --help
Usage:
    test.rb [OPTIONS]

Options:
    --config CONFIG    specify config file
    -h, --help         print help
```

当运行a.rb并带上选项`--config CONFIG`时，将执行execute()方法：
```bash
$ ruby a.rb --config /etc/a.conf
config file: /etc/a.conf
String
```

当使用`option`定义选项的时候，会自动根据选项名称进行推导，并定义一个变量(实际上是对象的属性而非变量，此处为了方便理解称之为变量)，然后将命令行属于该选项的参数值保存在该属性中，如果命令行中未指定该选项或者未设置该选项的参数，则该属性的值为nil。

例如，上面的示例中定义了选项`--config`，那么在execute()方法内就可以访问名为`config`的属性。
```bash
$ ruby test.rb --config /etc/a.conf
config file: /etc/a.conf
String

$ ruby test.rb --config  # 不指定--config选项的参数值
config file: 
NilClass

$ ruby test.rb   # 不指定--config选项
config file: 
NilClass
```

下面是一个更详细一点的示例：
```ruby
require 'clamp'

Clamp do 
  option ["-c", "--config"], "CONFIG", "specify config file"
  option ["-p", "--pid-file"], "PID_FILE", "specify pid file", required: true
  option "-d", "DATE", "specify pid file", attribute_name: "date"

  def execute
    puts "config file: #{config}"
    puts "pid file: #{pid_file}"
    puts "date: #{date}"
  end
end
```

对于上面的选项设置：
- 当`option`的第一个参数是数组时，表示定义别名选项  
- Clamp只会根据长选项推导属性名，不会根据短选项进行推导  
- 当长选项中包含短横线时，其推导的属性名将以下划线替代短横线，例如`--pid-file`对应的属性为`pid_file`  
- 可使用`:attribute_name`指定选项对应的属性名，此时将不会再根据长选项自动推导属性名  
- 当只设置了短选项而无长选项时，Clamp要求必须使用`:attribute_name`指定选项对应的属性名  
- 使用`required: true`可设置该选项是必须的选项，如果命令行中省略该选项或该选项的参数值，将直接报错提示  
- 没有设置`required: true`时，选项是可选的，不指定选项或不指定选项的参数值时，对应的属性值将设置为nil  

### 无参数选项(开关型选项)

将`option`的第二个参数设置为`:flag`，将表示该选项是一个开关型选项，该选项没有参数，是一个布尔类型的选项。

```ruby
Clamp do
  option ["-d", "--daemon"], :flag, "run in daemon mode"

  def execute
    p "daemon: #{daemon?}"
  end
end
```

对于无参数型选项，Clamp根据长选项名自动推导属性时，会在尾部加上`?`，所以这个属性实际上是一个方法。例如上面示例中`--daemon`对应的方法名为`daemon?`。

当命令行中指定了开关型选项时，属性将设置为true，如果省略，则其值为nil(而非false)。

此外，可方便地定义`--no-daemon`和`--daemon`：
```ruby
option "--[no-]daemon", :flag, "run in daemon mode"
```

此时，如果命令中给定了`--daemon`，则`daemon?`为true，如果指定了`--no-daemon`，则`daemon?`为false，如果没有指定该选项，则`daemon?`为nil。


### 可重复选项

设置`multivalued: true`后，表示可多次指定该选项，且所有该选项的值都会保存在`*_list`数组中，除非使用了`attribute_name`自定义了属性名。

例如：
```ruby
option "--format", "FORMAT", "output format", multivalued: true
```

上面的示例中，可在命令行中多次指定`--format`选项，它们的值保存在数组`format_list`中。

### 帮助信息中隐藏选项

有时候可能希望在帮助信息中隐藏某些选项。
```ruby
option "--some-option", "VALUE", "Just a little option", hidden: true
```

### option方法的语句块

`option`方法可以接语句块，该语句块在解析到命令行中该选项时立即执行。

例如：
```ruby
option "--version", :flag, "version info" do
  puts VERSION
  exit(0)
end
```

因为命令行中使用`--version`时，一般会立即退出。因此上面在语句块中加上了`exit()`使得解析到该选项时立即退出，而不会继续解析剩余的选项和参数。

## 非选项参数(位置参数, parameter)

通过`option`定义如何解析命令行的选项部分，通过`parameter`方法定义如何解析命令行的非选项型参数部分。

用法比较简单，看示例。

```ruby
require 'clamp'

Clamp do 
  option ["-c", "--config"], "CONFIG", "specify config file"
  option ["-p", "--pid-file"], "PID_FILE", "specify pid file", required: true
  option ["-d", "--daemon"], :flag, "run in daemon mode"
  parameter "TIMEOUT", "specify timeout"
  parameter "[USER]", "specify username"

  def execute
    puts "timeout: #{timeout}"
    puts "user: #{user}"
  end
end
```

上面定义了如何解析命令行中的前两个位置参数：TIMEOUT和USER，其中USER参数必须在TIMEOUT参数后。

Clamp也会自动推导参数对应的属性名，推导规则是将大写转换为小写。所以上面的示例会为这两个参数分别定义`timeout`属性和`user`属性。

没有使用中括号包围的`TIMEOUT`参数是必须的，命令行中省略TIMEOUT参数就会报错。

使用中括号包围的`[USER]`表明USER是可选参数，当命令行中省略该参数时，其对应的属性值为nil。

### 数量不定的位置参数

有时候位置参数的数量是不固定的，比如在命令行指定所有要处理的文件、要安装的插件名等。

当`parameter`方法的第一个参数的格式为`XXX ...`时，表示它将解析剩余所有的位置参数。Clamp默认为其推导的属性名为`XXX_list`，当然，可以使用`attribute_name`指定属性名。

```ruby
parameter "FILE ...", "input files", attribute_name: :files
```

## 选项和参数的更多设置

### 选项和参数的默认值

对于带参数的选项以及可选型位置参数，可以为它们指定默认值。

```ruby
option "--port", "PORT", "port for listening", default: 80
parameter "HOST", "server host", default: "localhost"
```

此外，还可以更灵活的通过定义`default_xxx`方法来指定默认值。其中xxx是属性名，方法返回值是默认值。

```ruby
option "--port", "PORT", "port for listening", default: 9000
option "--admin-port", "PORT", "admin port"

def default_admin_port
  port + 1
end
```

### 从环境变量获取参数值

对于带参数的选项以及可选的位置参数，可通过`:environment_variable`参数来表示从指定的环境变量上获取参数值。

```ruby
option "--port", "PORT", "port to listen on", environment_variable: "APP_PORT" do |val|
  Integer(val)
end

parameter "[HOST]", "server address", environment_variable: "APP_HOST"
```

一般没有必要同时设置默认值和从环境变量获取值，如果同时设置了，Clamp会优先从环境变量获取值。
```ruby
require 'clamp'

Clamp do
  parameter "[USER]", "specify username", default: "root", environment_variable: "APP_USER"

  def execute
    puts "user: #{user}"
  end
end
```

执行：
```bash
$ ruby a.rb
user: root

$ ruby a.rb gaoxiaofang
user: gaoxiaofang

$ APP_USER=malongshuai ruby a.rb
user: malongshuai
```

### 允许选项和位置参数在任意位置

很多命令的选项和参数的位置是不固定的。

Clamp默认只允许解析选项在位置参数前的命令行。但如果想要允许选项和位置参数的位置任意，许设置：
```ruby
Clamp.allow_options_after_parameters = true
```

例如：
```ruby
Clamp do 
  Clamp.allow_options_after_parameters = true

  option ["-c", "--config"], "CONFIG", "specify config file"
  option ["-p", "--pid-file"], "PID_FILE", "specify pid file", required: true
  parameter "TIMEOUT", "specify timeout"
  parameter "[USER]", "specify username", default: "root"

  def execute
    puts "config file: #{config}"
    puts "pid file: #{pid_file}"
    puts "timeout: #{timeout}"
    puts "user: #{user}"
  end
end
```

那么，下面的命令行都是正确的：
```bash
$ ruby a.rb -c a.conf -p a.pid 30 junmajinlong
$ ruby a.rb -c a.conf 30 -p a.pid junmajinlong
$ ruby a.rb -c a.conf 30 junmajinlong -p a.pid
```

## 对参数值做进一步处理

### 选项和参数的类型转换和有效性验证

对于存在性来说，只有选项需要验证，因为没有加中括号的位置参数是必须提供的，它是自验证的。

但可能还要验证其他方面的问题，例如数据类型，数据范围等。

`option`和`parameter`方法都可以接语句块。如果给定了语句块，则语句块在解析到该选项或该参数时立即执行，且将选项的参数值或位置参数的值传递给语句块变量。因此可以在语句块中对传递的参数值做一些最基本的验证。

如果解析的是可重复的选项或解析的是剩余所有位置参数，它们的参数值都保存在数组中，此时语句块会迭代数组中所有的值并执行语句块。

另外，语句块的返回值会重新作为选项的值或位置参数的值保存在对应属性中，这对于做数据转换是非常方便的，例如将字符串格式的日期时间转换为Ruby的日期时间对象。

```ruby
option "--port", "PORT", "port to listen" do |s|
  # 将参数转换成整数，并保存在属性port中
  # 如果转换失败，则Integer()会报错，注意"str".to_i()不会报错
  Integer(s)
end

parameter "TIMEOUT", "specify timeout" do |t|
  Integer(t)
end
```

在`option`或`parameter`的语句块中一般只做数据的转换工作，而选项或位置参数的有效性验证则放在execute()方法中进行。如果验证失败，则可以通过`signal_usage_error`方法来抛出参数错误的信息。

```ruby
def execute
  if not (0..100).include? timeout
    signal_usage_error "timeout show in (0,100)"
  end
  ...
end
```

### 参数重新赋值

Clamp推导或`attribute_name`所指定的属性名，其实是对象的属性。Clamp会自动将这些属性设置为`attr_accessor`。

因此，除了可以通过属性名访问它们，还可以通过属性的setter方法为它们重新赋值。不过这个setter方法需要自定义。

例如，将带有端口号的主机名的值`hostname:port`传递给命令行中的`--server`选项，现在要将它们分别保存在hostname变量和port变量中。

```ruby
Clamp do
  option "--server", "SERVER", "server address"

  # setter方法的参数值是原值，即来自于命令行的字符串
  # 注意，要保存为实例变量，execute中才可以访问
  def server=(server)
    @host, @port = server.split(":")
    @port ||= 80
  end
end
```

## Clamp定义子命令和嵌套多层子命令

Clamp也支持定义子命令，且支持多层嵌套子命令。

例如：
```bash
$ vagrant up
$ vagrant up NAME   # NAME是子命令up的可选参数
$ vagrant box list -h  # list是嵌套子命令，-h是list的选项
```

![](/img/ruby/1604580604474.png)

例如：
```ruby
Clamp do
  # 位置参数和选项的位置可随意
  # 无论在哪里设置该属性，都直接影响全局
  Clamp.allow_options_after_parameters = true

  # 全局选项[-c, --config]
  # 当allow_options_after_parameters为true时，全局选项
  # 可出现在任意位置，即使是子命令后面
  option ["-c", "--config"], "CONFIG", "specify config file"
  
  # 子命令 remove，接受选项[-f, --force]和可选参数NAME
  subcommand 'remove', "remove app" do
    option ["-f", "--force"], :flag, "force to remove"
    parameter "[NAME]", "remove this app", default: "default"
    def execute
      puts "remove subcommand: "
      puts "    force: #{force?}"
      puts "    name: #{name}"
      puts "    config: #{config}"
    end
  end

  # 子命令box
  subcommand 'box', "box manage" do
    # box的子命令list
    subcommand "list", "display boxes" do
      # list接受选项[-v, --verbose]和可选型参数NAME
      option ["-v", "--verbose"], :flag, "print more info"
      parameter "[NAME]", "display this box"

      def execute
        puts "list subcommand: "
        puts "    verbose: #{verbose?}"
        puts "    name: #{name}"
        puts "    config: #{config}"
      end
    end
  end
end
```

Clamp的帮助信息很智能，会自动列出指定层次的帮助信息：
```bash
$ ruby a.rb --help
$ ruby a.rb remove --help
$ ruby a.rb box --help
$ ruby a.rb box list --help
```

### 默认子命令

有时候可能需要设置默认子命令的功能。

```ruby
Clamp do 
  self.default_subcommand = "status"
  
  subcommand "statu", "display status" do
    def execute
      ...
    end
  end
end
```

### 子命令缺失

Clamp允许定义`subcommand_missing`方法来处理命令行上传递了不支持的子命令。

```ruby
Clamp do
  def subcommand_missing(name)
    if name == "foo"
      abort "Subcommand 'foo' not defined"
    end
  end
end
```





