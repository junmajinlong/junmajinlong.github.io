## 子程序的返回值细节

Perl子程序总是有返回值，且因为存在上下文的原因，子程序的返回值和其他语言有些不同，有必要去了解一下相关的细节。

Perl中设置返回值的方式有两种：使用return和无return。其中：  

- return的参数(即要指定的返回值)是一个列表上下文  
- 无return时将以最后被执行的语句计算结果作为返回值  

### 无参数的return

如果不指定return的参数，它也有指定返回值，此时return自身将根据调用子程序的上下文来决定返回何值：  

- 在列表上下文，无参数的return返回空列表  
- 在标量上下文，无参数的return返回未定义值(可当作undef或0或空字符串使用)  
- 在空上下文，无参数的return不返回任何数据，只用于终止子程序  

```perl
sub roll{
  print 1+int(rand(6));
  return;
}
```

### 无return的子程序返回值

在Perl中，即使不使用return指定返回值，子程序也会有返回值，此时子程序的返回值为最后一条被执行语句的计算结果。

下面是两个等价的子程序定义方式：

```perl
sub roll{
  1+int(rand(6));
}

sub roll(){
  return 1+int(rand(6));
}
```

这里要注意区分最后一条被执行语句和最后一条语句，最后一条被执行的语句不一定是子程序的最后一条语句。例如：

```perl
sub roll{
  if(COND){
  ## 分支1
    2;
  } else {
  ## 分支2
    3;
  }
}
```

如果COND条件为真，则最后一条被执行的语句分支1中的2，2直接被返回。如果COND条件为假，则最后一条被执行的语句是分支2中的3，3直接被返回。

由于子程序中不使用return关键字时也有返回值，有时候要注意它可能带来的副作用。

例如，下面的子程序也有返回值：

```perl
sub roll{
  print 1 + int(rand(6));
}
```

上述子程序中没有使用return，因此最后一条被执行语句的计算结果(或它的返回值)是该子程序的返回值。最后一条被执行的语句是print语句，print函数的返回值为1，因此roll()的返回值为1。

```perl
say roll();  # 1
```

### 返回单值和列表

使用return返回时，可直接返回标量，也可以返回多个数据组成的列表：

```perl
# 返回单个值
return 1
return 0
return ""

# 返回多值组成的列表
return 1,2,3;
return (1,2,3);
```

### 返回数组和hash

Perl中的return无法直接返回数组和hash，因为它们会先转换成列表，再以列表的方式返回。

如果确实要返回数组或列表，应当返回它们的引用。

```perl
return $array_ref;
return $hash_ref;
```

也可以返回匿名数组或匿名hash：

```perl
return [qw(a b c)];   # 返回匿名数组
return {(name=>"junmajinlong", age=>23,)};   # 返回匿名hash
```

### 不同上下文的子程序return

使用return返回时，将根据所处上下文的不同进行对应的转换。

例如，对于return返回列表的子程序，当它处于一个标量上下文中时，返回的列表将转换为标量数据，即列表的最后一个元素(再次提醒，要区分标量上下文中列表和数组转换为标量的区别：列表取最后一个元素，数组取数组长度)。

如果想要根据上下文的不同决定不同的返回值，可使用`wantarray`关键字。wantarray用来检测子程序在何种上下文环境：

- wantarray在列表上下文返回true  
- wantarray在标量上下文返回false  
- wantarray在空上下文返回未定义数据  

wantarray典型的用法大致如下：

```perl
# 如果在空上下文，则直接终止，不返回数据
return unless defined wantarray;
# 如果在列表上下文，返回数组数据(会转换为列表)，
# 在标量上下文，返回标量数据
return wantarray ? @array : $scalar;
```

实际上，wantarray更应该命名为wantlist，它期待的是列表上下文，而不是期待数组。在`perldoc -f wantarray`中也说明了，wantarray在未来的版本中可能会更换为wantlist。
