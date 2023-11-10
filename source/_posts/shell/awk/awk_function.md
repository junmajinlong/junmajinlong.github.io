---
title: 精通awk系列(22)：awk 自定义函数(1)
p: shell/awk/awk_function.md
date: 2020-04-12 15:40:35
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# awk 自定义函数

可以定义一个函数将多个操作整合在一起。函数定义之后，可以到处多次调用，从而方便复用。

使用function关键字来定义函数：
```
function func_name([parameters]){
    function_body
}
```

对于gawk来说，也支持func关键字来定义函数。
```
func func_name(){}
```

函数可以定义在下面使用下划线的地方：
```
awk '_ BEGIN{} _ MAIN{} _ END{} _'
```

无论函数定义在哪里，都能在任何地方调用，因为awk在BEGIN之前，会先编译awk代码为内部格式，在这个阶段会将所有函数都预定义好。

例如：
```
awk '
    BEGIN{
        f()
        f()
        f()
    }
    function f(){
        print "星期一"
        print "星期二"
        print "星期三"
        print "星期四"
        print "星期五"
        print "星期六"
        print "星期日"
    }
'
```

## awk 函数的return语句

如果想要让函数有返回值，那么需要在函数中使用return语句。

return语句也可以用来立即结束函数的执行。

例如：
```
awk '
    function add(){
        return 40
    }
    BEGIN{
        print add()
        res = add() 
        print res
    }
'
```

如果不使用return或return没有参数，则返回值为空，即空字符串。
```
awk '
  function f1(){        }
  function f2(){return  }
  function f3(){return 3}
  BEGIN{
    print "-"f1()"-"
    print "-"f2()"-"
    print "-"f3()"-"
  }
'
```

## awk函数参数

为了让函数和调用者能够进行数据的交互，可以使用参数。

```
awk '
  function f(a,b){
    print a
    print b
    return a+b
  }
  BEGIN{
    x=10
    y=20
    res = f(x,y)
    print res
    print f(x,y)
  }
'
```

例如，实现一个重复某字符串指定次数的函数：
```
awk '
    function repeat(str,cnt  ,res_str){
        for(i=0;i<cnt;i++){
            res_str = res_str""str
        }
        return res_str
    }
    BEGIN{
        print repeat("abc",3)
        print repeat("-",30)
    }
'
```

调用函数时，实参数量可以比形参数量少，也可以比形参数量多。但是，在多于形参数量时会给出警告信息。

```
awk '
  function f(a,b){
    print a
    print b
    return a+b
  }
  BEGIN{
    x=10
    y=20
    
    print "---1----"
    print "-"f()"-"          # 不传递参数

    print "---2----"
    print "-"f(30)"-"        # 传递1个参数

    print "---3----"
    print "-"f(10,20,30)"-"  # 传递多个参数
  }
'
```

## awk函数参数数据类型冲突问题

如果函数内部使用参数的类型和函数外部变量的类型不一致，会出现数据类型不同而导致报错。

```
awk '
    function f(a){
        a[1]=30
    }
    BEGIN{
        a="hello world"
        f(a)   # 报错

        f(x)
        x=10   # 报错
    }
'
```

函数内部参数对应的是数组，那么外面对应的也必须是数组类型。

## awk参数按值传递还是按引用传递

在调用函数时，将数据作为函数参数传递给函数时，有两种传递方式：
- 传递普通变量时，是按值拷贝传递  
   - 直接拷贝普通变量的**值**到函数中  
   - 函数内部修改不会影响到外部  
- 传递数组时，是按引用传递  
   - 函数内部修改会影响到外部  

```
# 传递普通变量：按值拷贝
awk '
  function modify(a){
    a=30
    print a
  }
  BEGIN{
    a=40
    modify(a)
    print a
  }
'

# 传递数组：按引用拷贝
awk '
  function modify(a){
    a[1]=20
  }

  BEGIN{
    a[1]=10
    modify(a)
    print a[1]
  }
'
```
