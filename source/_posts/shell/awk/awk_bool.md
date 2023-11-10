---
title: 精通awk系列(17)：awk布尔值、比较和逻辑运算
p: shell/awk/awk_bool.md
date: 2020-02-27 10:37:35
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# awk布尔值

在awk中，没有像其它语言一样专门提供true、false这样的关键字。

但它的布尔值逻辑非常简单：  
- 数值0表示布尔假  
- 空字符串表示布尔假  
- 其余所有均为布尔真  
   - 字符串"0"也是真，因为它是字符串  
- awk中，正则匹配也有返回值，匹配成功则返回1，匹配失败则返回0  
- awk中，所有的布尔运算也有返回值，布尔真返回值1，布尔假返回值为0  

```
awk '
BEGIN{
    if(1){print "haha"}
    if("0"){print "hehe"}
    if(a=3){print "hoho"}  # if(3){print "hoho"}
    if(a==3){print "aoao"}
    if(/root/){print "heihei"}  # $0 ~ /root/
}'
```

# awk中比较操作

<a name="strnum"></a>

### strnum类型

awk最基本的数据类型只有string和number(gawk 4.2.0版本之后支持正则表达式类型)。但是，对于用户输入数据(例如从文件中读取的各个字段值)，它们理应属于string类型，但有时候它们看上去可能像是数值(例如`$2=37`)，而有时候有需要这些值是数值类型。

awk的数据来源：  
1. awk内部产生的，包括变量的赋值、表达式或函数的返回值。  
2. 从其它来源获取到的数据，都是外部数据，也是用户输入数据，这些数据理应全部都是string类型的数据。  

所以POSIX定义了一个名为"numeric string"的"墙头草"类型，gawk中则称为strnum类型。当获取到的用户数据看上去是数字时，那么它就是strnum类型。strnum类型在被使用时会被当作数值类型。

注意，strnum类型只针对于awk中除数值常量、字符串常量、表达式计算结果外的数据。例如从文件中读取的字段`$1`、`$2`、ARGV数组中的元素等等。

```
$ echo "30" | awk '{print typeof($0) " " typeof($1)}'
strnum strnum
$ echo "+30" | awk '{print typeof($1)}'
strnum
$ echo "30a" | awk '{print typeof($1)}'
string
$ echo "30 a" | awk '{print typeof($0) " " typeof($1)}'
string strnum
$ echo " +30 " | awk '{print typeof($0) " " typeof($1)}'
strnum strnum
```

### 大小比较操作

比较操作符：
```
< > <= >= != ==  大小、等值比较
in     数组成员测试
```

比较规则：

```
       |STRING NUMERIC STRNUM
-------|-----------------------
STRING |string string  string
NUMERIC|string numeric numeric
STRNUM |string numeric numeric
```

简单来说，string优先级最高，只要string类型参与比较，就都按照string的比较方式，所以可能会进行隐式的类型转换。

其它时候都采用num类型比较。

```
$ echo ' +3.14' | awk '{print typeof($0) " " typeof($1)}'  #strnum strnum
$ echo ' +3.14' | awk '{print($0 == " +3.14")}'    #1
$ echo ' +3.14' | awk '{print($0 == "+3.14")}'     #0
$ echo ' +3.14' | awk '{print($0 == "3.14")}'      #0
$ echo ' +3.14' | awk '{print($0 == 3.14)}'        #1
$ echo ' +3.14' | awk '{print($1 == 3.14)}'        #1
$ echo ' +3.14' | awk '{print($1 == " +3.14")}'    #0
$ echo ' +3.14' | awk '{print($1 == "+3.14")}'     #1
$ echo ' +3.14' | awk '{print($1 == "3.14")}'      #0 
$ echo 1e2 3|awk ’{print ($1<$2)?"true":"false"}’  #false
```

采用字符串比较时需注意，它是逐字符逐字符比较的。

```
"11" < "9"  # true
"ab" < 99   # false
```

## 逻辑运算

```
&&          逻辑与
||          逻辑或
!           逻辑取反

expr1 && expr2  # 如果expr1为假，则不用计算expr2
expr1 || expr2  # 如果expr1为真，则不用计算expr2

# 注：
# 1. && ||会短路运算
# 2. !优先级高于&&和||
#    所以`! expr1 && expr2`等价于`(! expr1) && expr2`
```

`!`可以将数据转换成数值的1或0，取决于数据是布尔真还是布尔假。`!!`可将数据转换成等价布尔值的1或0。

```
$ awk 'BEGIN{print(!99)}'   # 0
$ awk 'BEGIN{print(!"ab")}' # 0
$ awk 'BEGIN{print(!0)}'    # 1
$ awk 'BEGIN{print(!ab)}'   # 1，因为ab变量不存在

$ awk 'BEGIN{print(!!99)}'   # 1
$ awk 'BEGIN{print(!!"ab")}' # 1
$ awk 'BEGIN{print(!!0)}'    # 0
$ awk 'BEGIN{print(!!ab)}'   # 0
```

由于awk中的变量未赋值时默认初始化为空字符串或数值0，也就是布尔假。那么可以直接对一个未赋值的变量执行`!`操作。

下面是一个非常有意思的awk技巧，它通过多次`!`对一个flag取反来实现只输出指定范围内的行。

```
# a.txt
$1==1{flag=!flag;print;next}    # 在匹配ID=1的行时，flag=1
flag{print}               # 将输出ID=2,3,4,5的行
$1==5{flag=!flag;next}    # ID=5时，flag=0
```

借此，就可以让awk实现一个多行处理模式。例如，将指定范围内的数据保存到一个变量当中去。

```
$1==1{flag=!flag;next}
flag{multi_line=multi_line$0"\n"}
$1==5{flag=!flag;next}
END{printf multi_line}
```

# 运算符优先级

优先级从高到低：man awk

```
()
$      # $(2+2)
++ --
^ **
+ - !   # 一元运算符
* / %
+ -
space  # 这是字符连接操作 `12 " " 23`  `12 " " -23`
| |&
< > <= >= != ==   # 注意>即是大于号，也是print/printf的重定向符号
~ !~
in
&&
||
?:
= += -= *= /= %= ^=
```

对于相同优先级的运算符，通常都是从左开始运算，但下面2种例外，它们都从右向左运算：

- 赋值运算：如`= += -= *=`  
- 幂运算  

```
a - b + c  =>  (a - b) + c
a = b = c  =>  a =(b = c)
2**2**3    =>  2**(2**3)
```

再者，注意print和printf中出现的`>`符号，这时候它表示的是重定向符号，不能再出现优先级比它低的运算符，这时可以使用括号改变优先级。例如：

```
awk 'BEGIN{print "foo" > a < 3 ? 2 : 1)'   # 语法错误
awk 'BEGIN{print "foo" > (a < 3 ? 2 : 1)}' # 正确
```


