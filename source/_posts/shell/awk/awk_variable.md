---
title: 精通awk系列(14)：细说awk中的变量和变量赋值
p: shell/awk/awk_variable.md
date: 2019-12-07 10:37:31
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# awk变量

awk的变量是动态变量，在使用时声明。

所以awk变量有3种状态： 
- 未声明状态：称为untyped类型  
- 引用过但未赋值状态：unassigned类型  
- 已赋值状态  

**引用未赋值的变量，其默认初始值为空字符串或数值0**。

在awk中未声明的变量称为untyped，声明了但未赋值(只要引用了就声明了)的变量其类型为unassigned。

gawk 4.2版提供了`typeof()`函数，可以测试变量的数据类型，包括测试变量是否声明。
```shell
awk 'BEGIN{
  print(typeof(a))            # untyped
  if(b==0){print(typeof(b))}  # unassigned
}'
```

除了typeof()，还可以使用下面的技巧进行检测：

```shell
awk 'BEGIN{
  if(a=="" && a==0){    # 未赋值时，两个都true
    print "untyped or unassigned"
  } else {
      print "assigned"
  }
}'
```

## 变量赋值

awk中的变量赋值语句也可以看作是一个**有返回值**的表达式。

例如，`a=3`赋值完成后返回3，同时变量a也被设置为3。

基于这个特点，有两点用法：  
- 可以`x=y=z=5`，等价于`z=5 y=5 x=5`  
- 可以将赋值语句放在任意允许使用表达式的地方  
   - `x != (y = 1)`  
   - `awk 'BEGIN{print (a=4);print a}'`  

问题：`a=1;arr[a+=2] = (a=a+6)`是怎么赋值的，对应元素结果等于？`arr[3]=7`。但不要这么做，因为不同awk的赋值语句左右两边的评估顺序有可能不同。

## awk中声明变量的位置

1. 在BEGIN或main或END代码段中直接引用或赋值  
2. 使用`-v var=val`选项，可定义多个，必须放在awk代码的前面  
   - 它的变量声明早于BEGIN块  
   - 普通变量：`awk -v age=123 'BEGIN{print age}'`  
   - 使用shell变量赋值：`awk -v age=$age 'BEGIN{print age}'`  
3. 在awk代码后面使用`var=val`参数  
   - 它的变量声明在BEGIN之后  
   - `awk '{print n}' n=3 a.txt n=4 b.txt`  
   - `awk '{print $1}' FS=' ' a.txt FS=":" /etc/passwd`  
   - 使用Shell变量赋值：`awk '{print age}' age=$age a.txt`

## awk中使用Shell变量

要在awk中使用Shell变量，有三种方式：

**1.在-v选项中将Shell变量赋值给awk变量**  

```shell
num=$(cat a.txt | wc -l)
awk -v n=$num 'BEGIN{print n}'
```

-v选项是在awk工作流程的第一阶段解析的，所以-v选项声明的变量在BEGIN{}、END{}和main代码段中都能直接使用。

![](/img/shell/awk/733013-20191208135442579-438043898.jpg)


**3.直接在awk代码部分暴露Shell变量，交给Shell解析进行Shell的变量替换**

```shell
num=$(cat a.txt | wc -l)
awk 'BEGIN{print '"$num"'}'
```

这种方式最灵活，但可读性最差，可能会出现大量的引号。


