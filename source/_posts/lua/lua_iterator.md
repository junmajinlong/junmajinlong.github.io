---
title: Lua迭代器
p: lua/lua_iterator.md
date: 2020-05-02 10:40:58
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# 定义迭代器

迭代器虽然定义时较为复杂，但使用迭代器很简单，它简化了遍历元素的方式，因此它开头难，后面简单。

在lua中，迭代器通过函数来实现，每次调用函数返回一个元素，直到没有元素时返回nil。

因为迭代时要保存一些中间状态，比如迭代到了哪个位置，所以一般采用闭包作为迭代函数，闭包可以保存中间状态。

因为迭代器是一个闭包，所以还需要一个工厂函数来创建闭包。

例如，下面的迭代器是迭代一个序列中的value：

```lua
function values(tab)
  local i = 0
  return function() i = i + 1; return tab[i] end
end
```

每次调用`values()`将返回一个闭包，即一个独立的迭代器。

可以在循环中不断迭代，比如while循环中：

```lua
t = {10,20,30,40}
local iter = values(t)   --> 返回一个闭包，即迭代器
while true do
  local v = iter()
  if v == nil then break end  --> 不建议if not v then...
  print(v)
end
```

还可以使用更方便的泛型for进行迭代：

```lua
t = {10,20,30,40}
for v in vaules(t) do
  print(v)
end
```

泛型for会在内部保存工厂函数values()创建的迭代函数，并且每次迭代时自动调用该迭代函数，同时将迭代函数的返回结果保存在控制控制变量v中，当迭代函数返回nil时，for将自动结束循环。

# Lua泛型for的工作原理

对于如下泛型for迭代语句：

```lua
for var_list in expr_list do
  code_block
end
```

其中`var_list`是控制变量列表，`expr_list`是一个表达式列表，通常表达式列表是单个元素，即迭代器工厂函数，它会返回迭代器。

泛型for首先会进行初始化操作，即解析in后面的`expr_list`，for要求`expr_list`要返回三个值：

- 迭代器函数  
- 不可变状态  
- 控制变量的初始值  

for会在内部保存这三个返回值，如果`expr_list`的返回值数量不符合for的要求，则会自动进行调整。比如多出的返回值会丢弃，缺少的返回值会以nil补充。所以，如果`expr_list`只返回一个值，则不可变状态和控制变量的初始值都为nil。

即：

```lua
local _f, _s, _i = expr_list
```

然后，for将不可变状态和控制变量初始值作为迭代函数的参数进行调用，并将迭代函数的返回值保存在控制变量中。即将后两个返回值作为参数调用迭代函数：

```lua
local var1, var2,... = _f(_s,_i)
```

当某次迭代中遇到第一个控制变量var1的值为nil，则终止迭代。

所以，for的整个工作过程类似如下代码：

```lua
do
  local _f, _s, _i = expr_list
  while true do
    local var1, var2, var3 ...= _f(_s, _i)
    _i = var1
    if _i == nil then break end
    <CODE_BLOCK>
  end
end
```

例如，按照泛型for的工作方式，实现一个ipairs()：

```lua
local function iter(t, i)
  i = i + 1
  local v = t[i]
  if v ~= nil then return i, v end
end

function ipairs(t)
  return iter, t, 0
end
```

思路：

```lua
local function iter(t, i)
  return i, t[i]
end

function ipairs(t)
  return iter, t, 0
end
```

然后再补充iter()的逻辑，递增索引元素，并判断是否还有元素。

```lua
local function iter(t, i)
  i = i + 1
  local v = t[i]
  if v ~= nil then return i, v end
end
```

当没有元素时，虽然不会在if中进行return，但iter()函数隐式地返回nil并赋值给控制变量i，于是终止迭代。

再比如用泛型for迭代一个链表list，假设该链表有一个next的字段指向下一个节点。那么可编写下面的代码来迭代链表：

```lua
local function getnext(list, node)
  if not node then
    return list
  else
    return node.next
  end
end

function traverse_list(list)
  return getnext, list, nil
end
```











