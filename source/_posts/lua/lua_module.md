---
title: Lua模块
p: lua/lua_module.md
date: 2020-05-02 10:40:57
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# require导入模块

lua中模块使用table的方式实现：定义一些常量或函数放入表中，然后导出表(return表)。

```lua
--> 导入模块并引用其中函数 <--
local mod = require "mod"
mod.foo()

--> 设置一个局部名称 <--
local m = require("mod")
m.foo()

--> 为其中某函数定义别名 <--
local m = require 'mod'
local f = m.foo
f()

--> 只引入部分函数 <--
local f = require 'mod'.foo --> 等价于(require("mod")).foo
f()
```

require()在导入模块时，如果导入成功，会将导入信息保存到package.loaded中(它是一个表)，使得下次再require()该模块时可以直接返回，而不是重复导入。

如果确实需要重新导入，可将其设置nil，再require()：

```lua
package.loaded.<modname> = nil
local m = require('modname')
```

# lua自定义模块

编写模块，只需创建一个table，然后将相关函数或常量放入table即可。

```lua
local M = {}     --> 用来返回的table，即模块

local function f(x,y)    --> 一个局部函数
  return x + y
end

M.f = f    --> 将局部函数也加入到table中导出

M.I = 1     --> 导出一个常量

function M.g(x,y) return x * y end -->导出一个函数

return M    --> 导出table中的所有内容
```

也可以将所有内容定义成局部变量，然后收集到table中返回：

```lua
local function f() end
local I = 0
local function g() end

return {
  f = f,
  I = I, 
  g = g,
}
```

















