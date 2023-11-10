---
title: lua table
p: lua/lua_table.md
date: 2020-05-02 10:40:40
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# table

Lua的table数据类型是同时包含数组结构和hash结构的组合体，table中的数组和hash分开存放。

![](/img/lua/1587224877344.png)

## table基础

```lua
a = {}    --> 空table

a[1] = 10
a["x"] = "xxx"
print(a['yy'])     --> nil
```

访问不存在的元素，其返回值为nil。

除了nil值，所有数据都可以作为table的索引。正因为如此，数值类型和字符串类型的键可能会出现容易混淆的键：

```lua
a ={}
a[1]=10
a["1"]=100

a[1]     --> 10
a[1.0]   --> 10
a["1"]   --> 100
```

当字符串索引符合标识符规范时，`a["xxx"]`可以写成等价的`a.xxx`。

```lua
a = {}
a["key"] = "value"
a.key     --> value
a.1       --> syntax error near '.1'
```

# 构造table

## 构造数组型的table

```lua
a = {}
a[1] = "Perl"
a[2] = "PHP"
a[3] = "Ruby"
a[4] = "Lua"

-- 直接构造
a1 = {"Perl", "PHP", "Ruby", "Lua"}
print(a1[1])   --> Perl

-- key/value方式构造
a2 = {[1] = "Perl", [2] = "PHP", [3] = "Ruby", [4] = "Lua"}
print(a2[1])   --> Perl
```

使用`#`获取数组长度：

```lua
a = {"Perl", "PHP", "Ruby", "Lua"}
print(#a)    --> 4
```

从数值索引1开始直到遇到nil终止的部分属于序列，即序列是数组的一部分，长度为n的索引范围为`{1,n}`。

要注意，如果数值索引出现了空隙，长度的计算将不可靠。

```lua
a = {"Perl", "PHP", "Ruby", "Lua"}
a[10] = "Python"
a[11] = "Javascript"
print(#a)   --> 4

-- 索引不从1开始
a1 = {}
a1[3] = "Perl"
a1[4] = "PHP"
a1[5] = "Java"
print(#a1)     --> 0
-- 补上a1[1]和a1[2]
a1[1] = "Ruby"
a1[2] = "Lua"
print(#a1)     --> 5

-- 索引从2开始
-- 长度为4，因为从1开始，到第一个nil结束，只是nil正好在1上
-- 出现了定义的歧义，虽然长度为4，但不可遍历
a1 = {}
a1[2] = "Perl"
a1[3] = "PHP"
a1[4] = "Java"
print(#a1)     --> 4 
for i in ipairs(a1) do print(i) end  --> 不输出任何东西
```

因table中的数组的特殊性，"长度"的概念并不很容易定义，通过下图可能更好描述一些。

![](/img/lua/1587225467430.png)

现在table的数组空间共有6个元素，`#`计算的结果为4，为什么不计算为6呢？为什么不计算为9呢？别考虑这些，因为是个迷。

```lua
a = {}
for i = 1,100 do a[i] = i * 2 end
#a        --> 100
for i = 50,60 do a[i] = nil end
#a        --> 100
for i = 50,64 do a[i] = nil end
#a        --> 49
```

如果真的需要记录数组中的元素个数，可将实际的元素个数保存在该table的hash中。比如上面6个元素，可保存`a["__n"] = 6`。


## 构造hash型的table

```lua
a = {}
a["one"] = 1
a["two"] = 2
a["three"] = 3

--> 直接构造，key不加引号，结尾逗号可选
a1 = {one = 1, two = 2, three = 3}
a11 = {one = 1, two = 2, three = 3,}

--> key符合标识符规则时，a.x等价于a["x"]
a2 = {}
a2.one = 1
a2.two = 2
a2.three = 3
```

## 构造数组、hash混合的table

```lua
a = {"perl","php","shell",
      one = 1,
      two = 2,
      three = 3,
      "lua","python",
}

print(#a)      --> 5
print(a[4])    --> lua
```

还可以类型嵌套：

```lua
a = {"perl","php","shell",
      { one = 1, oneone = 11 },
      { two = 2, twotwo = 22 },
      { three = 3, threethree = 33 },
      "lua","python",
}

print(a[4])            --> table: 0x7fffc3401960
print(a[4]["one"])     --> 1
```

## 通用构造式

更通用的构造方式是`[key] = value`方式。

例如：

```lua
a = { ["one"] = 1,
      ["two"] = 2,
      [1] = "one",
      [2] = "two",
}
```

