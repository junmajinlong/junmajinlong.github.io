---
title: Ruby带颜色输出
p: ruby/ruby_colorize.md
date: 2020-07-26 11:37:29
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby带颜色输出

```shell
$ gem install colorize
$ gem install win32console  # for Windows
```
注：有些颜色效果不一定能在所有终端中都生效。

直接在字符串上设置颜色：

```ruby
require 'colorize'

# 输出蓝色
puts "Blue text".blue

# 部分字符带颜色
puts "This is" + "fancy schmancy".red + "text"
puts "This is #{"fancy schmancy".red} text"

# 模式：加粗
puts 'Hi in ' + 'bold!'.bold

# 红色字体、白色背景、加粗
puts 'hello world'.red.on_white.bold
```
或者使用colorize()方法设置颜色：
```ruby
puts "hello world".colorize(:blue)
puts "hello world".colorize(color: :blue)
puts "hello world".colorize(color: :blue, background: :red)
puts "hello world".colorize(color: :blue).colorize(background: :red)
```
如果不想让colorize直接为字符串添加方法，可以使用封装过的`colorized_string`。
```ruby
require 'colorized_string'
puts ColorizedString["hello world"].blue
puts ColorizedString["hello world"].colorize(:blue)
```

使用如下方法可查看支持的颜色和模式:

```ruby
puts String.colors     # 支持的颜色，返回数组
puts String.modes      # 支持模式，返回数组
=begin
  bold:       粗体
  italic:     斜体
  underline:  下划线
  blink:      闪烁
  swap:       用颜色包围，类似于背景色
  hide:       隐藏不显示
=end
puts String.color_samples  # 查看颜色效果

# 对应的还有
ColorizedString.colors
ColorizedString.modes
ColorizedString.color_samples   
```
