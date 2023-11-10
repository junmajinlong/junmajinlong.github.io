---
title: 精通awk系列(12)：awk getline用法详解
p: shell/awk/awk_getline.md
date: 2019-11-23 10:37:40
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# getline用法详解

除了可以从标准输入或非选项型参数所指定的文件中读取数据，还可以使用getline从其它各种渠道获取需要处理的数据，它的用法有很多种。

getline的返回值：  
- 如果可以读取到数据，返回1  
- 如果遇到了EOF，返回0  
- 如果遇到了错误，返回负数。如-1表示文件无法打开，-2表示IO操作需要重试(retry)。在遇到错误的同时，还会设置`ERRNO`变量来描述错误  

为了健壮性，getline时强烈建议进行判断。例如：

![](/img/shell/awk/733013-20191123160751621-118764535.jpg)

上面的getline的括号尽量加上，因为`getline < 0`表示的是输入重定向，而不是和数值0进行小于号的比较。

## 无参数的getline

getline无参数时，表示从当前正在处理的文件中立即读取下一条记录保存到`$0`中，并进行字段分割，然后**继续执行后续代码逻辑**。

此时的getline会设置NF、RT、NR、FNR、$0和$N。

next也可以读取下一行。

- getline：读取下一行之后，继续执行getline后面的代码   

- next：读取下一行，立即回头awk循环的头部，不会再执行next后面的代码  

它们之间的区别用伪代码描述，类似于：

```
# next
exec 9<> filename
while read -u 9 line;do
  ...code...
  continue  # next
  ...code...  # 这部分代码在本轮循环当中不再执行
done

# getline
while read -u 9 line;do
  ...code...
  read -u 9 line  # getline
  ...code...
done
```

例如，匹配到某行之后，再读一行就退出：

```
awk '/^1/{print;getline;print;exit}' a.txt
```

为了更健壮，应当对getline的返回值进行判断。

```
awk '/^1/{print;if((getline)<=0){exit};print}' a.txt
```

## 一个参数的getline

没有参数的getline是读取下一条记录之后将记录保存到`$0`中，并对该记录进行字段的分割。

一个参数的getline是将读取的记录保存到指定的变量当中，并且不会对其进行分割。

```
getline var
```

此时的getline只会设置RT、NR、FNR变量和指定的变量var。因此$0和$N以及NF保持不变。

```
awk '
/^1/{
  if((getline var)<=0){exit}
  print var
  print $0"--"$2
}' a.txt
```

## 从指定文件中读取数据

![](/img/shell/awk/733013-20191123160927415-367963805.jpg)

filename需使用双引号包围表示文件名字符串，否则会当作变量解析`getline < "c.txt"`。此外，如果路径是使用变量构建的，则应该使用括号包围路径部分。例如`getline < dir "/" filename`中使用了两个变量构建路径，这会产生歧义，应当写成`getline <(dir "/" filename)`。

注意，每次从filename读取之后都会做好位置偏移标记，下次再从该文件读取时将根据这个位置标记继续向后读取。

例如，每次行首以1开头时就读取c.txt文件的所有行。

```
awk '
  /^1/{
    print;
    while((getline < "c.txt")>0){print};
    close("c.txt")
}' a.txt
```

上面的`close("c.txt")`表示在`while(getline)`读取完文件之后关掉，以便后面再次读取，如果不关掉，则文件偏移指针将一直在文件结尾处，使得下次读取时直接遇到EOF。

## 从Shell命令输出结果中读取数据

- `cmd | getline`：从Shell命令cmd的输出结果中读取一条记录保存到`$0`中  
   - 会进行字段划分，设置变量`$0 NF $N RT`，不会修改变量`NR FNR`  
- `cmd | getline var`:从Shell命令cmd的输出结果中读取数据保存到var中  
   - 除了var和RT，其它变量都不会设置  

如果要再次执行cmd并读取其输出数据，则需要close关闭该命令。例如`close("seq 1 5")`，参见下面的示例。

例如：每次遇到以1开头的行都输出seq命令产生的`1 2 3 4 5`。

```
awk '/^1/{print;while(("seq 1 5"|getline)>0){print};close("seq 1 5")}' a.txt
```

再例如，调用Shell的date命令生成时间，然后保存到awk变量cur_date中：

```
awk '
  /^1/{
    print
    "date +\"%F %T\""|getline cur_date
    print cur_date
    close("date +\"%F %T\"")
}' a.txt
```

可以将cmd保存成一个字符串变量。

