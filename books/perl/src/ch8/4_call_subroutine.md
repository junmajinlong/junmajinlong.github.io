## 调用子程序的几种方式

注：本节包含了原型子程序相关内容，关于原型，将在本章后面的小节中详细介绍。

在Perl中，调用子程序有几种方式。以调用subname子程序为例：
- (1).`&subname([args])`  
- (2).`subname([args])`  
- (3).`subname [args]`和无参数时的`subname`  
- (4).`&subname`  

从上面几种调用方式来看，容易发现不允许使用`&subname args`调用方式，即`&`前缀不能和没有小括号的参数传递一起使用。

必须得区分清楚这几种调用方式的不同之处：  

- 建议的调用子程序方式是(2)，即使用小括号调用方式  
- 如果子程序已经定义好了(即子程序先定义后调用)，则可以省略小括号，即(3)  
- 使用小括号调用子程序时，sigil前缀`&`是可选的，即(1)和(2)效果差不多：  
  - 对于一般的子程序来说，(1)和(2)等价  
  - 对于原型子程序来说，(1)和(2)不等价  
- 使用了sigil前缀`&`，有两层效果：  
  - 当未手动传递参数时，将调用子程序环境的`@_`传递给子程序，即(4)。例如，如果在子程序a中`&b`调用子程序b，调用子程序`a(1,2,3)`，则参数1、2、3也将传递给子程序b(可继续参考稍后的示例来理解)  
  - 忽略编译器的原型检测过程  

例如，下面两个子程序，without_prototype不带原型，with_prototype带有原型：

```perl
# (1)
sub without_prototype{ ... }
sub with_prototype(){ ... }
# (2)
```

在位置(1)处，可通过如下几种方式分别调用without_prototype和with_prototype：

```perl
&without_prototype;
without_prototype();
without_prototype(1,2,3); # 带参数调用
&without_prototype();
&without_prototype(1,2,3); # 带参数调用

# 原型子程序会先检测原型，因此应当先定义再调用
&with_prototype;    # 跳过原型检测
with_prototype();        # 不建议的调用方式，会给出警告
with_prototype(1,2,3);   # 不建议的调用方式，会给出警告
&with_prototype();  # 跳过原型检测
&with_prototype(1,2,3);
```

在位置(2)处，可通过如下几种方式分别调用without_prototype和with_prototype：

```perl
without_prototype;
without_prototype 1,2,3;
without_prototype(1,2,3);
&without_prototype;
&without_prototype(1,2,3);

# 上面的原型子程序要求了不能传递参数，
# 所以调用子程序时不能传递参数
with_prototype;     
&with_prototype;    # 会跳过原型检测
with_prototype();
&with_prototype(2); # &跳过了原型检测，因此可传递参数
```

此外，还需理解`&`调用子程序时，如果不手动传递参数，则会自动将所在环境的`@_`作为参数传递给子程序。例如：

```perl
sub first{
  &second;   # &subname不手动传递参数
}

sub second{
  say "@_";
}

first(1,2,3);  # 调用first并传递参数
```

上面示例中，调用first时传递了参数1、2、3，由于first子程序内部使用`&second`而非`&second(ARGS)`方式调用子程序second，由于未手动为second传递参数，因此在调用second时，perl会自动将调用second时所在环境(即first子程序内)的`@_`作为参数传递给second子程序。因此，first内部的`&second`等价于`&second(@_)`。

最后，作为总结，建议第一选择是使用小括号调用方式`subname([args])`，如果子程序已经定义好，则可以考虑不使用小括号的调用方式`subname [args]`，它们都不会跳过原型检测。如果是调用自己编写的而非从其他模块导入的函数，则可以考虑使用`&`前缀调用方式。或者，如果明确知道`&`前缀调用子程序的副作用，也可以考虑使用`&`前缀相关的几种调用方式。

### 子程序引用的调用方式

由于Perl还支持子程序引用，因此，除了上述几种调用子程序的方式，还有几种调用子程序引用的方式。

假如`$subref`是一个子程序的引用，有以下几种调用方式：  

- `&$subref([args])`或者完整格式的`&{$subref([args])}`  
- `$subref->([args])`  
- `&$subref`  

这里的`&`前缀仍然具有两层效果：  

- 跳过编译器的原型检测  
- 如果未手动传递参数，则将当前环境的`@_`作为参数传递给子程序  
