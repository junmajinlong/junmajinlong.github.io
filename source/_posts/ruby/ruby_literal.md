---
title: Ruby字面量、引号、%符号和heredoc相关
p: ruby/ruby_literal.md
date: 2020-05-11 09:37:30
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# 字面量、引号、%符号和heredoc

## 数值字面量

唯一需要说明的是分数字面量：数值后加上一个后缀字母`r`表示分数字面量。

```ruby
# 整数字面量
0
1
100
10_000_001   # 千分位

# 浮点数字面量
0.1
1.0
1.2222

# 分数字面量
1r                  # 等价于(1/1)
2r                  # 等价于(2/1)
0.3r                # 等价于(3/10)
0.4r-0.1r           # 等于(2/5) - (1/10) == (3/10)
0.4r-0.1r == 0.3r   # true
```


## 引号

引号和Perl中的引号类似。

例如，单引号不解释变量内插和反斜线序列等，双引号解释变量内插和反斜线序列等，反引号用于执行对应的Shell命令。此外，反引号中可以进行变量内插，也就是说反引号中的字符会按照双引号进行解释，例如`` a=haha;str=`echo #{a} one line` ``得到的结果为`str=haha one line`。

此外，Ruby还支持三引号，包括三个双引号、三个单引号，用来实现简单的heredoc功能。双引号可以进行变量内插，单引号不可变量内插：

```ruby
one=1
two=2
three=3

puts """
line #{one}
line #{two}
line #{three}
"""

puts '''
line #{one}
line #{two}
line #{three}
'''
```

## %符号

**`%`用于简化常量的定义**。

下面的中括号边界符可以替换为其它对称符号或相同的符号。例如`()`、`<>`、`||`、`##`。

![](/img/ruby/1589464731358.png)

注意，对于`%x[]`来说：

- shell命令的输出结果需保存到变量，否则结果被丢弃。例如`a=%x(echo hahaha)`表示将echo的输出结果保存到变量a，而`%x`执行过程中并不输出数据
- 它和Ruby中的反引号是一样的。它们都是调用Kernel模块中的`` ` ``方法，所以，也可直接调用该方法执行一段shell命令：``Kernel.`(echo hahaha)``

## heredoc

`heredoc`和Perl的heredoc基本一样。

- `<<END`：等价于双引号的heredoc
- `<<"END"`：按双引号去解释heredoc内容，所以会解释内插表达式和反斜线序列
- `<<'END'`：按单引号去解释heredoc内容，所以不会解释内插表达式和反斜线序列
- ``<<`END` ``：使用shell去执行heredoc的内容

```ruby
str1 = <<END
one line
END
# str1="one line"

a="one line"
str2 = <<"END"
#{a}
END
# str2="one line"

a="one line"
str3 = <<'END'
#{a}
END
# str3="#{a}"

a="one line"
str4 = <<`END`
echo haha #{a}
END
# str4="haha one line"
```

**heredoc的起始符前可以加上一个短横线【-】，使得终止符可以缩进编写**。例如：

```ruby
str1 = <<-ONE
one line
  ONE          # 终止符ONE缩进了

str2 = <<-"TWO"
two\nline      # 换行符被解释
  TWO
```

**heredoc的起始符前可以使用`~`，使得正文和结束符都能缩进，返回的结果从第一个非空白符号开始缩进**。例如：

```ruby
puts <<~END
  Usage: abc def  # 结果中，这行没缩进
    one line      # 结果中这两行缩进了两个空格
    two line
  END

puts <<~END
  one line    # 结果中，这两行没缩进
  two line
END
```

**heredoc的起始行中起始符后面的内容不会解释为heredoc的内容**。所以：

```ruby
arr = [<<END, "two", "three"]  # END后面的不会解释
one line
END

# arr = ["one line", "two", "three"]
```

事实上，当Ruby解释器遇到`<<`的时候就**立即跳转到它下面的行中读取数据直到遇到结尾符号，然后重新回到here doc的开头行继续向后读取**，所以上面`<<END`和`END`中间的行先被读取，读取完END之后回头读two、three字符串。

**因此heredoc可以堆叠多个doc栈**。例如：

```ruby
arr = [<<ONE, <<TWO, <<THREE]
one line
ONE
two line
TWO
three line
THREE

# arr=["one line", "two line", "three line"]
```

在解析到`<<ONE`时，会先读取直到ONE终止符中间的内容，读完后这部分内容就消失了，再回头处理`<<TWO`并读取直到TWO终止符中间的内容，同理`<<THREE`。