```
awk '
  BEGIN{get_date="date +\"%F %T\""}
  /^1/{
    print
    get_date | getline cur_date
    print cur_date
    close(get_date)
}' a.txt
```

更为复杂一点的，cmd中可以包含Shell的其它特殊字符，例如管道、重定向符号等：

```
awk '
  /^1/{
    print
    if(("seq 1 5 | xargs -i echo x{}y 2>/dev/null"|getline) > 0){
      print
    }
    close("seq 1 5 | xargs -i echo x{}y 2>/dev/null")
}' a.txt
```

## awk中的coprocess

awk虽然强大，但是有些数据仍然不方便处理，这时可将数据交给Shell命令去帮助处理，然后再从Shell命令的执行结果中取回处理后的数据继续awk处理。

awk通过`|&`符号来支持coproc。

```
awk_print[f] "something" |& Shell_Cmd
Shell_Cmd |& getline [var]
```

这表示awk通过print输出的数据将传递给Shell的命令Shell_Cmd去执行，然后awk再从Shell_Cmd的执行结果中取回Shell_Cmd产生的数据。

例如，不想使用awk的substr()来取子串，而是使用sed命令来替换。

```
awk '
    BEGIN{
      CMD="sed -nr \"s/.*@(.*)$/\\1/p\"";
    }

    NR>1{
        print $5;
        print $5 |& CMD;
        close(CMD,"to");
        CMD |& getline email_domain;
        close(CMD);
        print email_domain;
}' a.txt
```

对于`awk_print |& cmd; cmd |& getline`的使用，须注意的是：  
- `awk_print |& cmd`会直接将数据写进管道，cmd可以从中获取数据  
- 强烈建议在awk_print写完数据之后加上`close(cmd,"to")`，这样表示向管道中写入一个EOF标记，避免某些要求读完所有数据再执行的cmd命令被永久阻塞  
- 如果cmd是按块缓冲的，则getline可能会陷入阻塞。这时可将cmd部分改写成`stdbuf -oL cmd`以强制其按行缓冲输出数据  
   - `CMD="stdbuf -oL cmdline";awk_print |& CMD;close(CMD,"to");CMD |& getline`  

对于那些要求读完所有数据再执行的命令，例如sort命令，它们有可能需要等待数据已经完成后（遇到EOF标记）才开始执行任务，对于这些命令，可以多次向coprocess中写入数据，最后`close(CMD,"to")`让coprocess运行起来。

例如，对age字段(即`$4`)使用sort命令按数值大小进行排序：

```
awk '
    BEGIN{
      CMD="sort -k4n";
    }

    # 将所有行都写进管道
    NR>1{
      print $0 |& CMD;
    }

    END{
      close(CMD,"to");  # 关闭管道通知sort开始排序
      while((CMD |& getline)>0){
        print;
      }
      close(CMD);
} ' a.txt
```

<a name="close"></a>

## close()

```
close(filename)
close(cmd,[from | to])  # to参数只用于coprocess的第一个阶段
```

如果close()关闭的对象不存在，awk不会报错，仅仅只是让其返回一个负数返回值。

close()有两个基本作用：   

![](/img/shell/awk/733013-20191123161109842-1493772437.jpg)

awk中任何文件都只会在第一次使用时打开，之后都不会再重新打开。只有关闭之后，再使用才会重新打开。

例如一个需求是只要在a.txt中匹配到1开头的行就输出另一个文件x.log的所有内容，那么在第一次输出x.log文件内容之后，文件偏移指针将在x.log文件的结尾处，如果不关闭该文件，则后续所有读取x.log的文件操作都从结尾处继续读取，但是显然总是得到EOF异常，所以getline返回值为0，而且也读取不到任何数据。所以，必须关闭它才能在下次匹配成功时再次从头读取该文件。

```
awk '
  /^1/{
    print;
    while((getline var <"x.log")>0){
      print var
    }
    close("x.log")
}' a.txt
```

在处理Coprocess的时候，close()可以指定第二个参数"from"或"to"，它们都针对于coproc而言，from时表示关闭`coproc |& getline`的管道，使用to时，表示关闭`print something |& coproc`的管道。

```
awk '
BEGIN{
  CMD="sed -nr \"s/.*@(.*)$/\\1/p\"";
}
NR>1{
    print $5;
    print $5 |& CMD;
    close(CMD,"to");   # 本次close()是必须的
    CMD |& getline email_domain;
    close(CMD);
    print email_domain;
}' a.txt
```

上面的第一个close是必须的，否则sed会一直阻塞。因为sed一直认为还有数据可读，只有关闭管道发送一个EOF，sed才会开始处理。
