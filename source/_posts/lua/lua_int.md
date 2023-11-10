---
title: Lua数值
p: lua/lua_int.md
date: 2020-05-02 10:40:38
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# 数值

在Lua 5.2以及之前的版本只有number，它们都以双精度浮点数的方式表示，它能表达的最大整数是2^53^。在Lua 5.3中，引入了两种类型的数值类型：  
1. integer，64位整型，支持的最大值为2^63^-1  
2. float，双精度浮点数  

因为Lua 5.2及之前版本中支持的2^53^为`9,007,199,254,740,992`，这是一个极大的整数值，所以Lua 5.2和Lua 5.3并不会出现太大的不兼容性。

```lua
type(3)           --> number
type(3.5)         --> number

--要区分类型时，使用math.type()
math.type(3)      --> integer
math.type(3.0)    --> float
math.type("3")    --> nil
```

# 算术运算

算术运算符有：
```
+   -   *   /   %   ^   //
```

1. Lua可识别带符号的负数(如`-3`)，不可识别带正号的正数(如`+3`)  
2. 两个整数之间的算术运算，结果仍然是整数，除法运算和幂运算除外  
3. 如果有一个或两个都是浮点数，则算术运算结果是浮点数，如果有整数，则整数先转换成浮点数  
4. 无论是整数除还是涉及浮点除，Lua为了保证两种除法结果一致，使得除法运算结果均为浮点数  
5. 为了让整数除法运算的结果也是整数，Lua引入了floor除法运算符号`//`  
6. 幂运算总是返回浮点数，且幂运算  
7. 取模运算`a % b`的符号总是由b的符号决定，且大小值由b决定。例如`x % (-3)`的结果一定是负数，且值只能为0、-1、-2  

例如：

```lua
3+2      --> 5
3+2.0    --> 5.0
3.0/2.0  --> 1.5
3/2      --> 1.5

3.0 // 2.0   --> 1.0
4   // 2.0   --> 2.0
4   // 2     --> 2
-9  // 2     --> -5

3 % -3   --> 0
4 % -3   --> -2
```

# tonumber()

tonumber(arg1,arg2)尝试将arg1转换成数值，如果能够转换则返回转换后的值，不能转换则返回nil。可使用arg2指定将arg1识别为多少进制的数。

```lua
tonumber(3.3)       --> 3.3
tonumber("3.3")     --> 3.3
tonumber("3a")      --> nil
tonumber("a3")      --> nil
tonumber("a3",16)   --> 163
tonumber("az",36)   --> 395
```

# math库

math库提供了一些数学运算的函数，包括各种三角函数、指数和对数函数，还包括：

```lua
pi
huge     --> 表示无穷大，可用于无限迭代
maxinteger(), mininteger()   --> 当前lua版本支持的最大和最小的整数值
type()
max(), min()
abs()
floor(), ceil()
sqrt()
random(), randomseed()
tointeger()    
```

`math.tointeger(A)`将A转换成整数，如果不能转换，则返回nil。只有表示整数值的数才能转换成整数。

```lua
math.tointeger(3)       --> 3
math.tointeger(3.0)     --> 3
math.tointeger("3.0")   --> 3

math.tointeger(3.3)     --> nil
math.tointeger("3.3")   --> nil
math.tointeger("3a")    --> nil
```

`math.random()`生成随机数：

```lua
math.random()      --> 返回0到1之间的随机小数值
math.random(5)     --> 返回1到5之间的随机整数值
math.random(5,10)  --> 返回5到10之间的随机整数值
```

`math.randomseed()`指定随机数种子，一般以os.time()作为种子值。

```shell
$ lua -e 'math.randomseed(os.time());print(math.random(1,10))'
```
