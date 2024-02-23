---
title: Shell脚本深入教程：Bash数组基础
p: shell/script_course/shell_array.md
date: 2020-05-19 14:17:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Bash数组

Bash支持普通的数值索引数组，还支持关联数组。

## 数组基础

数组是最常见的数据结构，可以用来存放多个数据。

有两种类型的数组：数值索引类型数组和关联数组。

- 数值索引类型数组使用0、1、2、3...数值作为索引，通过索引可找到数组中对应位置的数据
- 关联数组使用名称(通常是字符串，但某些语言中支持其它类型)作为索引，是key/value模式的结构，key是索引，value是对应元素的值。通过key索引可找到关联数组中对应的数据value

关联数组在其它语言中也称为map、hash结构、dict等。所以，从严格意义上来说，关联数组不是数组，它不符合数组数据结构，只是因为访问方式类似于数值索引类型的数组，才在某些语言中称之为关联数组。

**数值索引类型的数组**中存放的每个元素在内存中连续的，只要知道第一个元素的地址，就能计算出第2个元素地址、第3个元素地址、第N个元素地址。

![](/img/shell/1581164046606.png)

为了方便访问数组中的各个元素，都会使用索引下标来一一对应各个元素，且索引下标从0开始。也就是说，index=0的下标表示数组中的第一个元素，index=1的下标表示数组中第二个元素，index=N的下标表示数组中第N+1个元素。

所以，要访问数组arr中的各个元素：

```shell
arr[0]:表示arr中第一个元素
arr[1]:表示arr中第二个元素
arr[2]:表示arr中第三个元素
arr[3]:表示arr中第四个元素
```

有些语言为了提供更方便的访问方式，还支持负数索引，表示从尾部向头部访问。

```shell
arr[-1]:表示arr中倒数第一个元素
arr[-2]:表示arr中倒数第二个元素
arr[-3]:表示arr中倒数第三个元素
arr[-4]:表示arr中倒数第四个元素
```

关联数组是key/value类型的hash结构，key和value一一对应。关联数组中各个元素在内存中不连续。

![](/img/shell/1581164068159.png)

在Bash中访问关联数组中元素的方式和访问数值索引类型数组中的元素是类似的，仅仅只是把数值索引换成key。

所以，要访问关联数组arr中的各个元素：

```
arr['name']：表示访问key=name所对应的value，即返回junmajinlong
arr['age']：表示访问key=age所对应的value，即返回22
```

## 定义、访问数值索引类型数组

直接赋值某元素，会自动创建数组：

```shell
$ arr1[1]=one
$ arr1[2]=two
$ arr1[3]=three

# 访问数组各元素
$ echo ${arr1[1]}
one
$ echo ${arr1[2]}
two
$ echo ${arr1[-1]}   # 负数索引
three
```

通过括号赋值创建数组：

```shell
$ arr2=(zero one two three)

# 访问数组
$ echo ${arr2[@]}
zero one two three
$ echo ${arr2[1]}
one
```

通过Bash内置命令`declare`创建数值索引类型数组：

```shell
# 下面等价
declare arr3
declare -a arr4
```

`declare`定义数组时可直接赋值：

```shell
declare arr5=(zero one two)
```

需注意，可将数组当作变量来访问，这时访问的是数组的第一个元素：

```bash
$ arr=(11 22 33)
$ echo $arr  # 11
```

如果是调试性的简单查看数组所有元素，可直接使用`printf`或`declare -p`：

```bash
$ arr=(11 22 33)
$ printf "%s " "${arr[@]}"   # 输出：11 22 33
$ declare -p arr # 输出：declare -a arr=([0]="11" [1]="22" [2]="33")
```

## 定义、访问关联数组

关联数组需要先使用`declare -A`来定义，然后才表示是关联数组。

```shell
declare -A aarr
aarr[one]=1      # key部分可不加引号
aarr[six]=6
aarr[five]=5
aarr[nine]=9
```

也可以使用括号直接赋值：

```shell
aarr=([one]=1 [two]=2)
```

访问关联数组中元素：

```shell
$ echo ${aarr[one]}
1
$ echo ${aarr[six]}
6
```

## 其它数组基本操作

下面的操作对数值索引类型的数组、关联数组都生效。

访问数组所有元素的值：

```shell
$ echo ${arr[@]}
$ echo ${arr[*]}
```

获取数组所有索引：

```shell
$ echo ${!arr[@]}
$ echo ${!arr[*]}
```

获取数组中非空元素的个数：

```shell
$ echo ${#arr[@]}
$ echo ${#arr[*]}
```

## 向shell函数传递数组引用

多数人对Bash数组有误解，认为无法将Bash数组传递给函数参数，但实际上是可以的。

Bash的`declare`命令有一个选项`-n`，可以定义变量的引用。例如：

```bash
function ff(){
  declare -n _inner_arr__=$1 # 现在_inner_arr__和$1的值是同一个变量
  echo ${_inner_arr__[@]}
  #...do something...
  declare +n _inner_arr__  # 取消变量引用
}

# 传递数组
$ arr=(11 22 33)
$ ff arr
11 22 33
```

`declare`命令是深入shell脚本必学的命令之一，请记住它很重要，以后遇到一些变量相关的功能需求，记得去找declare。

## 扩展(插入)数组和剔除(删除)数组元素

将一个或多个元素添加到数组尾部(或者新建数组)，即其它编程语言中push和extend到数组的操作：：
```shell
# 尾部添加一个元素、多个元素，如果arr当前不存在，则自动新建数组
$ arr+=(1)
$ arr+=(2 3 4)
$ declare -p arr
declare -a arr=([0]="1" [1]="2" [2]="3" [3]="4")
```

