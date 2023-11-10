---
title: Lua变量、数据类型、布尔运算
p: lua/var_datatype_bool.md
date: 2020-05-02 10:40:37
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------


# 8种基本类型

Lua中有8种基本数据类型：

```
number,   string, nil,   boolean
function, thread, table, userdata
```

其中userdata需要使用C或其它宿主语言来定义。

在Lua 5.2以及之前的版本只有number，它们都以双精度浮点数的方式表示，它能表达的最大整数是2^53^。在Lua 5.3中，引入了两种类型的数值类型：  
1. integer，64位整型  
2. float，双精度浮点数  

使用type可以查看数据类型：

```lua
type(3)                --> number
type("hello")          --> string
type(function () end)  --> function
type(false)            --> boolean
type(nil)              --> nil
type(abc)              --> nil，因为abc是不存在的变量，默认返回为nil
```

# 变量

1. Lua是动态语言，变量无需声明便可使用，未赋值的变量其返回值为nil  
2. Lua中值为nil的变量，默认情况下完全等同于未定义变量  
3. 当赋值nil给某变量时，可能会使得该变量的原值被GC  
4. Lua变量赋值时，总是先计算等号右边的值，计算完成后才开始赋值操作  
5. Lua支持多重赋值：  

   ```lua
   a = 1
   a, b = 1, 2
   a, b, c = 1, 2       -- c = nil
   a, b, c = 1, 2, 3, 4 -- 4被忽略
   a, b, _ = 1, 2, 3    -- 3被赋值给哑变量，即认为3这个值是不要的
   a, b = b, a          -- 变量交换
   
   a = nil              -- a原来的值1将等待被GC
   ```

6. Lua中只有全局变量和局部变量的概念，局部变量使用local关键字来声明，此外某些语句块(比如while/for)的控制变量也是局部变量  
7. Lua中的变量是变量名指向数据值，例如`a,b=123,"hello"`，是`a --> 123, b --> "hello"`  

# 布尔值

Lua中，只有nil和false两个值是False，其它所有值都是True，包括空字符串""、数值0，均为true。

Lua支持`not and or`三种布尔运算符(没有对应的` ! && ||`)，其中not优先级较高，而and和or优先级几乎是所有运算符中最低的。

`not`总是返回布尔值：

```lua
not ""      -- false
not 0       -- false
not nil     -- true
not false   -- true
```

`and`和`or`将短路运算，且返回能做出布尔运算结论时的值：

```lua
4 and 5        --> 5 
nil and 13     --> nil 
false and 13   --> false 
0 or 5         --> 0 
false or "hi"  -->"hi"
nil or false   --> false
```

此外，`a and b or c`绝大多数时候等价于三目运算`a ? b : c`，但前提是b不能为false。当b为true时：  
- 如果a为真，b为真，则a and b返回b，b or c又返回b  
- 如果a为假，则a and b返回a，a or c返回c  

当b为false时，如果a为true，则a and b返回b，b or c无法保证返回b，所以无法等价于`a ? b : c`。

# 大小比较

Lua中的比较运算符有：

```lua
< > <= >= == ~=    --> ~=是不等于比较
```

对于`< <= > >=`来说，只允许相同的两种数据类型进行比较，且默认只支持数值类型的比较和字符串类型的比较。

```lua
 3   >  2      --> true
"a"  < "c"    --> true
 3   < "a"    --> Error
```

对于`== ~=`来说，支持不同类型的比较，当比较非数值和非字符串时，它们比较的是引用地址，即面向对象中的"是否是同一个对象"。

```lua
3 == 3.0         --> true
"a" == "a"       --> true
{1,2} == {1,2}   --> false
```

注意，nil参与比较时，它只和nil自身相等。

```lua
nil == nil    --> true
nil == 3      --> false
nil ~= 3      --> true
nil == false  --> false
```
