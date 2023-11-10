---
title: 精通awk系列(20)：awk数组用法详解
p: shell/awk/awk_array.md
date: 2020-02-27 10:40:35
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# 数组

awk数组特性：  
- awk的数组是关联数组(即key/value方式的hash数据结构)，索引下标可为数值(甚至是负数、小数等)，也可为字符串  
   - 在内部，awk数组的索引全都是字符串，即使是数值索引在使用时内部也会转换成字符串  
   - awk的数组元素的顺序和元素插入时的顺序很可能是不相同的  
- awk数组支持数组的数组  

## awk访问、赋值数组元素

```
arr[idx]
arr[idx] = value
```

索引可以是整数、负数、0、小数、字符串。如果是数值索引，会按照CONVFMT变量指定的格式先转换成字符串。

例如：
```
awk '
  BEGIN{
    arr[1]   = 11
    arr["1"] = 111
    arr["a"] = "aa"
    arr[-1]  = -11
    arr[4.3] = 4.33
# 本文来自骏马金龙：www.junmajinlong.com
    print arr[1]     # 111
    print arr["1"]   # 111
    print arr["a"]   # aa
    print arr[-1]    # -11
    print arr[4.3]   # 4.33
  }
'
```

**通过索引的方式访问数组中不存在的元素时，会返回空字符串，同时会创建这个元素并将其值设置为空字符串**。

```
awk '
  BEGIN{
    arr[-1]=3;
    print length(arr);  # 1
    print arr[1];
    print length(arr)   # 2
  }'
```

## awk数组长度

awk提供了`length()`函数来获取数组的元素个数，它也可以用于获取字符串的字符数量。还可以获取数值转换成字符串后的字符数量。

```
awk 'BEGIN{arr[1]=1;arr[2]=2;print length(arr);print length("hello")}'
```

## awk删除数组元素

- `delete arr[idx]`：删除数组`arr[idx]`元素  
   - 删除不存在的元素不会报错  
- `delete arr`：删除数组所有元素  

```
$ awk 'BEGIN{arr[1]=1;arr[2]=2;arr[3]=3;delete arr[2];print length(arr)}'
2
```

## awk检测是否是数组

`isarray(arr)`可用于检测arr是否是数组，如果是数组则返回1，否则返回0。

`typeof(arr)`可返回数据类型，如果arr是数组，则其返回"array"。

```
awk 'BEGIN{
    arr[1]=1;
    print isarray(arr);
    print (typeof(arr) == "array")
}'
```

## awk测试元素是否在数组中

不要使用下面的方式来测试元素是否在数组中：

```
if(arr["x"] != ""){...}
```

这有两个问题：  
- 如果不存在arr["x"]，则会立即创建该元素，并将其值设置为空字符串  
- 有些元素的值本身就是空字符串  

应当使用数组成员测试操作符in来测试：

```
# 注意，idx不要使用index，它是一个内置函数
if (idx in arr){...}
```

它会测试索引idx是否在数组中，如果存在则返回1，不存在则返回0。

```
awk '
    BEGIN{
    # 本文来自骏马金龙：www.junmajinlong.com
        arr[1]=1;
        arr[2]=2;
        arr[3]=3;

        arr[1]="";
        delete arr[2];

        print (1 in arr);  # 1
        print (2 in arr);  # 0
    }'
```

## awk遍历数组

awk提供了一种for变体来遍历数组：
```
for(idx in arr){print arr[idx]}
```

因为awk数组是关联数组，元素是不连续的，也就是说没有顺序。遍历awk数组时，顺序是不可预测的。

例如：

```
# 本文来自骏马金龙：www.junmajinlong.com
awk '
    BEGIN{
        arr["one"] = 1
        arr["two"] = 2
        arr["three"] = 3
        arr["four"] = 4
        arr["five"] = 5

        for(i in arr){
            print i " -> " arr[i]
        }
    }
'
```

此外，不要随意使用`for(i=0;i<length(arr);i++)`来遍历数组，因为awk数组是关联数组。但如果已经明确知道数组的所有元素索引都位于某个数值范围内，则可以使用该方式进行遍历。

例如：
```
# 本文来自骏马金龙：www.junmajinlong.com
awk '
    BEGIN{
        arr[1] = "one"
        arr[2] = "two"
        arr[3] = "three"
        arr[4] = "four"
        arr[5] = "five"
        arr[10]= "ten"

        for(i=0;i<=10;i++){
            if(i in arr){
                print arr[i]
            }
        }
    }
'
```