删除数组中某个索引位置的元素，即其它编程语言中对数组进行的delete或者remove操作：
```shell
# 注意，对非尾部元素进行unset，数组的索引位不会改变，
# 即：unset所删除元素后面的元素不会自动左移
$ arr=(1 2 3 4)
$ unset arr[-1]
$ declare -p arr
declare -a arr=([0]="1" [1]="2" [2]="3")

$ unset arr[0] 
$ declare -p arr  # 注意输出结果中索引值是从1开始的，并不是0
declare -a arr=([1]="2" [2]="3")
```

删除数组尾部元素，即其它编程语言中对数组进行pop的操作，只需unset最后一个元素即可：
```shell
$ arr=(1 2 3 4)
$ unset arr[-1]
$ declare -p arr
declare -a arr=([0]="1" [1]="2" [2]="3")
```

删除数组头部元素(第一个元素)，即其它编程语言中对数组进行shift的操作：
```shell
# 从索引位1开始取到最后一个元素，并将它们重新构建成数组
$ arr=(1 2 3 4)
$ arr=("${arr[@]:1}")
```

在数组头部插入元素，即其它编程语言中对数组进行unshift的操作：
```shell
# 向arr数组头部插入1和2两个元素
$ arr=(3 4)  # 原数组
$ arr1=(1 2)
$ arr1+=("${arr[@]}")
$ arr=("${arr1[@]}")
```

## 遍历数组

遍历数组是指依次取得数组中的每个元素。Bash中可使用for循环来遍历。

遍历方式1：使用`for...in ${arr[@]}`，可以获取每个元素的值

```shell
for i in ${arr[@]};do
	echo $i
done
```

遍历方式2：使用`for...in ${!arr[@]}`，可以获取每个元素的索引，间接的可以获取每个元素的值

```shell
for i in ${!arr[@]};do
	echo index:$i
	echo value: ${arr[$i]}
done
```

遍历方式3：如果是数值索引类型，且每个元素都不为空，则还可以使用数组长度的方式遍历

```shell
for((i=0;i<${#arr[@]};i++)){
  echo $i
  echo ${arr[$i]}
}
```

例如，文件a.txt内容如下：

```shell
portmapper
portmapper
portmapper
portmapper
portmapper
portmapper
status
status
mountd
mountd
mountd
mountd
mountd
mountd
nfs
nfs
nfs_acl
nfs
nfs
nfs_acl
nlockmgr
nlockmgr
nlockmgr
nlockmgr
nlockmgr
nlockmgr
```

要统计每行数据出现的次数。

```shell
declare -A aarr
for i in `cat a.txt`;do
  let ++aarr[$i]
done

for i in ${!aarr[@]};do
  echo ------
  echo data: $i
  echo count: ${aarr[$i]}
done
```


## 判断数组中是否存在某元素

在Bash中没有直接判断数组中是否存在某元素的方式，但是可以遍历数组并比较每个元素来判断，虽然麻烦，但是也不难。

例如：
```shell
# 判断数组中是否存在abc元素
arr=("abc" "def" "abc defg")
for v in "${arr[@]}";do 
  if [ "${v}" == "abc" ];then
    echo "yes"
    break
  fi
done
```

可以将该功能定义为函数：
```shell
# 第一个参数为数组名，第二个参数为要判断是否存在的值
# 返回0表示存在，返回1表示不存在
# 第二个参数为空时，也认为存在，返回0
function arr_contains() {
    [ $# -eq 0 ] && {
        echo "argument error"
        exit 2
    }

    [ $# -eq 1 ] && return 0

    declare -n _arr="$1"
    declare v="$2"
    local elem

    for elem in "${_arr[@]}";do
        [ "$elem" == "$v" ] && return 0
    done

    return 1
}

export -f arr_contains
```

## 使用Shell数组保存命令行的选项

在Shell中，数组一个很常用的功能是保存某个命令的动态选项。

有时候要执行的命令选项可能是动态定义的，比如在某种条件下才添加某个选项，在另一个条件下添加另一个选项，通常会定义一个字符串变量，并在各种条件判断下将对应的选项和参数动态追加到该字符串变量中，但这需要精确掌握引号的使用规则，否则容易出错。这时候用Shell的数组很友好。

例如，动态定义rsync的`--exclude`选项，并假设rsync总是带有`-avz`选项。下面给出两种动态定义选项的方式。

1.使用字符串变量动态保存rsync选项：
```shell
rsync_opt="-avz"
while [ $# -ge 1 ];do 
  case "$1" in 
    "--no-log") 
      rsync_opts="$rsync_opts --exclude='*.log'"
      shift
      ;;
    "--no-lock") 
      rsync_opts="$rsync_opts --exclude='*.lock'"
      shift
      ;;
    *)
      :
  esac
done

# 下面的$rsync_opt一定不能使用双引号
rsync $rsync_opt <SRC> <DEST>
```

2.使用数组保存rsync选项:
```shell
rsync_opts=(-avz)
while [ $# -ge 1 ];do 
  case "$1" in 
    "--no-log") 
      rsync_opts+=(--exclude='*.log')
      shift
      ;;
    "--no-lock") 
      rsync_opts+=(--exclude='*.lock')
      shift
      ;;
    *)
      :
  esac
done

rsync "${rsync_opts[@]}" <SRC> <DEST>
```

如果某个选项带参数，或者一次性要添加多个选项或参数，可：
```shell
opts+=(-o arg)
opts+=(arg1 arg2)
```