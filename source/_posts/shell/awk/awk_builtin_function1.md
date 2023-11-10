---
title: 精通awk系列(26)：awk预定义函数(1)：数值和字符串预定义函数
p: shell/awk/awk_builtin_function1.md
date: 2020-04-12 15:48:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# awk预定义函数

预定义函数分为几类：  
- 数值类内置函数  
- 字符串类内置函数  
- 时间类内置函数  
- 位操作内置函数  
- 数据类型相关内置函数：isarray()、typeof()  
- IO类内置函数：close()、system()、fflush()  

## awk数值类内置函数

```
int(expr)     截断为整数：int(123.45)和int("123abc")都返回123，int("a123")返回0
sqrt(expr)    返回平方根
rand()        返回[0,1)之间的随机数，默认使用srand(1)作为种子值
srand([expr]) 设置rand()种子值，省略参数时将取当前时间的epoch值(精确到秒的epoch)作为种子值
```

例如：
```
$ awk 'BEGIN{srand();print rand()}'
0.0379114
$ awk 'BEGIN{srand();print rand()}'
0.0779783
$ awk 'BEGIN{srand(2);print rand()}'
0.893104
$ awk 'BEGIN{srand(2);print rand()}'
0.893104
```

生成`[10,100]`之间的随机整数。
```
awk 'BEGIN{srand();print 10+int(91*rand())}'
```

## awk字符串类内置函数

注意，awk中涉及到**字符索引的函数，索引位都是从1开始计算**，和其它语言从0开始不一样。

### 基本函数

