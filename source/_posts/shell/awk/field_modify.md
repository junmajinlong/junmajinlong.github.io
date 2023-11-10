---
title: 精通awk系列(9)：修改字段或NF引起的$0重新计算
p: shell/awk/field_modify.md
date: 2019-11-23 10:37:37
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

<a name="NF"></a>

# 修改字段或NF值的联动效应

注意下面的分割和计算两词：分割表示使用FS（field Separator），计算表示使用预定义变量OFS（Output Field Separator）。

1. 修改`$0`，将使用`FS`重新分割字段，所以会影响`$1、$2...`  
2. 修改`$1、$2`，将根据`$1`到`$NF`等各字段来重新计算`$0`  
   - 即使是`$1 = $1`这样的原值不变的修改，也一样会重新计算`$0`  
3. 为不存在的字段赋值，将新增字段并按需使用空字符串填充中间的字段，并使用`OFS`重新计算`$0`  
   - `awk 'BEGIN{OFS="-"}{$(NF+2)=5;print $0}' a.txt`  
4. 增加NF值，将使用空字符串新增字段，并使用`OFS`重新计算`$0`  
   - `awk 'BEGIN{OFS="-"}{NF+=3;print $0}' a.txt`  
5. 减小NF值，将丢弃一定数量的尾部字段，并使用`OFS`重新计算`$0`    
   - `awk 'BEGIN{OFS="-"}{NF-=3;print $0}' a.txt`  

# 关于$0

当读取一条record之后，将原原本本地被保存到`$0`当中。

```
awk '{print $0}' a.txt
```

![](/img/shell/awk/733013-20191123155324013-804159444.jpg)

换句话说，没有导致`$0`重建，`$0`就一直是原原本本的数据，所以指定OFS也无效。

```
awk 'BEGIN{OFS="-"}{print $0}' a.txt  # OFS此处无效
```

当`$0`重建后，将自动使用OFS重建，所以即使没有指定OFS，它也会采用默认值(空格)进行重建。

```
awk '{$1=$1;print $0}'  a.txt  # 输出时将以空格分隔各字段
awk '{print $0;$1=$1;print $0}' OFS="-" a.txt
```

如果重建`$0`之后，再去修改OFS，将对当前行无效，但对之后的行有效。所以如果也要对当前行生效，需要再次重建。

```
# OFS对第一行无效
awk '{$4+=10;OFS="-";print $0}' a.txt

# 对所有行有效
awk '{$4+=10;OFS="-";$1=$1;print $0}' a.txt
```

关注`$0`重建是一个非常有用的技巧。

例如，下面通过重建`$0`的技巧来实现去除行首行尾空格并压缩中间空格：

```
$ echo "   a  b  c   d   " | awk '{$1=$1;print}'
a b c d
$ echo "     a   b  c   d   " | awk '{$1=$1;print}' OFS="-"            
a-b-c-d
```

