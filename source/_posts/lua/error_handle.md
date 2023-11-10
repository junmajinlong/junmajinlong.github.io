---
title: Lua错误处理
p: lua/error_handle.md
date: 2020-05-02 10:40:56
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

lua一般嵌入在宿主语言中，所以不能简单地因报错而退出，它的错误必须合理处理。

## error和assert

error()可以显式抛出错误信息：

```lua
n = io.read("n")
if not n then error("invalid input") end
```

assert()是对如上代码模式的简化：

```
assert (v [, message])
```

assert()检测第一个参数v是否为真，如果v为真，则返回v的值，否则报错。报错信息来源于第二个参数message，如果省略第二个参数，则默认报错信息为"assertion failed!"。

如果v是一个函数，如果这个函数报错，通常来说，它会在返回nil或false的同时返回第二个和第三个值，第二个返回值是错误描述信息，第三个返回值是错误代码。函数错误时的多返回值正好可应用在assert()中，因为错误描述信息会成为assert()的第二个参数。

```lua
function f() return nil,"error message" end
assert(f())      --> 报错 stdin:1: error message
```

## 异常处理

当出现异常时，一般两种处理方式：  

- 返回错误代码(通常是nil或false)并基于一段错误描述信息  
- 直接error抛出错误  

至于选择何种处理方式，没有定论。

## pcall()

pcall()用于捕获异常，要求代码封装到一个函数中：

```
pcall(function()
  some code...
end
)
```

如果没有错误，则pcall返回true以及函数的所有返回值，否则返回false以及报错信息。

lua一般嵌入在宿主语言中，所以一般不需要进行pcall()捕获异常，而是在宿主语言中进行合理的处理。



