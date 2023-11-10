---
title: Lua函数
p: lua/lua_function.md
date: 2020-05-02 10:40:42
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# 函数

Lua函数是一段可执行、可被调用的代码块，在Lua中**函数是first class，它可以作为值赋值给变量，作为值传递给参数，也可以作为值被函数返回**。

Lua定义函数：

```lua
function f(x,y)
  print(x,y)
end

--> f1和f2完全等价，在引用同一个函数
f1 = function(x,y) print(x,y) end
f2 = f1
```

调用函数时，加上括号并传递参数即可执行。

函数调用时，如果**参数只有一个，且这个参数是一个字符串字面量或者是table字面量(即table构造式)，则可省略括号**。

```lua
f "hello"     --> f("hello")
f {a=3,b=4}   --> f({a=3,b=4})
```

## 函数返回值

函数使用return来返回，return(和break一样)必须在语句块的结尾，每个函数在结尾都有一个隐含的`return nil`，且函数可返回多个值。

```lua
--> 查找序列中最大的值，同时返回该值的索引
function find_max(a)
  local max = a[1]
  local max_val_idx = 1
  for i,v in ipairs(a) do
    if v > max then
      max = v
      max_val_idx = i
    end
  end
  return max, max_val_idx
end

a = {3,10,1,12,5,6}
print(find_max(a))  --> 12    4
```

因为函数可以在不同环境下被调用，它的返回值会根据如下规则进行调整：  

1. 如果函数是单独的一条语句，则丢弃所有返回值  
2. 如果函数是表达式的一部分，则只保留函数第一个返回值  
3. 只有函数是多个表达式的最后一个元素(或唯一的元素)，才能获取函数的所有返回值。包括如下几种情况：  
  - 函数调用并多重赋值给变量时  
  - 函数调用并多重赋值给函数形参时  
  - 作为return的返回值部分时  
  - 作为table的构造语句时  
4. 使用小括号包围函数调用(如`(f())`)，可强制返回第一个返回值(所有要注意return语句中的函数调用不能加括号，否则只会返回单个值)  


例如：

```lua
function f() return "a", "b", "c" end

-->   变量多重赋值   <--
f()                --> 三个返回值被丢弃
a = f()            --> 后两个返回值被丢弃
a,b = f()          --> 最后一个返回值被丢弃
a,b,c = f()        --> 三个返回值都对应赋值
a,b,c,d = f(),10   --> a="b",b=10,c=nil,d=nil，只获取了f()第一个返回值

-->   作为函数参数   <--
print(f())          --> a b c
print(1,f())        --> 1 a b c
print(f(),1)        --> a 1
print(f().."x")     --> ax

-->   在table构造式中   <--
t = { f() }      --> t = {"a","b","c"}
t = { f(),"d" }  --> t = {"a","d"}

-->   在return中   <--
function ff() return f() end      --> return "a","b","c"
function ff() return 1,f() end    --> return  1, "a","b","c"
function ff() return f(),1 end    --> return "a", 1

-->   在小括号中   <--
print( ( f() ) )    --> a
function fff() return(f()) end   --> return中的函数调用不要加括号
```


## 函数参数

函数形参数量和实参数量可以不一致，会按照变量赋值一样的方式进行调整。

```lua
function f(x,y,z) print(x,y,z) end

f()           --> x = nil, y = nil, z = nil
f(1)          --> x = 1, y = nil, z = nil
f(1,2)        --> x = 1, y = 2, z = nil
f(1,2,3)      --> x = 1, y = 2, z = 3
f(1,2,3,4)    --> x = 1, y = 2, z = 3, 4被丢弃
```

如果需要在函数内部验证是否传递了某参数，或者为某参数提供默认值，可参考如下：

```lua
function f(x) 
  x = x or 0       --> 检查x参数，并设置默认值
  ...
end
```

如果有多个参数要传递，可以将参数收集在table中，然后传递table，或者使用table.unpack()解包table。

```lua
function f(x,y,z) return x, y, z end

f(1,2,3)

a = {1,2,3}
f(table.unpack(a))
```

