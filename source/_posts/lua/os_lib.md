---
title: Lua OS标准库
p: lua/os_lib.md
date: 2020-05-02 10:40:44
tags: Lua
categories: Lua

---

------

**回到：**  

- **[Lua系列文章](/lua/index)**  

------

# OS库

Lua的OS库提供的功能较少。

```
os.exit()          --> 退出程序
os.tmpname()       --> 返回一个临时文件名
os.rename()        --> 文件重命名
os.remove()        --> 移除文件
os.execute()       --> 执行外部命令
os.getenv()        --> 获取环境变量
os.setlocale()     --> 设置locale

os.time()          --> 获取日期时间
os.data()          --> 解析日期时间为字符串或table
os.difftime()      --> 计算时间差
os.clock()         --> 计算代码执行时长
```

## os.exit()

```
os.exit ([code [, close]])
```

退出程序。可指定退出状态码，code参数可为true、false或一个数值。

## os.getenv()

```
os.getenv (varname)
```

获取指定环境变量varname的值，如果指定的环境变量不存在，则返回nil。

## os.rename()

```
os.rename(oldname, newname)
```

文件重命名，如果重命名成功，返回true，否则返回nil，同时返回报错信息和错误状态码。

## os.remove()

```
os.remove (filename)
```

删除名为filename的文件或空目录。删除成功，则返回true，否则返回nil，同时返回报错信息和错误状态码。

## os.tmpname()

```
os.tmpname()
```

返回一个可用于操作临时文件的字符串，它会自动创建该临时文件。返回临时文件名后，该文件还必须显式打开以及显式删除。

如果可以的话，建议使用`io.tmpfile()`来返回一个临时文件句柄，它会在程序退出的时候自动删除。

## os.execute()

```
os.execute([cmd])
```

该函数等价于C的system()函数。它实际上是将参数cmd传递给操作系统的shell去执行。

第一个返回值如果为true时，表示命令cmd成功执行完成，否则该返回值为nil。

第二个返回值和第三个返回值分别是描述字符串和一个数值：  

- 第二个返回值为"exit"，表示命令正常退出，此时第三个返回值为命令的退出状态码  
- 第二个返回值为"signal"：表示命令被信号终止，第三个返回值为终止该进程的信号代码  

例如，创建目录：

```lua
os.execute("mkdir -p"..dirname)
```

## os.clock()

返回CPU执行时长。通过它可以计算一段代码执行时CPU的耗费时间。

```lua
local start = os.clock()
local s = 0
for i = 1,10000 do s = s + i end
print(os.clock() - start)
```

## os.time()

构建时间点。

```
os.time([table])
```

如果不给参数，返回当前时间点的epoch值。

如果给定一个table类型的参数，则返回对应的epoch值。

参数是一个包含如下字段的table：

- year
- month
- day
- hour(默认值为12)
- min(默认值为0)
- sec(默认值为0)
- isdst(默认值为nil)

这些字段的数值大小可以不在它们的有效范围内，例如sec字段指定为-10，将自动后退10秒，hour为100，将自动进位到day甚至month。

例如：

```lua
os.time()
--> 1587377837

os.time({year=2017,month=10,day=12})
--> 1507780800

os.time({year=2017,month=10,day=12,hour=12,min=0,sec=1})
--> 1507780801

os.time({year=2017,month=10,day=12,hour=12,min=0,sec=-1})
--> 1507780799
```

## os.difftime()

```
os.difftime (t2, t1)
```

返回时间差，单位秒，其中t2和t1都是epoch值。它等价于t2-t1。

## os.date()

按一定的格式将epoch值解析成日期时间的字符串格式或table格式。

```
os.date ([format [, time]])
```

不指定任何参数时，以系统设置的格式返回当前时间。

```lua
os.date()
--> Mon Apr 20 18:27:37 2019
```

**time参数**是待解析的epoch时间点，如果不给定参数time，则解析当前时间。

format参数指定将epoch解析成何种格式。format参数有一个可选的格式前缀"!"，例如`"!*t"`、`"!%F %T"`，它表示以UTC去解析epoch。

当format为`"*t"`时，表示将epoch转换为table，包含以下字段：  

```
year
month (1–12)
day (1–31)
hour (0–23)
min (0–59)
sec (0–61)
wday (weekday, 1–7, Sunday is 1)
yday (day of the year, 1–366),
isdst (daylight saving flag, a boolean)
```

如果format不是`"*t"`，则根据用户指定的格式返回对应的字符串格式。其格式和C语言支持的一样，可man strftime。

例如：

```lua
os.date("*t").month     --> 4

os.date("%F %T")        --> 2020-04-20 19:32:39
```

可以将`os.date("*t")`结合`os.time()`一起使用，例如计算3天前的时间：

```lua
cur = os.date("*t")
cur.day = cur.day - 3
print(os.time(cur))
```

































