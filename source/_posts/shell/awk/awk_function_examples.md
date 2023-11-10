---
title: 精通awk系列(24)：awk 自定义函数(3)：awk自定义函数示例
p: shell/awk/awk_function_examples.md
date: 2020-04-12 15:43:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# 自定义函数示例

## 1.一次性读取一个文件所有数据

```
function readfile(file    ,rs_bak,data){
  rs_bak=RS
  RS="^$"
  if ( (getline data < file) < 0 ){
    print "read file failed"
    exit 1
  }
  close(file)
  RS=rs_bak
  return data
}


/^1/{
  print $0
  content = readfile("c.txt")
  print content
}
```

将RS设置为`^$`是永远不可能出现的分隔符，除非这个文件为空文件。

## 2.重读文件

实现一个rewind()功能来重置文件偏移指针，从而模拟实现重读当前文件。

```
function rewind(    i){
    # 将当前正在读取的文件添加到ARGV中当前文件的下一个元素
    for(i=ARGC;i>ARCIND;i--){
        ARGV[i] = ARGV[i-1]
    }

    # 随着增加ARGC，以便awk能够读取到因ARGV增加元素后的最后一个文件
    ARGC++

    # 直接进入下一个文件
    nextfile
}
```

要注意可能出现无限递归的场景：
```
awk -f rewind.awk 'NR==3{rewind()}{print FILENAME, FNR, $0}' a.txt

# 下面这个会无限递归，因为FNR==3很可能每次重读时都会为真
awk -f rewind.awk 'FNR==3{rewind()}{print FILENAME, FNR, $0}' a.txt
```

## 3.格式化数组的输出

实现一个a2s()函数。

```
BEGIN{
  arr["zhangsan"]=21
  arr["lisi"]=22
  arr["wangwu"]=23
  print a2s(arr)
}

function a2s(arr       ,content,i,cnt){
  for(i in arr){
    if(cnt){
      content=content""(sprintf("\t%s:%s\n",i,arr[i]))
    } else {
      content=content""(sprintf("\n\t%s:%s\n",i,arr[i]))
    }
    cnt++
  }
  return "{"content"}"
}
```

## 4.禁用命令行尾部的赋值语句

`awk '{}' ./a=b a.txt`中，`a=b`会被awk识别为变量赋值操作。但是，如果用户想要处理的正好是包含了等号的文件名，则应当去禁用该赋值操作。

禁用的方式很简单，只需为其加上一个路径前缀`./`即可。

为了方便控制，可通过`-v`设置一个flag类型的选项标记。
```
function disable_assigns(argc,argv,    i){
    for(i=1;i<argc;i++){
        if(argv[i] ~ /[[:alpha:]_][[:alnum:]_]*=.*/){
            argv[i] = ("./"argv[i])
        }
    }
}

BEGIN{
    if(assign_flag){
        disable_assigns(ARGC,ARGV)
    }
}
```
那么，调用awk时采用如下方式：
```
awk -v assign_flag=1 -f assigns.awk '{print}' a=b.txt a.txt
```