---
title: 使用bashly构建bash脚本命令行
p: shell/bashly.md
date: 2022-09-17 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------


# 使用bashly构建bash脚本命令行

众所周知，使用比较古老的shell(比如bash)去编写带命令行选项的脚本是非常让人头疼的事情，比较现代的shell，在设计命令行选项方面会更简单一些，比如现在已经烧出了一点火势的更现代化的nushell，它对编写命令行选项的支持力度就非常不错。

## Bash设计命令行选项参数的苦恼

以Bash为例，如果脚本的选项参数比较简单，倒也能接受，通常会采用`while + case`来处理选项和参数：

```bash
while [ $# -gt 0 ];do
  case "$1" in
    -a | --aaa)
      arg_a = $2;
      shift
      shift
      ;;
    -b | --bbb)
      arg_b = 1;
      shift
      ;;
      ......
    *)
      usage
      exit 1
  esac
done
```

如果选项和参数的逻辑稍微再复杂一点，一般会结合`getopt`命令或`getopts`命令先收集并整理参数，再结合`while + case`来处理整理后的选项和参数。

如果逻辑再复杂一些，我一般不会再有心思去设计一个健壮的、完整的shell脚本的命令行选项，这会花费大量时间，并且很难让它健壮，因此会让我在心态上就产生退却感：只是一个shell脚本，将就用用吧。

我个人比较常写shell脚本，也经常需要设计脚本的命令行选项，也常常因此而有些烦恼：设计shell脚本的命令行太耗时太啰嗦了。

## 谁能拯救我，唯有bashly

bashly是一个用来构建bash脚本命令行选项的框架(算是脚手架吧，没想到吧，bash竟然都有脚手架)。通过在yaml格式的配置文件中编写命令行选项和参数以及其它和命令行有关的信息，就可以生成一个功能完整的命令行脚本框架，在生成的框架结构下，用户只需填补命令行的运行逻辑即可。

从我看它的文档开始，我就迷上它了。我可以肯定(而且现在已经肯定)，它丝毫不逊色于通用编程语言的那些优秀的用来设计命令行选项的第三方库，甚至我觉得学习时比它们要更简单更统一。尽管刚开始看到它的时候我很惊讶，因为它颠覆了我一贯的看法：bash这个丑陋的老祖宗只能用非常古老和繁琐的手法来供着。

> 注意，bashly只适配bash，且要求bash版本高于4.0

下面给一个非常简单的示例过过眼，之后再详细介绍bashly的用法和功能。

假设现在已经安装好了bashly(后文介绍安装方式)，执行如下命令初始化脚本项目(项目目录`bashly_test`):

```bash
# 创建项目目录
$ mkdir bashly_test && cd bashly_test

# 最小化的命令行选项参数配置模板
$ bashly init --minimal
```

`bashly init`将会在项目目录下生成`src/bashly.yml`(目前仅生成了这一个文件)，要设计脚本的选项和参数，编辑这个bashly.yml配置文件即可。目前这个自动生成的文件内容为：

```yml
name: download
help: Sample minimal application without commands
version: 0.1.0

args:
- name: source
  required: true
  help: URL to download from
- name: target
  help: "Target filename (default: same as source)"

flags:
- long: --force
  short: -f
  help: Overwrite existing files

examples:
- download example.com
- download example.com ./output -f
```

这个文件的内容表示，脚本命令的名称是`download`，有一个选项和两个参数，用法在`examples`字段已经非常明显。

当然，现在只生成了这一个文件，并没有生成download命令。

当编写好或修改好bashly.yml之后，就可以通过bashly的`generate`子命令来生成download命令：

```bash
$ bashly generate
```

这将会生成如下目录结构：

```bash
$ tree .
.
├── download
└── src
    ├── bashly.yml
    ├── initialize.sh
    └── root_command.sh
```

download命令已经生成好了，在项目根目录之下，且具备执行权限。它是一个完整的bash编写的脚本，可以单独拷贝到任何地方去执行。

执行download命令：

```bash
# --help查看帮助信息
$ ./download --help
download - Sample minimal application without commands

Usage:
  download SOURCE [TARGET] [options]
  download --help | -h
  download --version | -v

Options:
  --help, -h
    Show this help

  --version, -v
    Show version number

  --force, -f
    Overwrite existing files

Arguments:
  SOURCE
    URL to download from

  TARGET
    Target filename (default: same as source)

Examples:
  download example.com
  download example.com ./output -f

# 不带任何参数，将报错，因为SOURCE参数是必须的
$ ./download 
missing required argument: SOURCE
usage: download SOURCE [TARGET] [options]

# 带两个参数和-f选项
$ ./download income outcome -f
# this file is located in 'src/root_command.sh'
# you can edit it freely and regenerate (it will not be overwritten)
args:
- ${args[--force]} = 1
- ${args[source]} = income
- ${args[target]} = outcome
```

最后执行`./download income outcome -f`的输出结果，是未做任何修改的初始模板运行逻辑，逻辑部分是可以修改的。后文再详细介绍如何修改逻辑。

从运行download命令的结果可以看到，bashly根据yml配置文件生成的download脚本，其命令行选项的功能很不错，确实是那个我们熟悉的味道。

## 安装bashly

最简单的方式是直接使用已经配置好bashly环境的docker镜像：

```bash
alias bashly='docker run --rm -it --user $(id -u):$(id -g) --volume "$PWD:/app" dannyben/bashly'
```

如果要在本机使用bashly，按照下面的步骤操作。

确保已经安装bash 4.0以上的版本，如果系统里没有bash或者版本低于4.0，自行搜索安装方法。我更建议直接换个自带bash 4.0以上版本的系统，之后再将bashly生成好的脚本命令拷贝到没有bash的环境。

bashly是Ruby编写的，bashly运行时也需要Ruby，因此先安装Ruby(要求版本号大于2.7)，再通过Ruby的包管理器gem(安装Ruby的同时会安装好gem)来安装bashly。