## 变长参数

形参使用`...`可以接收剩下的所有参数：

```lua
function f(x,y,z,...)
  print(x,y,z)
  print(string.rep('-',20))
  for k,v in ipairs({...}) do
    print(k,v)
  end
end

f(1,2,3,4,5,6,7,8)    --> ... 接收了4,5,6,7,8
```

在非形参为位置，`...`表示的是一个表达式，正如上面`{...}`，类似于多返回值的函数一样，被构造成一个序列，该序列中包含了变长参数符号`...`接收到的所有实参。

```lua
function x(...) print(...) end
x(1,2,3)       --> 1 2 3

--> 接收变长参数，并直接返回参数
function x(...) return ... end

--> 类似Perl的参数处理机制
function f(...)
  local a,b,c = ...
  <code......code>
end
```

有时候有些参数可能会传递nil值作为其实参，但nil值会破坏对`...`的遍历，这时可使用`select()`。

使用`select(index, ...)`可以处理变长参数表达式`...`，当index指定为字符"#"时，它将返回`...`的总长度，即接收到的变长参数数量，如果index是一个整数值，则返会该整数值为索引的元素以及其后的所有元素，index可以为负数。

```lua
function f(...) return select("#",...) end
f(1,2,3,4)    --> 4， 返回接收到的总参数数量

function f(...) return select("2",...) end
f(1,2,3,4)    --> 2  3  4， 返回从index=2开始剩下的所有参数

function f(...) return select("-3",...) end
f(1,2,3,4)    --> 2  3  4， 返回从倒数第三个开始剩下的所有参数
f(1,nil,nil,4)     --> nil  nil  4， 可以处理nil参数

function f(...)
  for i = 1,select("#", ...) do 
    local arg = select(i, ...)
  end
end
```

使用`table.pack(...)`可以将变长参数打包成一个table，该table还包含了一个名为"n"的key，其值为打包的参数数量。

```lua
function f(...)
  local args = table.pack(...)
  print(args[1])
  print(args.n)
end
```

例如，定义一个函数，返回所有参数之和：

```lua
function add(...)
  local sum = 0
  local v = 0
  for i = 1, select("#",...) do
    v = select(i, ...)
    if type(v) ~= "number" then goto continue end
      sum = sum + v
    ::continue::
  end
  return sum
end
```

## 参数默认值

有些语言中允许在参数列表中定义函数的默认值，例如：

```lua
function f(name="long",age=23)
  ...
end
```

Lua不直接支持这种定义方式，但是Lua的函数调用有一个特性：当只有一个参数且参数为字符串字面量或table字面量时，可以省略括号。

所以，可以以table作为实参的方式写成如下类似的函数定义：

```lua
function f(arg)
  return arg.name,arg.age
end
```

然后调用时就可以使用如下方式调用：

```lua
f{name="junmajinlong",age=23}
```

## 尾调用消除

Lua原生支持**尾调用消除**(tail call elimination)，使得在递归的时候可以直接**尾递归**(tail recursive)。

例如，下面的哈数：

```lua
function f(x) 
  x = x + 1
  return g(x)
end
```

在上述示例中，函数f()内部调用了函数g()，且调用g()的时刻是f()的最后一个动作，当从g()执行完成返回到f()，f()不会做出任何事，而是直接退出回到调用f()的地方。

所以，对于f()内部调用g()来说，如果调用g()是f()的最后一个动作，那么调用g()之后其实无需保留f()的栈帧(因为即使保留了也没有正面作用)，可以直接让g()复用f()的栈帧，当g()执行完成后，将直接从g()返回到调用f()的地方。这就是尾调用消除。

但是要注意，调用的g()必须是f()中最后一个操作才算是尾调用，即只有像`return g()`一样，最后执行的是return且return中只有函数调用的操作。

例如，下面的示例中g()就不是最后一个操作，调用完g()后，还将等待g()执行完后返回f()进行一次加法操作。

```lua
function f(x) 
  x = x + 1
  return g(x)+1
end
```

