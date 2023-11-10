---
title: 精通awk系列(13)：print、printf、sprintf和重定向
p: shell/awk/awk_print.md
date: 2019-12-07 10:37:30
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# 输出操作

awk可以通过print、printf将数据输出到标准输出或重定向到文件。

## print

```
print elem1,elem2,elem3...
print(elem1,elem2,elem3...)
```

逗号分隔要打印的字段列表，各字段都**会自动转换成字符串格式**，然后通过预定义变量OFS(output field separator)的值(其默认值为空格)连接各字段进行输出。

```
$ awk 'BEGIN{print "hello","world"}'
hello world
$ awk 'BEGIN{OFS="-";print "hello","world"}'
hello-world
```

print要输出的数据称为输出记录，在print输出时会自动在尾部加上输出记录分隔符，输出记录分隔符的预定义变量为ORS，其默认值为`\n`。

```
$ awk 'BEGIN{OFS="-";ORS="_\n";print "hello","world"}'
hello-world_
```

括号可省略，但如果要打印的元素中包含了特殊符号`>`，则必须使用括号包围(如`print("a" > "A")`)，因为它是输出重定向符号。

如果省略参数，即`print;`等价于`print $0;`。

### print输出数值

print在输出数据时，总是会先转换成字符串再输出。

对于数值而言，可以自定义转换成字符串的格式，例如使用sprintf()进行格式化。

print在自动转换数值（专指小数）为字符串的时候，采用预定义变量OFMT（Output format）定义的格式按照sprintf()相同的方式进行格式化。OFMT默认值为`%.6g`，表示有效位(整数部分加小数部分)最多为6。

```
$ awk 'BEGIN{print 3.12432623}'
3.12433
```

可以修改OFMT，来自定义数值转换为字符串时的格式：

```
$ awk 'BEGIN{OFMT="%.2f";print 3.99989}'
4.00

# 格式化为整数
$ awk 'BEGIN{OFMT="%d";print 3.99989}' 
3
$ awk 'BEGIN{OFMT="%.0f";print 3.99989}' 
4
```

## printf

```
printf format, item1, item2, ...
```

格式化字符：

![](/img/shell/awk/733013-20191208134716566-293209689.jpg)


修饰符：均放在格式化字符的前面

```
N$      N是正整数。默认情况下，printf的字段列表顺序和格式化字符
        串中的%号顺序是一一对应的，使用N$可以自行指定顺序。
        printf "%2$s %1$s","world","hello"输出hello world
        N$可以重复指定，例如"%1$s %1$s"将取两次第一个字段

宽度     指定该字段占用的字符数量，不足宽度默认使用空格填充，超出宽度将无视。
         printf "%5s","ni"输出"___ni"，下划线表示空格

-       表示左对齐。默认是右对齐的。
        printf "%5s","ni"输出"___ni"
        printf "%-5s","ni"输出"ni___"

空格     针对于数值。对于正数，在其前添加一个空格，对于负数，无视
        printf "% d,% d",3,-2输出"_3,-2"，下划线表示空格

+       针对于数值。对于正数，在其前添加一个+号，对于负数，无视
        printf "%+d,%+d",3,-2输出"+3,-2"，下划线表示空格

#       可变的数值前缀。对于%o，将添加前缀0，对于%x或%X，将添加前缀0x或0X

0       只对数值有效。使用0而非默认的空格填充在左边，对于左对齐的数值无效
        printf "%05d","3"输出00003
        printf "%-05d","3"输出3
        printf "%05s",3输出____3

'       单引号，表示对数值加上千分位逗号，只对支持千分位表示的locale有效
        $ awk "BEGIN{printf \"%'d\n\",123457890}"
        123,457,890
        $ LC_ALL=C awk "BEGIN{printf \"%'d\n\",123457890}"
        123457890

.prec   指定精度。在不同格式化字符下，精度含义不同
        %d,%i,%o,%u,%x,%X 的精度表示最大数字字符数量
        %e,%E,%f,%F 的精度表示小数点后几位数
        %s 的精度表示最长字符数量，printf "%.3s","foob"输出foo
        %g,%G 的精度表示表示最大有效位数，即整数加小数位的总数量
```

<a name="sprintf"></a>

## sprintf()

sprintf()采用和printf相同的方式格式化字符串，但是它不会输出格式化后的字符串，而是返回格式化后的字符串。所以，可以将格式化后的字符串赋值给某个变量。

```
awk '
    BEGIN{
        a = sprintf("%03d", 12.34)
        print a  # 012
    }
'
```

## 重定向输出

```
print[f] something >"filename"
print[f] something >>"filename"
print[f] something | "Shell_Cmd"
print[f] something |& "Shell_Cmd_Coprocess"
```

`>filename`时，如果文件不存在，则创建，如果文件存在则首先截断。之后再输出到该文件时将不再截断。

> awk中只要不close()，任何文件都只会在第一次使用时打开，之后都不会再重新打开。

```
awk '{print $2 >"name.txt";print $4 >"name.txt"}' a.txt
```

`>>filename`时，将追加数据，文件不存在时则创建。

`print[f] something | Shell_Cmd`时，awk将创建一个管道，然后启动Shell命令，print[f]产生的数据放入管道，而命令将从管道中读取数据。

```
# 例1：
awk '
    NR>1{
      print $2 >"name.unsort"
      cmd = "sort >name.sort"
      print $2 | cmd
      #print $2 | "sort >name.sort"
    }
    END{close(cmd)}
' a.txt

# 例2：awk中构建Shell命令，通过管道交给shell执行
awk 'BEGIN{printf "seq 1 5" | "bash"}'
```

`print[f] something |& Shell_Cmd`时，print[f]产生的数据交给Coprocess。之后，awk再从Coprocess中取回数据。这里的`|&`有点类似于能够让Shell_Cmd后台异步运行的管道。

## stdin、stdout、stderr

awk重定向时可以直接使用`/dev/stdin`、`/dev/stdout`和`/dev/stderr`。还可以直接使用某个已打开的文件描述符`/dev/fd/N`。

例如：

```
awk 'BEGIN{print "something OK" > "/dev/stdout"}'
awk 'BEGIN{print "something wrong" > "/dev/stderr"}'
awk 'BEGIN{print "something wrong" | "cat >&2"}'

awk 'BEGIN{getline < "/dev/stdin";print $0}'

$ exec 4<> a.txt
$ awk 'BEGIN{while((getline < "/dev/fd/4")>0){print $0}}'
```




