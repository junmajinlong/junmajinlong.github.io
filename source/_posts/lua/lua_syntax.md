---
title: Lua基本语法简述
p: lua/lua_syntax.md
date: 2020-05-02 10:40:36
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# lua运行方式

1. lua解释器直接执行lua代码文件：lua test.lua  
2. lua脚本方式执行，在代码文件第一行加上shabang，例如`#!/usr/bin/lua`  
3. lua命令的-e选项，例如：`lua -e 'print("Hello World")'`  
4. 可进入lua交互式Shell，直接输入lua命令即可  
  - 设置`_PROMPT`全局变量可设置交互式lua shell的命令提示符  
  - `lua -i -e '_PROMPT = "lua> "'`，其中-i表示先执行代码，再进入交互式Shell

# lua关键字

lua没有case语句。

```
not and or
true false
nil
in
local

do
end
if then else elseif
while for repeat until break
function return
```

# lua运算符

```lua
-- 没有自增自减运算a++，没有运算且赋值运算a+=1
-- 没有三目运算

+     -     *     /     %     ^
#    -- 取sequence table的长度
=
&     ~     |     <<    >>    //
==    ~=    <=    >=    <     >   -- ~=是不等比较，不是正则匹配
()    {}    []    ::
;     :     ,     .     ..    ...
```

位操作符是Lua 5.3引入的。

除了求幂运算符`^`和字符串串联符号`..`外，其它操作符都是以从左向右的顺序运算的。但是`^ ..`这两个运算符是右结合优先的。

```lua
3 ^ 2 ^ 2   --> 3^(2^2)
```

加法运算只适用于数值类型，如果相加涉及字符串，则尝试对字符串进行隐式数据转换。无法转换则报错。

字符串串联则尝试转换成字符串。

```lua
3 + "3"    --> 6.0
3 + "3a"   --> 报错
"a"..2     --> a2
```

# lua注释

```lua
print("Hello World") --comment

--[[
  print("Hello World")   --块注释
--]]
```

特别地，想要重新启用块注释里的代码，只需如下加上一个开头的短横线即可：

```lua
---[[      -- 这是单独的行注释，而非块注释
  print("Hello World")
--]]       -- 这是单独的行注释，而非块注释
```

# lua标识符

大小写字母、数字、下划线可参与命名。

- 不能数字开头  
- 约定俗成地，`_`当作哑变量  
- 下划线开头后全是大写字母的作为特殊对待的符号，应避免使用，如`_VERSION`  

# 语句解析：分号和换行

下面是等价的：

```lua
a = 1
b = a * 2

a = 1;
b = a * 2;

a=1; b = a * 2

a = 1 b = a * 2    -- 很难看懂
```

# lua全局变量

全局变量可无需声明直接引用，对于未赋值的全局变量，访问时其值为nil。

```lua
print(b)   -->nil
```

赋值变量后，可重新将nil赋值给它来表示该变量未定义：

```lua
b = 10
print(b)
b = nil
print(b)
```

# lua命令行的参数解析

lua在代码开始执行前，先解析命令行参数，它会将参数保存在一个名为`arg`的table中(table即关联数组或hash结构)，脚本名称为`arg[0]`，第一个位置参数为`arg[1]`。

此外，lua命令允许`-e "program"`和lua代码文件同时执行，此时脚本名前面的参数全保存在负数索引中。

例如：

```shell
lua -e 'print("hello world")' test.lua a b
```

那么：

```
arg[0] = "test.lua"
arg[1] = "a"
arg[2] = "b"
arg[-1] = 'print("hello world")'
arg[-2] = "-e"
arg[-3] = "lua"
```





