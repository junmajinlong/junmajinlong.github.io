---
title: Lua流程控制语句和作用域
p: lua/block_scope.md
date: 2020-05-02 10:40:41
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# 语句块的类型

1. if、while、for语句块
2. function语句块  
3. do...end语句块
4. repeat...until语句块  
5. 交互式Lua Shell中每一行可独立解释的是一个语句块  

```lua
--> 1.if...then...elseif...else...end
if a > 10 then
  print("xxx")
elseif a > 5 then
  print("yyy")
else
  print("zzz")
end

--> 2.while...do...end
while true do print("xxx") end

--> 3.for ... do ... end
for i = 1, 10 do print(i) end
for i = 10, 1, -1 do print(i) end

--> 4.for ... in ... do ... end
for k,v in pairs(a) do print(k,v) end

--> 5.function
function f(x,y)
  print(x,y)
end

--> 6.do...end
do
  print("hello")
end
```

# lua中的break

break跳出当前循环语句块(即只能在repeat、while、for中使用)。

由于语法构造的特殊性，Lua的`break`(以及退出函数的`return`)只能是一个块的最后一一条语句，即语句块的最后一条语句，或end/else/until前的一条语句。

例如，下面的break是end前的一条语句。

```lua
local i = 1
while a[i] do
  if i == 3 then break end
  i = i + 1
end
```

再例如:

```lua
if ... then
  ...
  break
else
  ...
  break
end

do break end
```

# lua中的continue

Lua中没有直接提供continue功能，但是Lua支持goto，通过goto可以间接实现continue。

```lua
i = 10
while i < 10 do
  if i % 2 == 0 then goto continue end
  print(i)
  
  ::continue::
end
```

# lua的作用域

local可声明局部变量。Lua中每个语句块都有自己的作用域环境，在这些语句块环境内使用local声明的变量只在对应语句块中生效，其它均为全局变量。

```lua
x  = 10         --> 全局变量x
local i = 1     --> 最外层的局部变量i
while i <= x do        --> 控制变量是上一层次的局部变量i
  local x = i * 2      --> 循环内的局部变量，每轮循环都重新定义
  print(x)
  aa = 10        --> 全局变量
  i = i + 1
end

if i > 20 then   --> i是上一层次的局部变量i
  local x        --> if中的局部变量
  x = 20 
  print(x+2)
  y = 10       --> 全局变量
else
  print(x)
end

print(x)        --> 输出10
```

需注意的是：  

1. for迭代时，其控制变量为局部变量  
2. 函数参数也是局部变量  

```lua
for i = 1, 10 do print(i) end
print(i)    --> nil

for k,v in ipairs({1,2,3,4}) do print(k) end --> 1,2,3,4
print(k,v)   --> nil     nil
```