尾调用消除主要用于尾递归，只要是满足尾调用的递归函数调用，无论递归多少次，都只占用常量的栈帧数量，不会出现栈溢出问题。

例如，不满足尾调用递归的阶乘计算方法：

```lua
function fact(n)
  if n == 1 then return 1 end
  return n * fact(n-1)
end
```

对于这个递归函数，假如执行的是fact(4)，它将在各层栈帧中维护如下数据(即保留状态)：

```
fact(4)
4 * f(3)
4 * 3 * f(2)
4 * 3 * 2 * f(1)
4 * 3 * 2 * 1
4 * 3 * 2
4 * 6
24
```

对上面的函数进行改装，将`n * fact(n-1)`这个操作想办法将乘积状态保存到函数调用中去：

```lua
function fact(n,m)
  m = m or 1      --> 用户调用时不会传递m参数，所以设置其为1
  if n == 1 then return m end
  m = m * n
  return fact(n-1,m)
end
```

为了测试，把上面的阶乘计算改成计算给定数的和。

```lua
-->  不使用尾调用   <--
function sum1(n)
  if n == 1 then return 1 end
  return n + sum1(n-1)
end

-->  使用尾调用消除   <--
function sum2(n,m)
  m = m or 1
  if n == 1 then return m end
  m = m + n 
  return sum2(n-1,m)
end

sum2(1000000)      --> 500000500000
sum1(1000000)      --> 栈溢出
--[[
stdin:3: stack overflow
stack traceback:
        stdin:3: in function 'sum1'
        ...
        stdin:3: in function 'sum1'
        stdin:3: in function 'sum1'
        (...tail calls...)
        [C]: in ?
]]
```

并非所有的递归函数都能改装成尾递归调用，也并非所有的语言都原生支持尾调用消除，有些语言可能需要添加额外的编译参数才能打开默认被禁用的尾调用消除功能。特别地，尾调用功能增加了基于栈帧的调试难度。

## 将函数保存在table中

Lua中函数也是一个值，它可以保存在table中。

例如：

```lua
List = {}
List.push = function (l,v) table.insert(l,v) end
List.pop = function (l) return table.remove(l) end

-->  等价定义形式  <--
List = {
  push = function (l,v) table.insert(l,v) end,
  pop = function (l) return table.remove(l) end
}

-->  Lua特意提供的更有意义的等价定义形式  <--
List = {}
function List.push(l,v) table.insert(l,v) end
function List.pop (l) return table.remove(l) end
```

如此定义后，就可以通过List.FuncName来调用对应的函数：

```lua
List.push(List,"a")
List.push(List,"b")
List.push(List,"c")
List.pop(List)     --> c
List.pop(List)     --> b
List.pop(List)     --> a
```

## 局部函数

Lua中函数可以赋值给一个变量，而变量可以是全局变量，也可以是局部变量。

如果将函数赋值给一个局部变量，它将成为局部函数，按照作用域内的局部变量可见性，局部函数将只能在对应作用域内可见。

```lua
local f = function(x,y) return x+y end     --> (1)
local function f(x,y) return x + y end     --> (2)
```

这两种方式并不完全等价。定义方式(1)是先定义函数，然后赋值给局部变量f，而定义方式(2)是先定义局部变量f，然后定义函数，再将函数赋值给局部变量f。即(2)等价于：

```lua
local f
f = function (x,y) return x + y end
```

要注意定义方式(1)的局部函数在递归调用时可能出现的错误：

```lua
local fact = function (n)
  if n == 0 then return 1 end
  return n * fact(n-1)       --> 有问题
end
```

因为在编译函数体内的fact(n-1)时，局部变量fact还未定义(总是先评估等号右侧，然后才赋值)。

所以，可改为定义方式(2)的局部函数，或者等价的如下方式定义：

```lua
-->   local function f    <--
local function fact(n) 
  if n == 0 return 1 end
  return n * fact(n-1)
end

-->   与之等价的   <--
local fact
fact = function(n) 
  if n == 0 then return 1 end
  return n * fact(n-1)
end
```