<a name="SUBSEP"></a>

## awk复杂索引的数组

在awk中，很多时候单纯的一个数组只能存放两个信息：一个索引、一个值。但在一些场景下，这样简单的存储能力在处理复杂需求的时候可能会捉襟见肘。

为了存储更多信息，方式之一是将第3份、第4份等信息全部以特殊方式存放到值中，但是这样的方式在实际使用过程中并不方便，每次都需要去分割值从而取出各部分的值。

另一种方式是将第3份、第4份等信息存放在索引中，将多份数据组成一个整体构成一个索引。

gawk中提供了将多份数据信息组合成一个整体当作一个索引的功能。默认方式为`arr[x,y]`，其中x和y是要结合起来构建成一个索引的两部分数据信息。逗号称为下标分隔符，在构建索引时会根据预定义变量SUBSEP的值将多个索引组合起来。所以`arr[x,y]`其实完全等价于`arr[x SUBSEP y]`。

例如，如果SUBSEP设置为『@』，那么`arr[5,12] = 512`存储时，其真实索引为`5@12`，所以要访问该元素需使用`arr["5@12"]`。

SUBSEP的默认值为`\034`，它是一个不可打印的字符，几乎不可能会出现在字符串当中。

如果我们愿意的话，我们也可以自己将多份数据组合起来去构建成一个索引，例如`arr[x" "y]`。但是awk提供了这种更为简便的方式，直接用即可。

为了测试这种复杂数组的索引是否在数组中，可以使用如下方式：
```
arr["a","b"] = 12
if (("a", "b") in arr){...}
```

例如，顺时针倒转下列数据：
```
1 2 3 4 5 6
2 3 4 5 6 1
3 4 5 6 1 2
4 5 6 1 2 3

结果：
4 3 2 1
5 4 3 2
6 5 4 3
1 6 5 4
2 1 6 5
3 2 1 6
```

```
{
  nf = NF
  nr = NR
  for(i=1;i<=NF;i++){
    arr[NR,i] = $i
  }
}

END{
  for(i=1;i<=nf;i++){
    for(j=nr;j>=1;j--){
      if(j%nr == 1){
        printf "%s\n", arr[j,i]
      }else {
        printf "%s ", arr[j,i]
      }
    }
  }
}
```


## awk子数组

子数组是指数组中的元素也是一个数组，即Array of Array，它也称为子数组(subarray)。

awk也支持子数组，在效果上即是嵌套数组或多维数组。

```
a[1][1] = 11
a[1][2] = 12
a[1][3] = 13
a[2][1] = 21
a[2][2] = 22
a[2][3] = 23
a[2][4][1] = 241
a[2][4][2] = 242
a[2][4][1] = 241
a[2][4][3] = 243
```

通过如下方式遍历二维数组：
```
for(i in a){
    for (j in a[i]){
        if(isarray(a[i][j])){
            continue
        }
        print a[i][j]
    }
}
```