可以先查看包管理器提供的官方Ruby版本是否大于2.7，如果版本高于2.7，可直接通过包管理器快速安装：

```bash
# 以ubuntu为例
$ apt show ruby
Package: ruby
Version: 1:2.7+1
Priority: optional
Section: interpreters
Source: ruby-defaults
......
```

如果包管理器提供的Ruby版本不够，考虑使用rbenv来管理安装Ruby。关于安装rbenv和安装Ruby的方法，参考我那古老的文章：<https://www.junmajinlong.com/ruby/ruby_rails_install/>。  

假如已经安装好了Ruby和gem，如下方式安装bashly：

```bash
$ gem install bashly
```

## 简单分析bashly生成的文件内容

bashly根据配置文件生成了模板命令之后，需要手动去填充命令的运行逻辑。

以前文生成的download命令为例，生成download后的目录结构如下：

```bash
$ tree .
.
├── download
└── src
    ├── bashly.yml
    ├── initialize.sh
    └── root_command.sh
```

除了download外，还生成了root_command.sh和initialize.sh两个文件。

- initialize.sh中编写用于环境初始化的shell代码
- root_command.sh中编写该命令行的本身逻辑

initialize.sh文件初始时是空的(只有一些注释行)。需注意，所有`##`开头的被认为是bashly的注释信息，是完全被忽略的，而只有一个`#`的注释行，将会被写入download文件。

```bash
## Code here runs inside the initialize() function
## Use it for anything that you need to run before any other function, like
## setting environment vairables:
## CONFIG_FILE=settings.ini
##
## Feel free to empty (but not delete) this file.
```

