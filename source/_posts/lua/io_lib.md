---
title: Lua IO操作
p: lua/io_lib.md
date: 2020-05-02 10:40:43
tags: Lua
categories: Lua

---

------

**回到：**  

- **[Lua系列文章](/lua/index)**  

------

## 简单IO模型

简单IO模型下，Lua只有两个文件的io流：输入流和输出流，默认的输入流是stdin，默认的输出流是stdout。所有操作都在这两个文件上操作。io.read()和io.write()两个函数可分别操作这两个IO流。

```lua
echo -e "hello\nworld" | lua -e 'print(io.read())'
```

使用`io.input(FILENAME)`和`io.output(FILENAME)`可以改变默认的输入流和输出流，使用它们设置输入流或输出流后，后续的IO操作都将基于其指定的文件，除非再次调用它们改变IO流。如果出现异常(比如无权限/文件不存在)，将直接抛出错误。后面还会介绍这两个函数的其它功能。

![](/img/lua/1587350639181.png)

```lua
-->  从/etc/passwd中读取  <--
io.input("/etc/passwd")      --> file (0x7fffd2349390)
io.read()
--> root:x:0:0:root:/root:/bin/bash
io.read()
--> daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin

-->  从/etc/resolv.conf中读取  <--
io.input("/etc/resolv.conf")  --> file (0x7fffd2340880)
io.read()
--> # This file was automatically...
io.read()
-->nameserver 192.168.1.1

-->  写入终端  <--
lua -e 'io.write("hello world","\n")'
hello world

-->  写入指定文件  <--
lua -e 'io.output("/tmp/a.log");io.write("hello world","\n")'
```

io.write()写入文件时，将覆盖原文件，当文件不存在时，将创建。

io.write()时，不建议采用`io.write("hello".."world")`将字符串连接起来，而是将各段内容作为io.write()的参数。例如：

```lua
io.write("hello".."world".."!\n")
io.write("hello","world","1\n")
```

### io.read()

io.read()默认每次读取一行，它有可选的参数来指定如何读取数据。当参数为：  

- `a`时，(从当前位置开始)读取整个文件内容，读取不到内容返回nil  
- `l`时，(从当前位置开始)读取一行内容，丢弃尾部换行符，读取不到内容返回nil  
- `L`时，(从当前位置开始)读取一行内容，保留尾部换行符，读取不到内容返回nil  
- `n`时，(从当前位置开始)读取一个数值，读取成功则返回该**数值**，否则返回nil  
- `NUM`时，表示(从当前位置开始)读取至多NUM个字符(即按块读取)，读取不到内容返回nil  
  - 特别地，如果NUM=0，即io.read(0)，它表示读取0个字符，可用于测试是否到了文件尾部，如果未到尾部，则返回空字符串，如果到了尾部，则返回nil  
- io.read()可以有多个参数(参数如上所述)，它将依次按照每个参数指定的方式读取数据(参见下方示例)  

例如，可以将(不算太大的)文件所有内容读取到变量中进行筛选：

```lua
io.input("/tmp/a.log")
data = io.read("a")
data = string.gsub(data, "hello", "HELLO")
```

通过`l`或`L`可以迭代行。例如：

```lua
io.input("/etc/resolv.conf")
while true do
  data = io.read("L")
  if not data then break end
  io.write(data)
end
```

也可以使用`io.lines()`来迭代行，注意，它不会保留尾部换行符：

```lua
io.input("/etc/resolv.conf")
for line in io.lines() do
  io.write(line,"\n")
end
```

例如，要对文件按行进行排序：

```lua
io.input("/etc/resolv.conf")
local lines = {}
for line in io.lines() do
  lines[#lines+1] = line
end

table.sort(lines)
for _,line in ipairs(lines) do 
  io.write(line,"\n")
end
```

如果文件较大，那么一次性读取所有内容将消耗比较大的内存，为了更高效读取，可以按块进行读取，即`io.read(NUM)`。

例如，下面高效拷贝一个文件：

```lua
io.input("bigfile")
io.output("outfile")

local data = nil
while true do
  data = io.read(8192)
  if not data then break end
  io.write(data)
end
```

按块读取时可能会读到行中间，有时候为了处理完整的行，可以按块读取之后再按行读取一次，这样便保证了行的完整性。

例如：

