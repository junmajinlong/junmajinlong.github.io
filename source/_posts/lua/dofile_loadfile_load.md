---
title: Lua dofile、loadfile和load
p: lua/dofile_loadfile_load.md
date: 2020-05-02 10:40:55
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

# dofile

dofile(FILENAME)加载文件中的代码，且加载代码时遇到错误会报错。

例如，a.lua文件内定义两个函数：

```lua
function f_a(x,y)
  return x + y
end

function g_a(x,y)
  return x * y
end
```

那么可以dofile加载这个文件并使用这两个函数：

```lua
dofile('a.lua')
f_a(3,4)        --> 7
g_a(3,4)        --> 12
```

# loadfile

loadfile()比dofile()更底层一些，它只加载代码，但有错误时不报错。而且，loadfile()返回一个函数，执行该函数后才会得到所加载文件中的内容。

实际上，dofile是由loadfile实现的，类似如下：

```lua
function dofile(filename)
  local f = assert(loadfile(filename))
  return f()
end
```

loadfile的基本使用方式如下：

```lua
local f = assert(loadfile("a.lua"))
f()
f_a(3,4)   --> 7
g_a(3,4)   --> 12
```

对于用户来说，可以使用loadfile()实现更灵活更自定义的功能，而且它加载文件后返回一个函数，这个函数可以复用。换句话说，它只做一次编译(但每次loadfile()都会编译)，而dofile()需要每次调用都编译。

```lua
loadfile('a.lua')       --> function: 0x7ffff0112910
loadfile('a.lua')       --> function: 0x7ffff0113230
```

# load

```
load (chunk)
```

load()从给定字符串中读取代码或从一个函数中读取代码，然后以函数的方式返回。如果chunk是一个函数，load()将不断调用该函数来获取代码片段，直到该函数遇到nil。

例如从字符串中读取代码：

```lua
f = load("i = i + 1")

i = 0
f();print(i)       --> 1
f();print(i)       --> 2
```

当执行`f = load("i = i + 1")`后，类似于执行了如下代码：

```lua
f = function (...) i = i + 1 end
```

注意load()时会**自动设置其形参为可变长度参数**的`...`。

load()方式比直接定义function要慢，因为它要额外编译。而且，load()总是在全局环境下进行编译，所以它不涉及词法作用域规则。

```lua
i = 32
local i = 0 
f = load("i = i + 1;print(i)")
g = function () i = i + 1; print(i) end
f()      --> 33
g()      --> 1
```

不过，load()的字符串代码中可以手动加入局部变量的声明语句，还可以加入return语句。例如：

```lua
local i = 0 
f = load("local i = ...;i = i + 1;return i")
g = function () i = i + 1; return i end
print(f(i))     --> 1
print(g())      --> 1
```

因为load()以函数方式返回，所以可以加载后立即执行这段代码，然后立即丢弃。即一次性代码：

```lua
i = 0
assert(load("i = i + 1"))()
print(i)          --> 1
```

load()常用于执行动态代码，比如读取用户输入的代码(可使用`io.read()`读取用户输入)，读取文件中的代码，读取宿主语言传送过来的代码。

下面是使用io.lines()读取文件内的代码：

```lua
f = load(io.lines(filename, "L"))
f = load(io.lines(filename, 1024))    --> 按块读取，效率更高
```

这样的读取方式会一直读取，直到io.lines()遇到nil，load()才加载代码完成。

# 预编译代码

使用luac命令可以对一个lua源码文件进行预编译，预编译后得到的是二进制文件，隐藏了源码内容。

几乎所有使用源代码的地方，都可以使用预编译后的代码。例如load()、loadfile()都可以加载预编译代码，且加载它们会更快。

load()和loadfile()都有相关的模式参数可以控制只能加载文本代码、二进制代码(即预编译代码)或两者均可。

```
load (chunk [, chunkname [, mode [, env]]])
loadfile ([filename [, mode [, env]]])
```

其中mode值：

- "bt"：表示可加载预编译代码也可以加载文本代码  
- "b"：表示可加载预编译代码  
- "t"：表示可加载文本代码  

`string.dump(func_name,[strip])`可以将一个函数进行预编译并返回它的二进制格式(但注意，它仍然是字符串类型，只是以二进制方式保存)，这正好可以用在load()中。其中的strip参数可用于移除func_name函数中调试信息，可使得预编译后的字符串更小。

例如：

```lua
function f(x,y) return x + y end
ff = load(string.dump(f,strip))
ff(3,4)     --> 7
```