一个经典的子数组的案例，参见[统计多项数据](/shell/awk/awk_examples#subarray)

<a name="arrsort"></a>

## awk指定数组遍历顺序

由于awk数组是关联数组，默认情况下，`for(idx in arr)`遍历数组时顺序是不可预测的。

但是gawk提供了`PROCINFO["sorted_in"]`来指定遍历的元素顺序。它可以设置为两种类型的值：  
- 设置为用户自定义函数  
- 设置为下面这些awk预定义好的值：  
   - `@unsorted`：默认值，遍历时无序  
   - `@ind_str_asc`：索引按字符串比较方式升序遍历  
   - `@ind_str_desc`：索引按字符串比较方式降序遍历  
   - `@ind_num_asc`：索引强制按照数值比较方式升序遍历。所以无法转换为数值的字符串索引将当作数值0进行比较  
   - `@ind_num_desc`：索引强制按照数值比较方式降序遍历。所以无法转换为数值的字符串索引将当作数值0进行比较  
   - `@val_type_asc`：按值升序比较，此外数值类型出现在前面，接着是字符串类型，最后是数组类(即认为`num<str<arr`)  
   - `@val_type_desc`：按值降序比较，此外数组类型出现在前面，接着是字符串类型，最后是数值型(即认为`num<str<arr`)  
   - `@val_str_asc`：按值升序比较，数值转换成字符串再比较，而数组出现在尾部(即认`str<arr`)  
   - `@val_str_desc`：按值降序比较，数值转换成字符串再比较，而数组出现在头部(即认`str<arr`)  
   - `@val_num_asc`：按值升序比较，字符串转换成数值再比较，而数组出现在尾部(即认`num<arr`)  
   - `@val_num_desc`：按值降序比较，字符串转换成数值再比较，而数组出现在头部(即认为`num<arr`)  


例如：
```
awk '
  BEGIN{
    arr[1] = "one"
    arr[2] = "two"
    arr[3] = "three"
    arr["a"] ="aa"
    arr["b"] ="bb"
    arr[10]= "ten"

    #PROCINFO["sorted_in"] = "@ind_num_asc"
    #PROCINFO["sorted_in"] = "@ind_str_asc"
    PROCINFO["sorted_in"] = "@val_str_asc"
    for(idx in arr){
      print idx " -> " arr[idx]
    }
}'

a -> aa
b -> bb
1 -> one
2 -> two
3 -> three
10 -> ten

# 本文来自骏马金龙：www.junmajinlong.com
```

如果指定为用户自定义的排序函数，其函数格式为：
```
function sort_func(i1,v1,i2,v2){
    ...
    return <0;0;>0
}
```

其中，i1和i2是每次所取两个元素的索引，v1和v2是这两个索引的对应值。

如果返回值小于0，则表示i1在i2前面，i1先被遍历。如果等于0，则表示i1和i2具有等值关系，它们的遍历顺序不可保证。如果大于0，则表示i2先于i1被遍历。

例如，对数组元素按数值大小比较来决定遍历顺序。
```
awk '
function cmp_val_num(i1, v1, i2, v2){
  if ((v1 - v2) < 0) {
    return -1
  } else if ((v1 - v2) == 0) {
    return 0
  } else {
    return 1
  }
  # return (v1-v2)
}

NR > 1 {
  arr[$0] = $4
}

END {
  PROCINFO["sorted_in"] = "cmp_val_num"
  for (i in arr) {
    print i
  }
}' a.txt
```

再比如，按数组元素值的字符大小来比较。
```
function cmp_val_str(i1,v1,i2,v2) {
    v1 = v1 ""
    v2 = v2 ""
    if(v1 < v2){
        return -1
    } else if(v1 == v2){
        return 0
    } else {
        return 1
    }
    # return (v1 < v2) ? -1 : (v1 != v2)
}

NR>1{
    arr[$0] = $2
}

END{
    PROCINFO["sorted_in"] = "cmp_val_str"
    for(line in arr)
    {
        print line
    }
}
```

再比如，对元素值按数值升序比较，且相等时再按第一个字段ID进行数值降序比较。
```
awk '
function cmp_val_num(i1,v1,i2,v2,    a1,a2) {
    if (v1<v2) {
        return - 1
    } else if(v1 == v2){
        split(i1, a1, SUBSEP)
        split(i2, a2, SUBSEP)
        return a2[2] - a1[2]
    } else {
        return 1
    }
}

NR>1{
    arr[$0,$1] = $4
}

END{
    PROCINFO["sorted_in"] = "cmp_val_num"
    for(str in arr){
        split(str, a, SUBSEP)
        print a[1]
    }
}

' a.txt
```

上面使用的`arr[x,y]`来存储额外信息，下面使用`arr[x][y]`多维数组的方式来存储额外信息实现同样的排序功能。

```
NR>1{
  arr[NR][$0] = $4
}

END{
  PROCINFO["sorted_in"] = "cmp_val_num"
  for(nr in arr){
    for(line in arr[nr]){
      print line
    }
  # 本文来自骏马金龙：www.junmajinlong.com
  }
}

function cmp_val_num(i1,v1,i2,v2,   ii1,ii2){
  # 获取v1/v2的索引，即$0的值
  for(ii1 in v1){ }
  for(ii2 in v2){ }

  if(v1[ii1] < v2[ii2]){
    return -1
  }else if(v1[ii1] > v2[ii2]){
    return 1
  }else{
    return (i2 - i1)
  }
}
```


此外，gawk还提供了两个内置函数asort()和asorti()来对数组进行排序。
