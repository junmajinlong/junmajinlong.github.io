---
title: 精通awk系列(27)：awk预定义函数(2)：IO类预定义函数
p: shell/awk/awk_builtin_io_function.md
date: 2020-04-12 15:49:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# awk IO类内置函数

- `close(filename [, how])`：关闭文件或命令，参考[close](#close)  
- `system(command)`：执行Shell命令，参考[system](#system)  
- `fflush([filename])`：gawk会按块缓冲模式来缓冲输出结果，使用fflush()会将缓冲数据刷出  

从gawk 4.0.2之后的版本(不包括4.0.2)，无参数fflush()将刷出所有缓冲数据。

此外，终端设备是行缓冲模式，此时不需要fflush，而重定向到文件、到管道都是块缓冲模式，此时可能需要fflush()。

此外，system()在运行时也会flush gawk的缓冲。特别的，如果system的参数为空字符串`system("")`，则它不会去启动一个shell子进程而是仅仅执行flush操作。

没有flush时：
```
# 在终端输入，将不会显示，直到按下Ctrl + D
awk '{print "first";print "second"}' | cat
```

使用fflush()：
```
# 在终端输入
awk '{print "first";fflush();print "second"}' | cat
```

使用system()来flush：
```
awk '{print "first";system("echo system");print "second"}' | cat
awk '{print "first";system("");print "second"}' | cat
```

也可以使用`stdbuf -oL`命令来强制gawk按行缓冲而非默认的按块缓冲。
```
stdbuf -oL awk '{print "first";print "second"}' | cat
```

fflush()也可以指定文件名或命令，表示只刷出到该文件或该命令的缓冲数据。
```
# 刷出所有流向到标准输出的缓冲数据
awk '{print "first";fflush("/dev/stdout");print "second"}' | cat
```

最后注意，fflush()刷出缓冲数据不代表发送EOF标记。

# awk数据类型内置函数

- `isarray(var)`：测试var是否是数组，返回1(是数组)或0(不是数组)  
- `typeof(var)`：返回var的数据类型，有以下可能的值：  
   - "array"：是一个数组  
   - "regexp"：是一个真正表达式类型，强正则字面量才算是正则类型，如`@/a.*ef/`  
   - "number"：是一个number  
   - "string"：是一个string  
   - "strnum"：是一个strnum，参考[strnum类型](#strnum)  
   - "unassigned"：曾引用过，但未赋值，例如"print f;print typeof(f)"  
   - "untyped"：从未引用过，也从未赋值过  

例如，输出awk进程的内部信息，但跳过数组

```
awk '
  BEGIN{
    for(idx in PROCINFO){
      if(typeof(PROCINFO[idx]) == "array"){
        continue
      }
      print idx " -> "PROCINFO[idx]
    }
  }'
```