> 注：新版本的bashly不会主动生成initialize.sh，请参考下文[Bashly Hook](#bashly_hook)关于initialize.sh的更多说明。

再看root_command.sh文件。root_command.sh文件就是填充命令行运行逻辑的地方。该文件的初始内容：

```bash
echo "# this file is located in 'src/root_command.sh'"
echo "# you can edit it freely and regenerate (it will not be overwritten)"
inspect_args
```

两个`echo`命令和一个`inspect_args`命令，`inspect_args`命令是bashly自动生成的用来查看选项和值信息的函数，该函数被定义在`download`文件中。

root_command.sh文件的内容在每次执行`bashly generate`的时候都会自动填充到`download`文件中。因此，在没有对root_command.sh文件做出任何修改的时候，执行download命令的输出结果正是这里初始化的内容：

```bash
$ ./download income outcome -f
# this file is located in 'src/root_command.sh'
# you can edit it freely and regenerate (it will not be overwritten)
args:
- ${args[--force]} = 1
- ${args[source]} = income
- ${args[target]} = outcome
```

最后，看一下bashly生成的download文件的内容(比较长，因此删除一部分)：

```bash
#!/usr/bin/env bash

# root_command.sh文件中的内容被拷贝到这里面
root_command() {
  # src/root_command.sh
}

# 输出命令的用法帮助信息
download_usage() {
  ......
}

# 收集命令行的输入选项和参数
normalize_input() {
  local arg flags

  while [[ $# -gt 0 ]]; do
    ......
  done
}

# 查看命令行的输入选项和参数信息
inspect_args() {
  ......
}

# 解析收集好的选项和参数
parse_requirements() {
  ......
  action="root"

  while [[ $# -gt 0 ]]; do
    key="$1"
    case "$key" in
      ......
    esac
  done
}

# 初始化，initialize.sh文件中的内容被拷贝到这里
initialize() {
  # src/initialize.sh
}

# 执行命令逻辑
run() {
	# ★★★★★★ 重点重点重点 ★★★★★★
	# args数组包含了所有解析完成后的选项和选项的值
	# other_args数组包含了解析完成后剩余的所有参数值
	# input数组包含选项参数解析前从命令行中收集到的所有参数信息
	# ★★★★★★ 重点重点重点 ★★★★★★
  declare -A args=()
  declare -a other_args=()
  declare -a input=()
  normalize_input "$@"
  parse_requirements "${input[@]}"

  if [[ $action == "root" ]]; then
    root_command
  fi
}

# 先初始化，再执行逻辑
initialize
run "$@"
```

因此，bashly生成的脚本命令的逻辑很简单：

- (initialize函数)执行初始化操作(操作内容来自src/initialize.sh文件)  
- (normalize_input函数)收集所有命令行中的选项和参数，放入`input`数组  
- (parse_requirements函数)处理input数组，将选项和选项的值放进`args`数组，将剩余的参数放进`other_args`数组  
- (root_command函数)执行代码逻辑(代码逻辑来自src/root_command.sh文件)  

> 注意，在本示例中生成的download文件中没有填充`other_args`数组的代码，因为本示例的download命令不允许额外参数。如果要允许额外参数，需在配置文件中使用`catch_all`指令。后文会介绍该指令。

当需要编写自己的命令行选项逻辑时，应该将相关的逻辑填充到`src/root_command.sh`文件中，在这过程中，最重要的就是从名为`args`的数组获取选项和参数的值。

## bashly如何定义Command

### 定义命令行时的常用指令

在bashly.yml文件中，大概以这种方式定义一个命令行选项和参数：

```yaml
name:  cli
help:  这是一个命令行
version: 0.0.1
flags:
  - long: --flag1
    short: -f
    help: 这是flag1
  - long: --flag2
    short: -F
    arg: FLAG2
    help: 这是flag2, 该选项需要一个参数
  - long: --flag3
    required: true
    help: 这是flag3, 该选项没有对应的短选项，且该选项不可省略 
args:
  - name: arg1
    required: true
    help: 这是参数1，不可省略
  - name: arg2
    help: 这是参数2，可以省略
examples:
  - cli -f -F F_arg --flag3 arg1 arg2
  - cli --flag3 arg1
footer: |
  这是footer信息，在help信息的下方显示，
  可以使用多行
```

根据这个配置文件，生成命令行：

```bash
$ bashly generate
$ ./cli --help
cli - 这是一个命令行

Usage:
  cli ARG1 [ARG2] [options]
  cli --help | -h
  cli --version | -v

Options:
  --help, -h
    Show this help

  --version, -v
    Show version number

  --flag1, -f
    这是flag1

  --flag2, -F FLAG2
    这是flag2, 该选项需要一个参数

  --flag3 (required)
    这是flag3, 该选项没有对应的短选项，且该选项不可省略

Arguments:
  ARG1
    这是参数1，不可省略

  ARG2
    这是参数2，可以省略

Examples:
  cli -f -F F_arg --flag3 arg1 arg2
  cli --flag3 arg1

这是footer信息，在help信息的下方显示，
可以使用多行
```

### bashly定义子命令(sub command)

bashly定义子命令的方式出乎意料的简单，直接在command下使用`commands`指令就可以。

需要注意，在使用bashly定义子命令时，是不支持父命令带有其它选项的。也就是说，bashly.yml文件中`commands`指令的同一层次，不允许和`args`指令共存，如果共存将报错，但是可以和`flags`指令共存，此时flags指定的是这些子命令的全局选项，全局选项需在命令行的子命令之前指定，例如`cli -w a subcmd`，此处`-w VALUE`是子命令的全局选项。

另外，子命令可以嵌套，即子命令中定义子命令。

```yaml
name: cli
help: 这是一个命令行
version: 0.0.1
commands:
  - name: sub1
    help: 这是子命令1
    flags:
      - short: -f
        help: 子命令1的选项1，该选项只有短选项
      - long: --FFF
        short: -F
        help: 子命令1的选项2
    args:
      - name: arg1
        help: 子命令1的参数
  - name: sub2
    help: 这是子命令2
    flags:
      - short: -f
        help: 子命令2的选项1，该选项只有短选项
      - long: --FFF
        short: -F
        help: 子命令2的选项2
    args:
      - name: arg1
        help: 子命令2的参数
```

生成：

```bash
$ bashly generate

# 父命令自身的帮助信息
$ ./cli --help
cli - 这是一个命令行

Usage:
  cli [command]
  cli [command] --help | -h
  cli --version | -v

Commands:
  sub1   这是子命令1
  sub2   这是子命令2

Options:
  --help, -h
    Show this help

  --version, -v
    Show version number

# 子命令的帮助信息
$ ./cli sub1 --help
cli sub1 - 这是子命令1

Usage:
  cli sub1 [ARG1] [options]
  cli sub1 --help | -h

Options:
  --help, -h
    Show this help

  -f
    子命令1的选项1，该选项只有短选项

  --FFF, -F
    子命令1的选项2

Arguments:
  ARG1
    子命令1的参数
```

需注意，使用子命令时，在`bashly generate`的时候会为每一个子命令都生成一个与子命令对应的shell脚本文件，这些文件专门用来编写只属于该子命令的逻辑代码。

```bash
$ tree .
.
├── cli
└── src
    ├── bashly.yml
    ├── initialize.sh
    ├── sub1_command.sh  # sub1子命令的逻辑，写在这个文件中
    └── sub2_command.sh  # sub2子命令的逻辑，写在这个文件中
```

由于定义了子命令而没有定义父命令，因此没有生成root_command.sh文件。

### alias指令：子命令别名

alias指令定义子命令的别名，只能用于子命令中。

有以下几种别名定义方式：

```yaml
name: index
alias: i  # 定义具体的别名名称

name: download
alias: d*  # 任何以d开头的都是别名，都等价于download

name: upload
alias: [u, push]  # 数组内的所有元素都是别名
```

例如：

```yaml
name: cli
help: Sample application
version: 0.1.0
commands:
- name: download
  alias: d
  help: Download a file
  args:
  - name: source
    required: true
    help: URL to download from
  - name: target
    help: "Target filename (default: same as source)"
- name: upload
  alias: [u, push]
  help: Upload a file
  args:
  - name: source
    required: true
    help: File to upload
```

对于上面配置文件生成的脚本文件，它：

```bash
# 等价
cli download example.com ./output
cli d        example.com ./output
  
# 等价
cli upload README.md
cli push README.md
cli u README.md
```

### group指令：子命令分组显示

如果子命令数量较多，在输出的帮助信息中，可以按照类别进行分类显示。

```yaml
name: ftp
help: Sample application with command grouping
version: 0.1.0
commands:
- name: download
  help: Download a file
  group: File
  args:
  - name: file
    required: true
    help: File to download
- name: upload
  help: Upload a file
  group: File
  args:
  - name: file
    required: true
    help: File to upload

- name: login
  help: Write login credentials to the config file
  group: Login
- name: logout
  help: Delete login credentials to the config file
  group: Login
```

输出帮助信息：

```bash
$ ./ftp --help
ftp - Sample application with command grouping

Usage:
  ftp [command]
  ftp [command] --help | -h
  ftp --version | -v

File Commands:
  download   Download a file
  upload     Upload a file

Login Commands:
  login      Write login credentials to the config file
  logout     Delete login credentials to the config file

Options:
  --help, -h
    Show this help

  --version, -v
    Show version number
```

### 嵌套子命令

下面是子命令中可以嵌套子命令的示例，用法之一是`./get_ks url -w spot cm -f 2022-03-22`。

```yaml
name: get_ks
help: 解析和下载数据, 并保存和转换
version: 0.1.0
commands:
  - name: url
    alias: u
    help: 分析待下载的URL并输出至标准输出
    flags:
      # url子命令的选项，即嵌套子命令的全局选项-w
      - long: --which
        short: -w
        arg: WHICH
        allowed: [spot, future, all]
        help: spot还是future, 或是两者
    # 子命令中嵌套子命令
    commands:
      - name: history
        alias: hs
        help: 返回所有历史数据
      - name: check-missed
        alias: cm
        help: 检查缺失数据
        flags: 
          - long: --from
            short: -f
            arg: Date
            help: 从什么时候开始检查
```


### environment_variables指令：设置环境变量

`environment_variables`用来设置命令执行时的环境变量。

```yml
environment_variables:
  - name: config_path
    help: Location of the config file
    default: ~/config.ini
  - name: api_key
    help: Your API key
    required: true
```

当将某环境变量设置为required，如果在运行时如果没有提供该环境变量，脚本的执行将报错。

可以在bashly.yml的命令顶层使用该指令指定整个命令的环境变量，也可以在子命令层指定只有运行该子命令时的环境变量。

```yaml
name: ftp
help: Sample application with command grouping
version: 0.1.0

environment_variables:
  - name: config_path
    help: Location of the config file
    default: ~/config.ini

commands:
- name: login
  help: Write login credentials to the config file
  environment_variables:
  - name: api_key
    help: Your API key
    required: true
- name: logout
  help: Delete login credentials to the config file
```

查看帮助信息：

```bash
$ ./ftp --help                                                                                                                                        
ftp - Sample application with command grouping

Usage:
  ftp [command]
  ftp [command] --help | -h
  ftp --version | -v

Commands:
  login    Write login credentials to the config file
  logout   Delete login credentials to the config file

Options:
  --help, -h
    Show this help

  --version, -v
    Show version number

Environment Variables:
  CONFIG_PATH
    Location of the config file
    Default: ~/config.ini

# 子命令的帮助信息
$ ./ftp login --help
ftp login - Write login credentials to the config file

Usage:
  ftp login
  ftp login --help | -h

Options:
  --help, -h
    Show this help

Environment Variables:
  API_KEY (required)
    Your API key
```

### private指令：隐藏某个子命令

`private`指令可在输出帮助信息时隐藏某个子命令。

```yaml
name: cli
help: Sample application with private commands
version: 0.1.0
commands:
- name: connect
  alias: c
  help: Connect to the metaverse
  args:
  - name: protocol
    required: true
    allowed: [ftp, ssh]
    help: Protocol to use for connection
- name: connect-ftp
  help: Connect via FTP
  private: true
- name: connect-ssh
  help: Connect via SSH
  private: true
```

输出帮助信息：

```bash
$ ./cli --help
cli - Sample application with private commands

Usage:
  cli [command]
  cli [command] --help | -h
  cli --version | -v

Commands:
  connect   Connect to the metaverse

Options:
  --help, -h
    Show this help

  --version, -v
    Show version number
```

### catch_all指令：允许定义之外的参数

在没有使用`catch_all`的情况下，bashly解析命令行选项时，要求命令行中所给的参数必须完全符合所定义的选项和参数，不允许多提供无法识别的参数。

例如，对于这个配置文件：

```yaml
name: cli
help: 这是一个命令行
version: 0.0.1
flags:
  - long: --flag1
    short: -f
    help: 这是flag1
  - long: --flag2
    short: -F
    arg: FLAG2
    help: 这是flag2, 该选项需要一个参数
args:
  - name: arg1
    required: true
    help: 这是参数1，不可省略
  - name: arg2
    help: 这是参数2，可以省略
```

该命令只能接受最多两个特定的选项，最多两个参数，如果再多传递一个参数，就会报错：

```bash
$ ./cli -f -F a a1 a2
# this file is located in 'src/root_command.sh'
# you can edit it freely and regenerate (it will not be overwritten)
args:
- ${args[--flag1]} = 1
- ${args[--flag2]} = a
- ${args[arg1]} = a1
- ${args[arg2]} = a2

# 报错，多提供了一个参数
$ ./cli -f -F a a1 a2 a3
invalid argument: a3

# 报错，多提供了一个选项
$ ./cli -f -F a a1 a2 -a
invalid option: -a
```

如果使用`catch_all`指令，则可以提供更多参数而不报错：

```yaml
name: cli
help: 这是一个命令行
version: 0.0.1
catch_all: true
flags:
  - long: --flag1
    short: -f
    help: 这是flag1
  - long: --flag2
    short: -F
    arg: FLAG2
    help: 这是flag2, 该选项需要一个参数
args:
  - name: arg1
    required: true
    help: 这是参数1，不可省略
  - name: arg2
    help: 这是参数2，可以省略
```

执行：

```bash
# 多提供一个参数，不报错，多余的参数被收集到other_args数组
$ ./cli -f -F a a1 a2 a3
# this file is located in 'src/root_command.sh'
# you can edit it freely and regenerate (it will not be overwritten)
args:
- ${args[--flag1]} = 1
- ${args[--flag2]} = a
- ${args[arg1]} = a1
- ${args[arg2]} = a2

other_args:
- ${other_args[*]} = a3
- ${other_args[0]} = a3

# 多提供一个选项，不报错，多余的选项被收集到other_args数组
$ ./cli -f -F a a1 a2 -a
# this file is located in 'src/root_command.sh'
# you can edit it freely and regenerate (it will not be overwritten)
args:
- ${args[--flag1]} = 1
- ${args[--flag2]} = a
- ${args[arg1]} = a1
- ${args[arg2]} = a2

other_args:
- ${other_args[*]} = -a
- ${other_args[0]} = -a
```

从结果看到，额外的参数都会被收集到`other_args`数组中。

`catch_all`指令有三种设置方式：

```yaml
# 方式1：只是开启，帮助信息中显示[...]
catch_all: true

# 方式2：开启并在帮助信息中显示[LABEL_NAME...]
catch_all: label_name

# 方式3：开启，并在帮助信息中显示[LABEL_NAME...]或LABEL_NAME...
# 如果提供了help，则还在显示参数的地方再显示该信息
# required默认为false，表示额外的参数是可选的，
# 如果requied设置为true，则必须提供至少一个额外的参数
catch_all:
  label: label_name
  help: help message
  required: true
```

看输出示例：

```bash
# 方式1输出信息：
Usage:
  cli ARG1 [ARG2] [options] [LABEL_NAME...]
  cli --help | -h
  cli --version | -v

# 方式2输出信息：
Usage:
  cli ARG1 [ARG2] [options] [LABEL_NAME...]
  cli --help | -h
  cli --version | -v

# 方式3，如果只提供label，等价于方式2
# 方式3，提供help
Usage:
  cli ARG1 [ARG2] [options] [LABEL_NAME...]
  cli --help | -h
  cli --version | -v
Options:
  ...
Arguments:
  ...
  LABEL_NAME...
    help message

# 方式3，提供help和required设置为true
Usage:
  cli ARG1 [ARG2] [options] LABEL_NAME...
  cli --help | -h
  cli --version | -v
Options:
  ...
Arguments:
  ...
  LABEL_NAME...
    help message
```

### dependencies指令：外部命令依赖

通过`dependencies`指令，可以指定该命令或该子命令依赖于某些外部命令，如果外部命令不存在，则执行将会报错。

```yaml
name: cli
help: Sample application that requires dependencies
version: 0.1.0
commands:
- name: download
  help: Download something
  dependencies:
  - git
  - curl
  - shmurl
- name: upload
  help: Upload something
```

### extensible指令：委托给外部命令

类似于`git`，当执行`git something`的时候，会转而执行`git-something`命令。通过`extensible`指令，也可以实现这样的效果。

有两种委托方式：

```yaml
# 方式1：设置为true，
# 执行mygit something的时候，会查找mygit-something命令来执行
name: mygit
extensible: true

# 方式2：设置为某个具体的值
# 执行mygit something的时候，实际上会执行git something
name: mygit
extensible: git
```

`extensible`不能和`default`共存，因为它们的逻辑存在冲突。

## bashly如何定义选项

bashly通过`flags`指令定义选项，在`flags`指令之下，支持下面这些指令：

- `long`：指定长选项的名称(和short至少提供一个)  
- `short`：指定短选项的名称(和long至少提供一个)  
- `help`：选项说明
- `arg`：如果选项需要参数，加上该指令，该指令指定参数显示名称。不加该指令，则表示选项不需要参数  
- `default`：为该选项指定默认的参数值，需同时指定`arg`指令  
- `required`：该选项是否必须存在，设置为false(默认)表示该选项可以省略  
- `allowed`：一个数组，该选项的参数的值必须是该数组中的一个，需同时指定`arg`指令，可同时结合default或required指令来使用  
- `conflicts`：一个数组，明确指定该选项和哪些选项冲突，即不能和哪些选项共存。使用该指令时，应当在所有互相冲突的选项上都指定该指令  
- `repeatable`：该选项可以重复出现多次，如果使用短选项没有参数，则可以结合，例如`-v -v`等价于`-vv`，长选项或者带有参数时，必须分开多次指定，例如`--f1 --f1`和`-v v1 -v v2`。当带有参数时，各参数将以引号包围并空格分隔，因此应通过类似于`eval "datas=(${args[--data]})"`的方式将各参数提取出来保存在另一个数组中，然后访问该数组获取各值。参考<https://bashly.dannyb.co/configuration/flag/#repeatable)>  
- `unique`：必须配合`repeatable`指令使用，且该选项必须带参数，即含有`arg`指令。指定该指令时，如果重复选项的参数值也重复了，将忽略所有重复参数值  
- `validate`：对参数进行验证，需同时指定`arg`指令。如何进行验证，参考后文[validate：验证参数](#validate)。

当选项带有参数时，传递参数时，下面两种方式是等价的：

```bash
-a=arg  <=> -a arg
--flag=arg <=> --flag arg
```

## bashly如何定义参数

bashly通过`args`指令定义参数，在`args`指令之下，支持下面这些指令：

- `name`：指定参数名称  
- `help`：参数说明
- `default`：指定默认的参数值，意味着该参数可选  
- `required`：该参数是否必须存在，设置为false(默认)表示可以省略  
- `allowed`：一个数组，该选项的参数的值必须是该数组中的一个，可以结合`default`指令或`required`指令  
- `repeatable`：该参数可以重复出现多次，各参数将被被引号包围并以空格分隔，因此应使用类似于`eval "datas=(${args[data]})"`的方式将这些参数提取出来并保存在另一个数组datas中。参考<https://bashly.dannyb.co/configuration/argument/#repeatable>  
- `unique`：该指令需结合`repeatable`指令同时使用，指定该指令时，将忽略多余的值相同的重复参数值
- `validate`：对参数进行验证。如何进行验证，参考后文[validate：验证参数](#validate)。

## filters指令：前置筛选条件

通过`filters`指令，可以指定一个或多个命令执行前的筛选函数：

- 这些筛选函数都是自己定义的  
- 如果任一筛选函数输出了任何内容，都会认为筛选失败，不会执行后续的命令  
- 只有所有筛选函数都不输出任何内容，才会在筛选完成之后继续执行命令的逻辑  

例如：

```yaml
name: viewer
filters:
- docker_running
```

这里指定的是`docker_running`，将会寻找`src/lib/filter_docker_running.sh`文件中的`filter_docker_running`函数并执行(在生成最终的脚本命令时会将该函数拷贝到脚本中)。

```bash
filter_docker_running() {
  docker info >/dev/null 2>&1 || echo "Docker must be running"
}
```

需要说明的是，如果是要判断环境变量是否设置，则更建议使用`environment_variables`指令，如果是要判断某个命令是否存在，则更建议使用`dependencise`指令。


<a name="validate"></a>

## validate指令：验证参数

如果要验证参数的值(可以是选项的参数，也可以是直接定义的参数)，可以使用`validate`指令。

通过`validate`指令，可以指定一个或多个参数验证函数：

- 这些验证函数有内置的，也可以是自己定义的  
- 如果任一验证函数输出了任何内容，都会认为验证失败从而报错  
- 只有所有验证函数都不输出任何内容，才会继续执行  

例如：

```yaml
name: viewer
args:
- name: path
  validate: file_exists
```

这里指定了验证函数`file_exists`，这将会寻找`src/lib/validate_file_exists.sh`文件中的`validate_file_exists`函数。

```bash
validate_file_exists() {
  [[ -f "$1" ]] || echo "must be an existing file"
}
```

bashly通过执行`bashly add validations`命令，可以自动获得一些内置的验证函数：

- `file_exists`：验证参数指定的是一个文件且存在  
- `dir_exists`：验证参数指定的是一个目录且存在  
- `integer`：验证参数是一个数值  
- `not_empty`：验证参数是非空的  

## 内置库函数和自定义库函数

在`src/lib`目录下的所有文件内容，都会合并到最终生成的脚本文件中。

因此，可以在`src/lib/xxx.sh`文件中定义一些函数，然后在编写代码逻辑的文件(如`root_command.sh`文件)中调用这些函数。

bashly通过执行下面命令也可以获得带颜色输出的内置库函数：

```bash
# 带颜色输出的库函数
$ bashly add colors
```

例如：

```bash
echo "before $(red this is red) after"
echo "before $(green_bold this is green_bold) after"
```

`colors`默认提供了下面这些输出方式：

```
red         red_bold      red_underlined
green       green_bold    green_underlined
yellow      yellow_bold   yellow_underlined
blue        blue_bold     blue_underlined
magenta     magenta_bold  magenta_underlined
cyan        cyan_bold     cyan_underlined
bold
underlined
```

<a name="bashly_hook"></a>

## bashly Hook

bashly允许用户编写钩子(Hook)使得一些逻辑能够在特定的时间点被执行。

通过`bashly add hooks`命令将生成3个hook文件，它们分别在三种时间点被执行：

- src/initialize.sh：该脚本文件中定义的内容全都会被写入initialize()函数中，这些代码将会在任何逻辑执行之前被执行，通常用于定义全局变量、全局函数、全局设置等操作  
- src/before.sh：该脚本文件中编写的代码将会在解析完命令行选项参数之后，且在命令行执行之前被执行  
- src/after.sh：该脚本文件中编写的代码将会在执行完命令行之后被执行

这些文件也可以手动创建，并且可以删除不需要的hook sh文件，比如可以只保留src/initialize.sh文件用来做全局初始化，可删除或不创建src/before.sh和src/after.sh文件。

initialize.sh通常用来做环境初始化，在调用任何函数之前会先执行这个文件中的命令，使得其它小文件中(包括src/lib目录下的库函数文件)都能访问这些全局变量。看bashly生成的最终脚本文件中执行逻辑：

```bash
# :command.initialize
initialize() {
  version="0.1.0"
  long_usage=''
  # 默认设置了set -e，意味着脚本中只要存在退出状态码非0的命令时就会退出整个脚本
  # 如果想要更改bashly的这种行为，参考下文"设置bashly的工作方式"的相关内容
  set -e

  # initialize.sh文件中的内容将被拷贝到此处
  # src/initialize.sh       
}

# :command.run
run() {
  ......
}

initialize
run "$@"
```

initialize.sh文件初始时是空的(只有一些注释行)。

```bash
## Code here runs inside the initialize() function
## Use it for anything that you need to run before any other function, like
## setting environment vairables:
## CONFIG_FILE=settings.ini
##
## Feel free to empty (but not delete) this file.
```

比如可以在initialize.sh中来设置脚本内部运行的全局变量。

```shell
# src/initialize.sh

MAIN_IP="192.168.200.100"
RUNTIME_DIR="/data"

# 通过 declare 定义全局变量时，记得使用-g选项
declare -g -A IPS
```

注意，bashly中如果想要通过declare定义全局数组(或全局变量)，一定记得加上`-g`选项。这是因为在bashly的任何有效文件中的自定义代码都会被bashly重新以函数的方式包裹写入最终生成的命令脚本文件中，比如initialize.sh中的代码将会被写入initialize()函数内。而在函数内部declare不使用`-g`选项时，它默认定义的是函数内的局部变量，这样src下的其它文件将无法访问该全局变量。

## 设置bashly自身工作方式

如果不做任何修改，bashly将以默认方式工作，但是这些工作方式可以修改。

通过`bashly add settings`命令将在根目录下生成settings.yml文件：

```
❯ tree
.
├── settings.yml
├── src
│   ├── bashly.yml
│   ├── initialize.sh
│   ├── lib
│   │   └── colors.sh
│   └── my_sub_command.sh
└── trade.sh
```

settings.yml文件中已经定义好默认的工作方式，可以修改这些配置项：

```yaml
# All settings are optional (with their default values provided below), and
# can also be set with an environment variable with the same name, capitalized
# and prefixed by `BASHLY_` - for example: BASHLY_SOURCE_DIR
#
# When setting environment variables, you can use:
# 某个配置项设置为 0 flase no 是等价的，都代表布尔 false
# 某个配置项设置为 1 true yes 是等价的，都代表布尔 true
# 某个配置项设置为 ~ 时表示设置为 null 或 nil
# - "0", "false" or "no" to represent false
# - "1", "true" or "yes" to represent true
#
# If you wish to change the path to this file, set the environment variable
# BASHLY_SETTINGS_PATH.

# 源代码文件所在目录
# The path containing the bashly source files
source_dir: src

# bashly.yml的路径
# The path to bashly.yml
config_path: "%{source_dir}/bashly.yml"

# 最终生成的命令行脚本脚本文件的目录
# The path to use for creating the bash script
target_dir: .

# 库函数lib的目录路径
# The path to use for common library files, relative to source_dir
lib_dir: lib

# bashly生成的sh文件所在路径，比如在何处创建/读取root_command.sh文件
# 设置为空`~`时，表示在source_dir下生成这些小文件
# The path to use for command files, relative to source_dir
# When set to nil (~), command files will be placed directly under source_dir
# When set to any other string, command files will be placed under this
# directory, and each command will get its own subdirectory
commands_dir: ~

# 默认bashly会设置set -e，意味着命令行脚本中遇到非0退出状态码时退出整个脚本
# 该配置项配置为`''`时，bashly将不会设置set -e
# 也可以设置其它值
# Configure the bash options that will be added to the initialize function:
# strict: true       Bash strict mode (set -euo pipefail)
# strict: false      Only exit on errors (set -e)
# strict: ''         Do not add any 'set' directive
# strict: <string>   Add any other custom 'set' directive
strict: false

# When true, the generated script will use tab indentation instead of spaces
# (every 2 leading spaces will be converted to a tab character)
tab_indent: false

# When true, the generated script will consider any argument in the form of
# `-abc` as if it is `-a -b -c`.
compact_short_flags: true

# Set to 'production' or 'development':
# env: production    Generate a smaller script, without file markers
# env: development   Generate with file markers
env: development

# The extension to use when reading/writing partial script snippets
partials_extension: sh

# Display various usage elements in color by providing the name of the color
# function. The value for each property is a name of a function that is
# available in your script, for example: `green` or `bold`.
# You can run `bashly add colors` to add a standard colors library.
# This option cannot be set via environment variables.
usage_colors:
  caption: ~
  command: ~
  arg: ~
  flag: ~
  environment_variable: ~
```


## bashly添加命令行自动补全和提示功能

bashly支持为最终生成的命令添加命令行的`TAB`键自动补全和提示功能，非常人性化。

设置也非常简单，只需执行`bashly add completions`命令，这将会生成`src/lib/send_comletions.sh`文件，bashly将命令行自动补全和提示的相关代码生成在此文件的`send_completions`函数中。以后只要修改了bashly.yml文件，只需再执行`bashly generate --upgrade`就会自动更新send_completions函数(也可以先执行bashly generate再执行bashly add completions)。

然后，在bashly.yml的各个你想要提供自动补全功能的子命令、参数下加上`completions`指令即可。例如：

```yml
name: trade.sh
help: Manage AutoTrade Processes and Some Utils
version: 0.1.0

# 在--help帮助信息的最下面Footer处提示用户如何开启自动补全功能
footer: |
  You can append the following line into ~/.bashrc and then re-login bash to set bash completions
      eval "\$(trade.sh completions)"

commands:
  # 额外提供一个completions子命令给用户，并将`eval "$(trade.sh completions)"`
  # 添加到.bashrc中实现每次重启bash就开启自动补全功能
  - name: completions
    help: create script completions
    # 将该子命令隐藏起来
    private: true

  - name: run
    alias: r
    help: Start/Kill/Restart/Show AutoTrade Process
    group: Process
    # 在需要开启自动补全的子命令上添加`completions`指令，
    # 按TAB键时将会提示命令自身和命令别名。
    # <...>内的字符会在tab键时被识别并尝试补全。
    # 开启了自动补全的命令下的长短选项会自动开启补全和提示。
    completions:
      - <run>
    commands:
      - name: kill
        help: Kill Process
        # completions指令下可以给多个补全提示
        completions:
          - <kill>
        args:
          - name: proc
            help: which process to kill
            required: false
            repeatable: true
            unique: true
            # 参数位上可以设置completions指令，但使用allowed时会自动将允许的参数列表开启自动补全和提示
            allowed: [ks,os,au,ct,all]
```

然后执行`bashly generate --upgrade`，bashly会自动将命令补全和提示相关的代码更新到`src/lib/send_completions.sh`文件中的send_completions函数，只需调用这个函数就可以在当前shell环境下开启自动补全功能。为了方便且每次登录bash后自动开启自动补全，在bashly.yml中额外添加了一个子命令`completions`，bashly会为该子命令生成一个completions_command.sh的文件，在该文件中调用send_completions函数：

```shell
## src/send_completions.sh

## 注释下面的行
## echo "# this file is located in 'src/completions_command.sh'"
## echo "# code for 'trade.sh completions' goes here"
## echo "# you can edit it freely and regenerate (it will not be overwritten)"
## inspect_args

# 添加这一行
send_completions
```

再执行`bashly generate --upgrade`重新生成一下最终的命令行文件。

最后，将`eval "$(trade.sh completions)"`添加到家目录的.bashrc文件中并重新登录bash，即可开启trade.sh的自动补全和提示功能。

## 使用bashly时需谨记在心的注意事项

- 所有自定义的代码都会被bashly重新包裹在bashly生成的shell函数中，因此要记得在使用declare定义全局变量时加上`-g`选项。  
- 能不使用here document就不要使用here document，而是改用多行echo的方式，bashly中使用here document的限制很大。
- 默认bashly会设置`set -e`，使得命令行脚本中只要遇到退出状态码非0时就退出，如果想要改变这种行为，应设置bashly的工作方式，参考前文相关内容。
- 除了初始化脚本(initialize.sh)中，其它脚本文件中尽量手动明确地定义局部变量，这是因为bashly最终生成的命令行脚本可能是由比较多的小文件组合而成的，如果定义的不是局部变量，很可能会修改或读取到其它小文件中定义的同名变量。要在小文件中定义局部变量，函数内可以使用local或declare，函数外可以使用declare。

## 较为复杂的命令行选项解析示例

下面是我个人实际使用的一个命令行选项解析的示例，由多个子命令组成，如果不打散为子命令，逻辑将非常复杂，分散为子命令让每个子命令的逻辑都变得非常简洁。

bashly.yml的文件内容如下：

```yml
name: get_ks
help: 解析和下载K数据, 并保存和转换
version: 0.1.0
commands: 
  - name: build
    alias: b
    help: 编译并将二进制拷贝到 /tmp/get_ks_apps 目录下
  - name: url
    alias: u
    help: 分析待下载的URL并输出至标准输出
    flags: 
      - long: --url-type
        short: -u
        arg: URL-TYPE
        validate: url_type
        conflicts: [--his-urls, --his-symbols]
        help: 指定url类型, 有效值 zip/api
      - long: --kline-type
        short: -k
        arg: KLINE-TYPE
        validate: kline_type
        conflicts: [--his-urls, --his-symbols]
        help: 指定K类型, 有效值 [1|5|15|30]m [1|2|4|6|8|12]h 1d
      - long: --force-from
        short: -f
        arg: Date
        validate: from_date
        conflicts: [--his-urls, --his-symbols]
        help: 强制从何处开始计算要下载的zip, --url-type的值必须是zip, 格式："2021-12-23"
      - long: --his-symbols
        short: -H
        conflicts: [--his-urls, --force-from, --kline-type, --url-type]
        help: 返回所有历史上的USDT交易对(包含曾经下架的交易对)
      - long: --his-urls
        short: -h
        arg: Year
        conflicts: [--his-symbols, --force-from, --kline-type, --url-type]
        validate: from_year
        help: 从哪一年开始下载历史数据, 有效值：2017及之后
    examples: 
      - (1).下载 1m 的 zip 数据
      - get_ks url --url-type zip --kline-type 1m
      - (2).下载 1h 的 api 数据
      - get_ks url --url-type api --kline-type 1h
      - (3).强制从2021-01-01开始下载 1h 的 zip 数据
      - get_ks url --url-type zip --kline-type 1h --force-from 2021-01-01
      - (4).输出包含已下架的所有USDT交易对
      - get_ks url --his-symbols
      - (5).下载包含已下架的所有USDT交易对从2021年开始的zip数据
      - get_ks url --his-urls 2021
  - name: download
    alias: d
    help: 根据给定的URL下载数据
    flags: 
      - long: --url-file
        short: -u
        arg: URL-FILE
        validate: file_exists
        help: 保存了urls的文件, 将从此文件读取待下载的url链接, 不指定时默认从标准输入中读取
      - long: --csv-dir
        short: -c
        arg: CSV-DIR
        default: /tmp/ks/csv
        help: 保存csv文件的目录路径, 默认 /tmp/ks/csv
      - long: --delete-csv
        short: -d
        help: 开始下载前, 是否先清空csv_dir
      - long: --zip-file
        short: -z
        arg: ZIP-FILE
        help: |
          指定该选项时, 将在下载完成后把所有csv文件归档压缩为zip文件, 
          该选项指定zip文件的路径(注意，该选项会在归档完成后删除csv_dir)
      - long: --jobs
        short: -j
        arg: NUM
        validate: check_jobs
        help: 并发下载的数量(值的范围：1-100), 默认值50
      - long: --no-bar
        short: -n
        help: 是否禁止输出进度条信息, 默认已开启进度条
  - name: store
    alias: s
    help: 将csv中的数据存储到数据库
    flags: 
      - long: --csv-dir
        short: -c
        arg: CSV-DIR
        default: /tmp/ks/csv
        validate: dir_exists
        help: 指定csv文件的存放目录
      - long: --delete
        short: -d
        help: 存储csv完成后, 是否删除csv文件
    examples: 
      - get_ks store --csv-dir /tmp/ks/csv --delete
  - name: convert
    alias: c
    help: 转换数据库中的K类型
    flags: 
      - long: --from
        short: -f
        arg: SRC-KLINE-TYPE
        allowed: [1m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d]
        help: 转换的源类型
      - long: --to
        short: -t
        arg: DEST-KLINE-TYPE
        allowed: [1m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d]
        help: 转换的目标类型
    footer: 注意：要求 to 的类型要能够从 from 进行转换, 例如 4h 不能转换为 8h
  - name: delete
    alias: D
    help: 删除或清空数据库中的数据
    flags: 
      - long: --symbols
        short: -s
        arg: SYMBOLS
        required: true
        help: |
          指定要删除哪些交易对中的数据, 多个交易对使用逗号分隔,
          不区分大小写, 可省略尾部USDT. 特殊值all表示所有交易对,
          格式参考: "btcusdt,ETHUSDT,doge"
      - long: --types
        short: -t
        arg: TYPES
        required: true
        allowed: [1m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d]
        help: |
          指定要删除的类型, 多个类型使用逗号分隔, 不区分大小写, 特殊值all表示删除所有类型的数据,
          格式参考：1m,5m,15m,30m,1h,1d
      - long: --from
        short: -f
        arg: FROM
        required: true
        validate: integer
        help: |
          从哪里开始删除数据(秒级Epoch), 特殊值0表示清空数据(truncate), 比删除所有数据更快
```

最终的文件和目录结构如下：

```
.
├── get_ks   # 最终生成的命令
└── src
    ├── bashly.yml
    ├── build_command.sh     # build子命令对应的脚本
    ├── convert_command.sh   # convert子命令对应的脚本
    ├── delete_command.sh    # delete子命令对应的脚本
    ├── download_command.sh  # download子命令对应的脚本
    ├── store_command.sh     # store子命令对应的脚本
    ├── url_command.sh       # url子命令对应的脚本
    ├── initialize.sh        # 用于初始化动作的脚本，其中定义了一些全局变量
    └── lib
        ├── check_apps.sh    # 我自己自定义的库函数
        ├── colors.sh        # bashly add colors添加的库函数
        ├── validate_check_jobs.sh  # 下面几个都是自定义的参数验证函数
        ├── validate_from_date.sh
        ├── validate_from_year.sh
        ├── validate_kline_type.sh
        ├── validate_url_type.sh
        └── validations     # bashly add validations添加的验证库函数
            ├── validate_dir_exists.sh
            ├── validate_file_exists.sh
            ├── validate_integer.sh
            └── validate_not_empty.sh
```