这种通用构造方式可以构造hash的键值对，且key可以包含特殊符号，也可以以数值作为key构造数组元素。

# 遍历table

## 遍历所有元素

使用pairs()遍历table中的所有元素，包括数组和hash，会同时被遍历。注意，pairs()遍历时，只能保证每个元素只出现一次，不保证元素出现的顺序，即使数值索引的顺序也不保证。

```lua
a = {"perl","php","shell",
      one = 1,
      two = 2,
      three = 3,
      "lua","python",
}
a[10] = "java"
a[11] = "javascript"

--> 以下等价
for k,v in pairs(a) do print(k,v) end
for i in pairs(a) do print(i, a[i]) end
--[[
1       perl
2       php
3       shell
4       lua
5       python
one     1
two     2
11      javascript
10      java
three   3
]]
```

## 遍历序列

序列是从索引1开始到第一个nil元素为止的数组。

```lua
a = {"Perl", "Lua", "Go", "Shell"}
a[6] = "Python"
a[7] = "Java"

--> 方法1：使用长度进行遍历，
-->       注意，对于有间隙的数组，长度不可靠
for i = 1, #a do
  print(i, a[i])     --> 全输出，包括a[5] = nil也输出
end
--[[
1       Perl
2       Lua
3       Go
4       Shell
5       nil
6       Python
7       Java
]]

--> 方法2：使用ipairs(a)进行遍历，只会遍历序列部分
for i in ipairs(a) do print(i, a[i]) end
--[[
1       Perl
2       Lua
3       Go
4       Shell
]]
```

## 数组重构成序列

如果想要移除数组的所有间隙，将所有数组元素重组成一个序列：

```lua
a = {"perl","php","shell","lua","python"}
a[10] = "java"
a[11] = "javascript"
a[33] = "c"
a[44] = "go"
a[45] = "sql"

-- 因为pairs()遍历不保证顺序(即使是数值索引也不保证顺序)
-- 所以必须先将数值索引排序
a.idx = {}
for i in pairs(a) do 
  if(type(i) == "number") then
    a.idx[ #a.idx+1 ] = i
  end
end
table.sort(a.idx)
for _,i in pairs(a.idx) do 
  cnt = ( cnt or 0 ) + 1
  a[cnt] = a[i]
  if cnt ~= i then a[i]=nil end
end
a.idx = nil
```

# table标准库

table标准库只有7个函数：它们全都只适合于**序列**

```
table.insert
table.remove
table.move
table.concat
table.sort
table.pack
table.unpack
```

## table.concat

用于串联table中**序列部分**的元素。

```
table.concat (list [, sep [, i [, j]]])

sep： 串联的连接分隔符
i：起使索引，默认为1
j：终止索引，默认为序列的最后一个索引位置
注：
 (1).索引对应的元素必须存在(即，也不能包含间隙)，否则报错
 (2).i大于j时，返回空串
```

例如：

```lua
a = {"perl","php","shell",
      one = 1,
      two = 2,
      three = 3,
      "lua","python",
}
a[10] = "java"
a[11] = "javascript"
```

不给任何额外参数时：

```lua
table.concat(a)     --> perlphpshellluapython
```

指定分隔符：

```lua
table.concat(a,"-")   --> perl-php-shell-lua-python
```

指定起使、终止的索引位置：

```lua
table.concat(a,"-",1,3)   --> perl-php-shell
table.concat(a,"-",3)     --> shell-lua-python
```

## table.insert

向table的**序列部分**插入元素。

```
table.insert (list, [pos,] value)
注：
 (1).省略pos时，表示插入在序列的结尾
 (2).插入到指定位置后，序列中该元素后的元素全都后移，
     但间隙后的元素不会移动
```

例如：

```lua
a = {"perl","php","shell","lua","python"}
a[10] = "java"
a[11] = "javascript"

--> 不给位置参数时，插入在序列尾部，即push
table.insert(a, "go")
a[6]     --> "go"

table.insert(a, 7, "c")
a[7]     --> "c"

--> 补全间隙
table.insert(a, 8, "sql")
table.insert(a, 9, "c++")

--> 再插入，后面的元素都移动
table.insert(a, 9, "lisp")
a[10]    --> "c++"
```

## table.remove

从table的序列中移除某个元素并返回该元素：