```lua
io.input("/etc/passwd")
io.output("passwd")

while true do
  data = io.read(8192)
  if not data then break end

  --> 一定要用L选项，而不能自己加"\n"，否则有些文件尾部没有\n的将导致文件不同
  line_end = io.read("L")
  if not line_end then line_end = "" end
  io.write(data,line_end)
end
```

如果使用`io.read("n")`，则从前向后扫描，直到遇到数值(可识别正负号)，然后读取该数值。但是一定要注意，扫描时只能跳过空白符号，如果有字母、下划线或其它符号，则读取立即停止，它认为文件格式错误。所以，该读取功能的使用场景是非常受限的。

此外，io.read()可以提供多个参数。

例如：

```lua
a,b,c = io.read("n","n","n")    --> 读取三个数值，赋值给a b c
a,b,c = io.read("L","L","L")    --> 读取三行，分别赋值给a b c
```

## 复杂io模型

复杂IO模型使用文件句柄来执行IO操作。

> 简单IO模型是复杂IO模型的特例，它使用的是隐式的文件句柄。

使用`io.open(FILENAME,MODE)`可打开文件并返回文件句柄。其中MODE:  

- "r": 只读方式打开文件(默认模式)  
- "w": 可写方式打开文件  
- "a": 追加模式打开文件  
- "r+": 更新模式打开文件，可读可写，文件指针置于文件尾部  
- "w+": 更新模式打开文件，可读可写，文件指针置于文件头部  
- "a+": 追加更新模式打开文件，可读可写，文件指针置于文件尾部  

还可以在上述模式后加上`b`，表示二进制方式打开文件。

如果io.open()打开错误，则返回nil并报错。

```lua
io.open("/etc/fstab")
--> file (0x7fffd237bd00)

io.open("/etc/fstabbb")
-->   nil   /etc/fstabbb: No such file or directory 2
```

所以，根据io.open()的返回值可判断是否成功，并做出错误处理。当然，更常见的处理方式是使用assert()：

```lua
fh,err = io.open(FILENAME,MODE)
if not fh then print(err) end

fh = assert(io.open(FILENAME,MODE))
```

文件句柄是对象，通过它来执行IO操作，需以对象的方式来调用各函数：

```lua
<--  以下是Lua支持的文件句柄对象的方法  -->
fh:read()
fh:write()
fh:lines()
fh:close()
fh:flush()
fh:seek()
fh:setvbuf()
```

Lua提供了三个预定义的文件句柄：**io.stdin、io.stdout和io.stderr**。所以，从标准输入读数据、写入标准输出和写入标准错误的方式：

```lua
io.stdin:read()
io.stdout:write()
io.stderr:write()
```

io.input()、io.output()可以**获取或设置**当前默认的输入流、输出流：  
- 当**无参数**时，可获取当前的输入流、输出流  
- 当给定**字符串参数**时，可打开某文件并设置其为当前的输入流、输出流  
- 当给定**文件句柄对象作为参数**时，可设置该句柄作为当前的输入流、输出流  

例如：

```lua
local tmp_fh = io.input()   --> 备份当前输入流

io.input("a.log")      --> 打开a.log并设置其隐式句柄作为当前输入流
io.input():close()     --> 关闭当前输入流，即关闭a.log对应的句柄
io.input(tmp_fh)       --> 恢复输入流
```

实际上，io.read()和io.write()分别是`io.input():read()`以及`io.output:write()`的简写形式。

### io.type()

```
io.type (obj)
```

检测obj是否是一个有效的句柄：

- 如果obj是一个已经打开的文件句柄，则返回"file"字符串  
- 如果obj是一个已关闭的文件句柄，则返回"closed file"字符串  
- 如果不是文件句柄，则返回nil  

### io.lines()

```
io.lines([filename,read_mode])
fh:lines(read_mode)
```

`io.lines()`或`fh:lines()`返回一个不断读取文件数据的迭代器。

对于io.lines()来说：  
- 当不指定任何参数时，则从当前输入流中按照read_mode指定的方式不断读取数据  
- 当指定文件名时，将以只读方式打开该文件并按照read_mode指定的方式不断读取数据   

从Lua 5.2开始，`fh:lines()`和`io.lines()`的read_mode用于指定读取方式，是和io.read()一样的参数，比如io.lines(8192)。

例如：

```lua
for line in io.lines("/etc/resolv.conf","L") do
  io.write(line)
end

local fh = assert(io.open("/etc/resolv.conf"))
for line in fh:lines("L") do
  io.write(line)
end
```

### io.flush()

