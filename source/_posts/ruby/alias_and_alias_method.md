---
title: Ruby alias和alias_method
p: ruby/alias_and_alias_method.md
date: 2021-05-27 17:46:16
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# alias和alias_method

`alias`关键和`alias_method`方法都可以用来给方法取别名。

```ruby
def m1
  xxx
end

# m1是旧方法，m2或m3是新方法
alias m2 m1
alias_method m3, m1

# 或使用Symbol的方式
alias :m2 :m1
alias_method :m3, :m1
```

但它们有区别：
1. `alias`是关键字，方法之间空格隔开，`alias_method`是方法，两方法作为参数，之间使用逗号隔开  
2. `alias`可用于任何上下文，而`alias_method`只能用在class/module关键字内部  
3. `alias`是词法作用域，定义在哪里就看到哪里的方法定义内容，`alias_method`是动态作用域，取决于运行时看到的方法定义内容  

使用建议：**更多时候建议使用alias_method，因为它定义的方法别名具有多态效应，比如子类重写旧方法后，调用新方法时可以调用到重写后的旧方法**。

例如，下面`alias_method`定义在class内部，而`alias`既可以在class/module内部，还可以在class外面甚至在对象内部。

```ruby
# alias_method定义在class或module内部
class User
  def full_name
    puts "junmajinlong"
  end
  alias_method :name, :full_name
end
User.new.name

# alias可以在class/module内部，也可以在外面
class User1
  def full_name
    puts "junmajinlong"
  end
  alias name full_name
end
User1.new.name

def full_name
  puts "junmajinlong"
end
alias name full_name
name
```

下面是`alias`和`alias_method`上下文的区别：

```ruby
# alias_method是动态作用域，运行时看到的full_name是什么就是什么
class User
  def full_name
    puts "junmajinlong"
  end
  def self.add_alias
    alias_method :name, :full_name
  end
end

class Teacher < User
  def full_name
    puts "teacher junmajinlong"
  end
  add_alias
end
Teacher.new.name   # teacher junmajinlong

# alias是词法作用域，定义时看到的full_name是什么就是什么
class User1
  def full_name
    puts "junmajinlong"
  end
  def self.add_alias
    alias name full_name
  end
end

class Teacher1 <User1
  def full_name
    puts "teacher junmajinlong"
  end
  add_alias
end

Teacher1.new.name   # "junmajinlong"
```