```
table.remove (list [, pos])

注： 
  (1).如果不指定pos，则默认移除序列的最后一个元素
  (2).移除序列某元素后，其后方元素将前移，但间隙后的元素不受影响
  (3).序列无元素可移除时，返回nil
```

例如：

```lua
a = {"perl","php","shell","lua","python"}
a[10] = "java"
a[11] = "javascript"

--> 移除序列的最后一个元素，即pop
table.remove(a)    --> python
a[5]     --> nil
a[10]    --> java

--> 移除指定位置处的元素
table.remove(a, 3)  --> shell
a[3]                --> lua
a[4]                --> nil
a[10]               --> java
```

## table.move

移动table中的元素。

```
table.move (a1, f, e, t [,a2])

注： 
  (1).将table a1中的a[f]到a[e]部分的元素移动到a2的a2[t],a2[t+1]...
  (2).不指定a2时，a2为a1，即直接在a1中覆盖式移动
  (3).被移动元素自身不会被删除，换句话说，不应该称之为移动，而是元素拷贝
  (4).返回a2表，但要求a2表已经存在，如果未指定a2，则返回a1表自身
  (5).如果将返回值赋值给某变量，则该变量和a2引用同一table
```

例如：

```lua
a = {"perl","php","shell","lua","python"}
a[10] = "java"
a[11] = "javascript"

a2 = {}
--> a2和a3引用同一张表
a3 = table.move(a,10,11,1,a2)
a2[1] == a3[1]    --> true

a2[1] = "JAVA"
a3[1]             --> JAVA
```

table.move()可用于拷贝序列。

```lua
--> 将table a的序列部分拷贝到a1
a1 = table.move(a,1,#a,1,a1)
```

## table.sort

对序列进行排序：

```
table.sort (list [, comp])
```

原处排序(排序后，原table中元素的顺序发生改变)，默认升序排序，comp为可选参数，其为函数，用于指定排序依据。

```lua
a = {"perl","php","shell","lua","python"}
a[10] = "java"
a[11] = "javascript"
a1 = table.move(a,1,#a,1,a1)

--> 默认排序规则：升序
table.sort(a)
for k,v in ipairs(a) do print(k,v) end
--[[
1       lua
2       perl
3       php
4       python
5       shell
]]

--> 指定排序规则，按字符串长度降序排序
table.sort(a1,function(x,y) return #x > #y end)
for k,v in ipairs(a1) do print(k,v) end
--[[
1       python
2       shell
3       perl
4       php
5       lua
]]

--> 指定排序规则，从另一个表读取数据作为排序依据
list = {"perl","shell","php","python","lua"}
score = {
  perl = 70,
  shell = 66,
  php = 69,
  python = 69,
  lua = 63,
}
table.sort(list,function(n1,n2) return score[n1] > score[n2] end)
for i in ipairs(list) do print(i,list[i]) end
--[[
1       perl
2       python
3       php
4       shell
5       lua
]]
```

## table.pack

将一系列元素打包成table：

```
table.pack (···)
```

它会返回一个table，该table包含从1开始的序列，以及一个名为"n"的key，其值为被打包参数的数量，也即序列的长度。

```lua
a = table.pack("a","b","c","d","e")
for k,v in pairs(a) do print(k,v) end
--[[
1       a
2       b
3       c
4       d
5       e
n       5
]]
```

## table.unpack

序列解包：

```
table.unpack (list [, i [, j]])

注：
  (1).等价于return list[i], list[i+1], ···, list[j]
  (2).i默认值为1，j默认值为#list
```

例如：

```lua
a = {"a","b","c","d","e",n=5}
print(table.unpack(a))      --> a  b  c  d  e
print(table.unpack(a,2,3))  --> b  c
print(table.unpack(a,2,2))  --> b
```

# table实现一个双端队列

table实现双端队列非常简单，直接在table中记录first和last两个端点即可。

```lua
function dequeue()
  return {first = 0, last = -1}
end

-- push value to queue
function push(l, v)
  local last = l.last + 1
  l.last = last
  l[last] = v
end

-- unshift value to queue
function unshift(l, v)
  local first = l.first - 1
  l[first] = v
  l.first = first
end

-- pop value from queue
function pop(l)
  local last = l.last
  if last < l.first then error("empty queue") end
  l.last = last - 1
  local last_v = l[last]
  l[last] = nil
  return last_v
end

-- shift value from queue
function shift(l)
  local first = l.first
  if first > l.last then error("empth queue") end
  l.first = first + 1
  local first_v = l[first]
  l[first] = nil
  return first_v
end
```