flush写io的缓存。

- io.flush() flush当前的输出流buffer  

- fh:flush() flush fh的输出流buffer  

### io.setvbuf()

设置io buffer的缓冲模式。

```
file:setvbuf (mode [, size])
```

mode的值：  

- "no"：无缓冲模式，数据直接写入文件  
- "full"：全缓冲(或块缓冲)模式，即堆满缓冲空间才写入文件  
- "line"：行缓冲模式，遇到换行符才写入文件  

对于full和line两种模式，可以设置第二个参数，它是一个表示字节数的数值，用于指定缓冲空间的大小。它们有默认值，默认值应该来源于操作系统。

### fh:seek()

获取或设置文件指针的位置。

```
fh:seek ([whence [, offset]])
```

返回设置后的文件指针所在位置偏移。

whence的值：  

- "set"：表示相对于文件开头的偏移  
- "cur"：表示相对于当前位置的偏移  
- "end"：表示相对于文件尾部的偏移  

offset的单位为字节。

当不给定任何参数时，其whence默认值为cur，offset默认值为0。所以：

- `fh:seek()`可获取当前文件指针的偏移量  
- `fh:seek("set")`将指针重置到文件开头  
- `fh:seek("end")`将偏移到文件结尾并返回结尾处的偏移量，即文件大小  

所以，要临时获取文件大小时，可如下操作：

```lua
function fsize(fh)
  if io.type(fh) ~= "file" then error("not a file handler") end
  local cur = fh:seek()           --> 备份当前偏移位置
  local size = file:seek("end")   --> 偏移到文件尾部
  file:seek("set", cur)           --> 恢复偏移
  return size
end
```

### io.popen()

运行外部程序，并指定向其写入数据还是从中读取数据。

```
io.popen (prog [, mode])
```

以子进程的方式运行外部程序，并返回一个文件句柄。可指定模式参数mode：  

- "r"：表示从外部程序读数据  
- "w"：表示向外部程序写数据  

所以：

```
io.popen(CMD,"r")     -->  CMD | lua_program
io.popen(CMD,"w")     -->  lua_program | CMD
```

例如：

```lua
fh = io.popen("head -n 5 /etc/passwd", "r")
print(fh:read("a"))
--[[
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
]]
```

### io.tmpfile()

`io.tmpfile()`创建一个临时文件并返回其句柄，该句柄可读可写，在程序终止时文件自动删除。

### 二进制文件读写

io.input()和io.output()总以文本方式打开文件，在Unix系统中，二进制文件和文本文件是没有区别的，但是在Windows系统中，必须以二进制方式打开文件才能识别某些特殊的字符。

通常来说，以二进制方式打开文件时的模式要么是"a"表示读取所有字节，要么是NUM表示读取指定字节数量的数据。在二进制读写中，没有行的概念，换行符自身就是二进制字节，所以按行读取是没有意义的。

例如，DOS格式的文件转换成UNIX格式：

```lua
local fh_in = assert(io.open(arg[1]),"rb")
local fh_out = assert(io.open(arg[2]),"wb")

local data = fh_in:read("a")
data = string.gsub(data, "\r\n", "\n")
fh_out:write(data)
assert(fh_in:close())
assert(fh_out:close())
```

## 字符串缓冲区

想要将所有包含某类字符的行串联起来：

```lua
local buf = ""
io.input("/tmp/a.log")
for line in io.lines('l')
  if string.find(line,"error") then buf = buf..line.."\n" end
end
```

这种方式效率很低，因为每次字符串连接的时候都会拷贝之前的buf。而这具有滚雪球效应，比如buf已经保存了2K数据，保存下一行时，将拷贝这2K，下一次又拷贝2K+。

比较高效的方式是将所有符合条件的行保存在序列中：

```lua
-- store lines into table
local buf = {}
io.input("/tmp/a.log")
for line in io.lines('l')
  if string.find(line,"error") then buf[#buf + 1] = line end
end

-- concat table elements
s = table.concat(buf,"\n").."\n"
```

为了避免最后一次字符串串联导致的拷贝，可以在table.concat()之前，额外添加一个尾部空字符串元素。

```lua
-- store lines into table
local buf = {}
io.input("/tmp/a.log")
for line in io.lines('l')
  if string.find(line,"error") then buf[#buf + 1] = line end
end

-- concat table elements
buf[#buf + 1] = ""
s = table.concat(buf,"\n")
buf[#buf] = nil
```































