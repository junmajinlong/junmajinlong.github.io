---
title: Ruby获取用户输入的密码
p: ruby/ruby_getpass.md
date: 2020-07-21 12:35:34
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby获取用户输入的密码

`io/console`提供了getpass方法用于获取用户输入的密码，用户输入的密码字符不会显示在终端上。

```ruby
require 'io/console'

# The prompt is optional
password = IO::console.getpass "Enter Password: "
puts "Your password was #{password}"
```

或者，也可以使用`io/console`提供的noecho()方法，它获取用户非回显输入的字符。用于输入密码时，需要去除尾部换行符以及可能的空白符号。
```ruby
require 'io/console'

# Another option using $stdin.noecho()
$stdout.print "Enter password: "
# STDIN.noecho{|io| io.gets}
password = $stdin.noecho(&:gets)
password.strip!
puts "\nYour password was #{password}"
```
