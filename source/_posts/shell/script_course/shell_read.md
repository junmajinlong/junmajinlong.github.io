---
title: Shell脚本深入教程：Bash read命令读数据
p: shell/script_course/shell_read.md
date: 2020-05-19 14:29:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash read命令读数据

read命令是bash内置命令，用来从标准输入中读取数据。比如可以交互式读取用户在终端的输入，读取管道数据，读取标准输入重定向数据，等等。

读取文件中数据的方式：  
1. 按字符数读取  
2. 按分隔符读取  
3. 按行读取  
4. 一次性读完所有数据  
5. 按字节数读取(read命令不支持)  

语法：


```shell
read [-a aname] [-d delim] [-n nchars]
    [-N nchars] [-p prompt] [-t timeout] [-u fd] [name …]

-a arr
读取数据保存到数组arr中，arr数组已存在则先清空

-d delim
指定读取数据时的分隔符，不指定时默认为换行符。当指定为空字符串时，则默认为\0。当指定为文件中不存在的符号，则直接读整个文件

-n X
每次读取X个字符，但如果读满X字符前遇到了分隔符，则也停止本次读取

-N X
每次读取X个字符，即使遇到了分隔符也不停止

-p prompt
交互式提示用户在终端输入的信息，默认不输出换行符

-s
用户在终端中的输入不会显示出来。在提示用户输入密码时应设置该选项

-r
使得读取到的反斜线不进行转义，而是作为一个普通反斜线字符 

-t timeout
超时时间，如果在超时时间内还未读满数据，则读取失败，可指定为小数时间

-u fd
从文件描述符中读取数据
```

## 示例

示例文件a.txt：

```shell
$ cat a.txt
portmapper abcd
status efgh
mountd ijklmn
nfs  opg rst
nfs_acl uvw
nlockmgr xyz
```

1.按字符数读取

```shell
while read -n 1 char;do echo $char;done <a.txt
```

2.按分隔符读取

```shell
while read -d "m" chars;do echo "$chars";done <a.txt
```

![](/img/shell/1589873694971.png)

5.读一行，并按IFS划分本行，将划分后的各个字段赋值给多个变量

```shell
while read first second;do echo $first, $second;done < a.txt
```

6.读一行，并按IFS划分本行，将划分后的各个字段保存在数组变量中

```shell
 while read -a arr;do echo ${arr[0]}, ${arr[1]};done < a.txt 
```

7.交互式提示用户输入姓名、年龄

```shell
read -p "please give me your username and age:\n" user age
echo $user, $age

read -p $'please give me your username and age:\n' user age
echo $user, $age
```

8.从文件描述符中读取数据

```shell
exec 3<> a.txt
while read -u 3 line;do echo $line;done
```

9.不指定保存所读数据的变量时，则默认保存在REPLY变量中

```shell
read
echo $REPLY
```

10.使用read的-t超时时间，模拟一个支持小数的sleep命令

```bash
read_sleep() {
    # Usage: read_sleep 1
    #        read_sleep 0.2
    read -rt "$1" <> <(:) || :
}

$ read_sleep 0.5
$ read_sleep 1
```

## 使用read的优点

使用read命令，可以让Shell脚本实现比管道更简洁、更丰富的数据处理逻辑。比如：

```shell
# 使用管道，每个管道一次处理
cat /etc/fstab | grep 'UUID'

while read line;do
  echo $line | grep 'UUDI'
  echo $line >>/tmp/fstab
  echo $line | sed -r 's/UUID/uuid/' >>/tmp/fstab
  #...
done < /etc/fstab
```

## 使用read时的注意事项

注意事项1.不建议在管道中使用`while read line`，因为管道会让命令在子Shell中执行，使得while中定义的变量退出while就失效了

```shell
cat /etc/fstab | while read line;do
  let num=num+1
  echo $num: $line
done
echo $num
```

取而代之的是使用标准输入重定向或进程替换。

```shell
# 标准输入重定向
while read line;do
  let num=num+1
  echo $num: $line
done < /etc/fstab

# 进程替换
while read line;do
  let num=num+1
  echo $num: $line
done < <(cat /etc/fstab)
```

注意事项2.读取内容中包含特殊符号时、包含前缀后缀空白时，要记得：
- (1).修改IFS变量：防止忽略前缀和后缀空白  
- (2).read加-r选项：防止所读取内容中的反斜线执行转义行为  
- (3).引用保存数据的变量时，加双引号保护，防止变量替换后特殊符号被解析

例如，如下示例文件a.txt内容：

```
abc def
ghi jkl \*txt
  mno pqr
```

不安全读取和安全读取：

```shell
# 不安全读取，输出的数据和原始数据不同
while read line;do
  echo $line
done <a.txt

# 安全读取，原样输出所有数据
while IFS= read -r line;do
  echo "$line"
done <a.txt
```

注意事项3：while read line的时候：

为什么不用

```shell
while read line <a.txt;do
 echo $line
done
```

而要用

```shell
while read line;do
  echo $line
done <a.txt
```

因为前者会在每轮循环时都打开文件a.txt，使得每轮循环都只读取第一行内容，且无限循环。