- `sprintf(format, expression1, ...)`：返回格式化后的字符串，参考[sprintf](#sprintf)  

   - `a=sprintf("%s\n","abc")`  

- `length()`：返回字符串字符数量、数组元素数量、或数值转换为字符串后的字符数量  

    ```
    awk '
        BEGIN{
            print length(1.23)     # 4   # CONVFMT %.6g
    
            print 1.234567         # 1.23457
            print length(1.234567) # 7 
            print length(122341223432.1213241234)  # 11
        }'
    ```

- `strtonum(str)`：将字符串转换为十进制数值  

   - 如果str以0开头，则将其识别为8进制  
   - 如果str以0x或0X开头，则将其识别为16进制  

- `tolower(str)`：转换为小写  

- `toupper(str)`：转换为大写  

- `index(str,substr)`：从str中搜索substr(子串)，返回搜索到的索引位置(索引从1开始)，搜索不到则返回0  

### awk substr()

- `substr(string,start[,length])`：从string中截取子串  

start是截取的起始索引位(索引位从1开始而非0)，length表示截取的子串长度。如果省略length，则表示从start开始截取剩余所有字符。

```
awk '
    BEGIN{
        str="abcdefgh"
        print substr(str,3)   # cdefgh
        print substr(str,3,3) # cde
    }
'
```

如果start值小于1，则将其看作为1对待，如果start大于字符串的长度，则返回空字符串。

如果length小于或等于0，则返回空字符串。

### awk split()和patsplit()

- `split(string, array [, fieldsep [, seps ] ])`：将字符串分割后保存到数组array中，数组索引从1开始存储。并返回分割得到的元素个数  

其中fieldsep指定分隔符，可以是正则表达式方式的。如果不指定该参数，则默认使用FS作为分隔符，而FS的默认值又是空格。

seps是一个数组，保存了每次分割时的分隔符。

例如：
```
split("abc-def-gho-pq",arr,"-",seps)
```

其返回值为4。同时得到的数组a和seps为：
```
arr[1] = "abc"
arr[2] = "def"
arr[3] = "gho"
arr[4] = "pq"

seps[1] = "-"
seps[2] = "-"
seps[3] = "-"
```

split在开始工作时，会先清空数组，所以，将split的string参数设置为空，可以用于清空数组。
```
awk 'BEGIN{arr[1]=1;split("",arr);print length(arr)}'  # 0
```

如果分隔符无法匹配字符串，则整个字符串当作一个数组元素保存到数组array中。
```
awk 'BEGIN{split("abcde",arr,"-");print arr[1]}' # abcde
```

- `patsplit(string, array [, fieldpat [, seps ] ])`：用正则表达式fieldpat匹配字符串string，将所有匹配成功的部分保存到数组array中，数组索引从1开始存储。返回值是array的元素个数，即匹配成功了多少次  

如果省略fieldpat，则默认采用预定义变量FPAT的值。

```
awk '
    BEGIN{
        patsplit("abcde",arr,"[a-z]")
        print arr[1]   # a
        print arr[2]   # b
        print arr[3]   # c
        print arr[4]   # d
        print arr[5]   # e
    }
'
```

### awk match()

- `match(string,reg[,arr])`：使用reg匹配string，返回匹配成功的索引位(从1开始计数)，匹配失败则返回0。如果指定了arr参数，则arr[0]保存的是匹配成功的字符串，arr[1]、arr[2]、...保存的是各个分组捕获的内容  

match匹配时，同时会设置两个预定义变量：RSTART和RLENGTH  
- 匹配成功时：  
   - RSTART赋值为匹配成功的索引位，从1开始计数  
   - RLENGTH赋值为匹配成功的字符长度  
- 匹配失败时：  
   - RSTART赋值为0  
   - RLENGTH赋值为-1  

例如：
```
awk '
    BEGIN{
        where = match("foooobazbarrrr","(fo+).*(bar*)",arr)
        print where   # 1
        print arr[0]  # foooobazbarrrr
        print arr[1]  # foooo
        print arr[2]  # barrrr
        print RSTART  # 1
        print RLENGTH # 14
    }
'
```

因为match()匹配成功时返回值为非0，而匹配失败时返回值为0，所以可以直接当作条件判断：
```
awk '
  {
    if(match($0,/A[a-z]+/,arr)){
      print NR " : " arr[0]
    }
  }
' a.txt
```

### awk sub()和gsub()

- `sub(regexp, replacement [, target])`  
- `gsub(regexp, replacement [, target])`：sub()的全局模式  

sub()从字符串target中进行正则匹配，并使用replacement对第一次匹配成功的部分进行替换，替换后保存回target中。返回替换成功的次数，即0或1。

target必须是一个可以赋值的变量名、$N或数组元素名，以便用它来保存替换成功后的结果。不能是字符串字面量，因为它无法保存数据。

如果省略target，则默认使用`$0`。

需要注意的是，如果省略target，或者target是`$N`，那么替换成功后将会使用OFS重新计算`$0`。

```
awk '
    BEGIN{
        str="water water everywhere"
        #how_many = sub(/at/, "ith", str)
        how_many = gsub(/at/, "ith", str)
        print how_many   # 1
        print str        # wither water everywhere
    }
'
```

在replacement参数中，可以使用一个特殊的符号`&`来引用匹配成功的部分。注意sub()和gsub()不能在replacement中使用反向引用`\N`。

```
awk '
    BEGIN{
        str = "daabaaa"
        gsub(/a+/,"C&C",str)
        print str  # dCaaCbaaa
    }
'
```

如果想要在replacement中使用`&`纯字符，则转义即可。
```
sub(/a+/,"C\\&C",str)
```

> 两根反斜线：  
> 因为awk在正则开始工作时，首先会扫描所有awk代码然后编译成awk的内部格式，扫描期间会解析反斜线转义，使得`\\`变成一根反斜线。当真正开始运行后，sub()又要解析，这时`\&`才表示的是对&做转义。
> 扫描代码阶段称为词法解析阶段，运行解析阶段称为运行时解析阶段。

### awk gensub()

gawk支持的gensub()，完全可以取代sub()和gsub()。

- `gensub(regexp, replacement, how [, target])`：

可以替代sub()和gsub()。

how指定替换第几个匹配，例如指定为1表示只替换第一个匹配。此外，还可以指定为`g`或`G`开头的字符串，表示全局替换。

gensub()返回替换后得到的结果，而target不变，如果匹配失败，则返回target。这和sub()、gsub()不一样，sub()、gsub()返回的是替换成功的次数。

gensub()的replacement部分可以使用`\N`来引用分组匹配的结果，而sub()、gsub()不允许使用反向引用。而且，gensub()在replacement部分也还可以使用`&`或`\0`来表示匹配的整个结果。

```
awk 'BEGIN{
    a = "abc def"
    b = gensub(/(.+) (.*)/, "\\2 \\1, \\0 , &", "g", a)
    print b  # def abc, abc def , abc def
}'
```

### awk asort()和asorti()

- `asort(src,[dest [,how]])`  
- `asorti(src,[dest [,how]])`  

asort对数组src的值进行排序，然后将排序后的值的索引改为1、2、3、4...序列。返回src中的元素个数，它可以当作排序后的索引最大值。

asorti对数组src的索引进行排序，然后将排序后的索引值的索引改为1、2、3、4...序列。返回src中的元素个数，它可以当作排序后的索引最大值。

```
arr["last"] = "de"
arr["first"] = "sac"
arr["middle"] = "cul"
```

asort(arr)得到：
```
arr[1] = "cul"
arr[2] = "de"
arr[3] = "sac"
```

asorti(arr)得到：
```
arr[1] = "first"
arr[2] = "last"
arr[3] = "middle"
```

如果指定dest，则将原始数组src备份到dest，然后对dest进行排序，而src保持不变。

how参数用于指定排序时的方式，其值指定方式和`PROCINFO["sorted_in"]`一致：可以是预定义的排序函数，也可以是用户自定义的排序函数。参考[指定数组遍历顺序](#arrsort)。
