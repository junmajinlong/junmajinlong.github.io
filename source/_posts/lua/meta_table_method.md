---
title: Lua元表和元方法
p: lua/meta_table_method.md
date: 2020-05-02 10:40:59
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# 元表基本概念

元表(meta table)可以修改一个值在面对一个未知操作时的行为。

比如数值和字符串相加时默认是报错的，可以通过元表修改这种默认行为，比如table和table是可以相加的，这个相加操作是通过元表定义的。

Lua中每一个值都可以定义它的元表，但只通过Lua语言本身，只能为table设置元表，如果要为其它类型定义元表，需使用C代码完成。

默认情况下：  

- 新创建出来的table是没有元表的  
- 新创建出来的字符串，均使用同一个字符串标准库设置的预定义元表  
- 其它类型的数据在新创建出来时均没有元表  

可以使用`getmetatable(t)`获取t表的元表，使用`setmetatable(t,t1)`设置t1表为t的元表。

例如：

```lua
t = {}
getmetatable(t)         --> nil

t1 = {}
setmetatable(t,t1)
getmetatable(t)         --> table: 0x7fffe62af4e0

getmetatable("hello")    --> table: 0x7fffe6288d60
getmetatable("world")    --> table: 0x7fffe6288d60
getmetatable(true)       --> nil
getmetatable(10)         --> nil
```

一个表可以作为任意值的元表，多个表也可以共享同一个元表来描述它们之间具有的共同行为，某表还可以成为它自己的元表来描述其自身的行为。

# 元方法查找机制

以`a + b`运算为例。按照如下顺序进行查找：  
- 查找a是否有元表，元表中是否有`__add`这个元方法，如果有则调用该方法进行加法运算  
- 查找b是否有元表，元表中是否有`__add`这个元方法，如果有则调用该方法进行加法运算  
- 两者均无`__add`元方法，所以报错  

通常，不相同的数据类型不允许运算，比如type1 + type2从理论上来说是不允许的，这时应在元方法(比如`__add`)中加入判断机制：如果它们的元表相同，说明具有共同行为，属于同类数据，允许运算，否则报错。

参见下方示例。

# 算术运算相关元方法

在元表中加入如下方法，可获得对应的算术操作符的运算能力。

```lua
__add       -->   +
__sub       -->   -
__mul       -->   *
__div       -->   /
__idiv      -->   //
__unm       -->   -   负数
__mod       -->   %
__pow       -->   ^   幂运算
```


例如，通过序列定义一个集合，并定义集合的并集运算符`+`和交集运算符`*`。

```lua
local mt = {}

local Set = {}

function Set.new(s)
  local res = {}
  setmetatable(s, mt)
  for i,v in ipairs(s) do
    res[v] = true
  end
  return res
end

function Set.union(a,b)
  if getmetatable(a) ~= mt or getmetatable(b) ~= mt then 
    error("attempt to 'add' a set with a non-set value")
  end
  local res = Set.new{}

  for k in pairs(a) do res[k] = true end
  for k in pairs(b) do res[k] = true end
  return res
end

function intersection(a,b)
  if getmetatable(a) ~= mt or getmetatable(b) ~= mt then 
    error("attempt to 'add' a set with a non-set value")
  end
  local res = Set.new{}
  for k in pairs(a) do
    res[k] = b[k]
  end

  return res
end

function Set.tostring(s)
  local t = {}
  for k in pairs(s) do
    t[#t+1] = k
  end
  return "{"..table.concat( t, ", " ).."}"
end


mt.__add = Set.union
mt.__mul = Set.intersection

return Set
```

# 关系运算元方法

可以定义关系运算元方法：

```lua
__eq   -->    ==
__lt   -->    <
__le   -->    <=
```

其它的关系运算没有对应的元方法，因为它们都可以通过关系转换得到。例如`~=`等价于`not (a==b)`。

比如，集合的`a <= b`表示子集，`a < b`表示真子集，`a == b`表示相同集合。

```lua
function mt.__le(a,b) 
  for k in pairs(a) do
    if not b[k] then return false end
  end
  return true
end

function mt.__lt( a,b )
  return a <= b and not (b <= a)
end

function mt.__eq( a, b )
  return a <= b and b <= a
end
```

# 其它元方法

```lua
__concat      -->   ..  连接运算符
__tostring    -->  print()输出时自动调用__tostring进行转换
__metatable   -->  设置后，getmetatable将获取该字段的值，setmetatable将失败报错
```

# table的元方法

## \_\_index

默认情况下，访问table中不存在的元素时返回nil，但是可以通过定义元表中的`__index`元方法来自定义这个行为。

`__index`可以是一个方法，也可以是一个表：  

- 当是一个方法时，将调用该方法来决定访问不存在元素时的返回值  
  - 将以表名和key作为参数调用该方法  
- 当是一个表时，将从该表中寻找是否有该元素，如果有则返回，如果没有则返回nil

如果想要跳过`__index`，可以使用`rawget()`来检索表中元素。

例如，定义一个具有默认值的原型：

```lua
prototype = {x = 0, y = 0, w = 10, h = 20}

local mt = {}
function new(o)
  setmetatable(o, mt)
  return o
end

mt.__index = function (_,k)
  return prototype[k]
end
```

这样定义之后，所有new()创建出来的table都将具有x、y、w、h的四个默认值。例如：

```lua
t = new({x=100,y=100})
print(t.w)
```

在`t.w`的时候，发现t中没有w字段，于是从元表中查找`__index`，它是一个方法，于是以t和w作为参数调用`__index(t,w)`，于是返回`prototype[w]`，即返回10。

也可以直接让`__index`为一个table：

```lua
prototype = {x = 0, y = 0, w = 10, h = 20}

local mt = {}
function new(o)
  setmetatable(o, mt)
  return o
end

mt.__index = prototype

t = new({x=100,y=100})
print(t.w)
```

这样在寻找t.w的时候，将从prototype中找出w字段。

## \_\_newindex

`__index`用于定义查询表中不存在元素时的行为。`__newindex`则用于定义为表中不存在元素进行赋值时的行为。

`__newindex`也可以是两种值：函数或者表。

- 如果是函数，则在为不存在元素赋值时，将调用该函数，而不是进行赋值操作  
- 如果是表，则在此表中进行赋值操作  

可以使用`rawset()`函数跳过`__newindex`，从而强制为元素进行赋值操作。

