---
title: Ruby异常处理
p: ruby/ruby_exception_handle.md
date: 2021-05-09 22:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# Ruby异常处理

## 抛出错误

Kernel模块中定义了raise方法，用来抛出异常：

```ruby
raise
raise 'wrong'
raise RuntimeError 'wrong'
```

raise抛出异常后，如果不对其进行处理，将会终止线程或进程(如果设置了Thread.abort_on_exception，则线程内raise时也会终止主线程，否则，只会终止该线程)。

Ruby中处理异常的方式是使用begin...rescue...end语句块。

## 异常处理

如果某些代码可能会抛出异常，并且想要处理这些异常，则可以使用begin...rescue...end语句块。

```ruby
begin
  # 可能抛出异常的语句
rescue
  # 处理异常
end
```

当begin...rescue中间的代码抛出异常时，rescue将可以捕获抛出的异常并立即进入rescue...end部分的代码部分。

此外，方法定义语句、语句块、class定义语句、module定义语句也都能使用rescue来捕获异常并处理：

```ruby
# 方法定义语句使用rescue
def my_method
  # ...
rescue
  # ...
end

# 语句块使用rescue
[0, 1, 2].map do |i|
  10 / i
rescue ZeroDivisionError
  nil
end

# class定义语句使用rescue
class A
  ...
rescue
  ...
end

# module定义语句使用rescue
module M
  ...
rescue
  ...
end
```

默认情况下，rescue捕获StandardError异常，但可以明确指定要捕获哪种或哪几种类型的异常：

```ruby
begin
  # ...
rescue ArgumentError, NameError
  # 抛出ArgumentError或NameError异常时都在此处理
end

begin
  # ...
rescue ArgumentError
  # 处理ArgumentError
rescue NameError
  # 处理NameError
rescue
  # 处理其他的StandardError
end
```

可以在rescue子句的最后将抛出的异常赋值给局部变量，该局部变量只在对应的异常分支有效：

```ruby
begin
  # ...
rescue ArgumentError, NameError => e
  # 可以使用e变量
rescue => e
  # 可以使用e变量
end
```

如果想要让异常冒泡到调用者，可以在rescue分支中处理异常时再使用raise抛出一次错误：

```ruby
begin
  # ...
rescue
  raise # re-raise the current exception
end
```

## retry、else和ensure

rescue语句块内部可以使用retry关键字(retry能且只能在rescue中使用)，retry的效果是重新回到begin语句块的开头再次执行：

```ruby
begin  # (1)
  ...
rescue
  retry  # 回到(1)
end
```

可以使用ensure语句块，该语句块的作用是：无论是否抛出异常，以及异常是否被处理，都执行这部分语句块中的代码。ensure不仅可以用在begin...end中，还可以用在其他任何rescue可以使用的地方。例如方法定义时也可以使用ensure表明无论是否抛出异常以及是否处理异常，都执行这部分代码。

```ruby
begin
  ...
rescue
  ...
ensure
  ...
end

def f
  ...
ensure 
  ...
end
```

begin...rescue...end语句块中还可以使用else子句(注：else只能结合rescue使用，否则会给出警告信息提示单独使用else没有任何意义)，else子句中的代码只在未抛出异常的情况下被处理：

```ruby
begin
  ...
rescue
  ...
else
  ...
end
```