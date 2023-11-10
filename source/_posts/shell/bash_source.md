---
title: 理解$0和BASH_SOURCE
p: shell/bash_source.md
date: 2021-01-25 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# 理解$0和BASH_SOURCE

一个shell(bash)脚本有两种执行方式：

- 直接执行，类似于执行二进制程序  
- source加载，类似于加载库文件  

`$0`保存了被执行脚本的程序名称。注意，它保存的是以二进制方式执行的脚本名而非以source方式加载的脚本名称。

例如，执行a.sh时，a.sh中的`$0`的值是a.sh，如果a.sh执行b.sh，b.sh中的`$0`的值是b.sh，如果a.sh中source b.sh，则b.sh中的`$0`的值为a.sh。

除了`$0`，bash还提供了一个数组变量`BASH_SOURCE`，该数组保存了bash的SOURCE调用层次。这个层次是怎么体现的，参考下面的示例。

执行shell脚本a.sh时，shell脚本的程序名`a.sh`将被添加到`BASH_SOURCE`数组的第一个元素中，即`${BASH_SOURCE[0]}`的值为`a.sh`，这时`${BASH_SOURCE[0]}`等价于`$0`。

当在a.sh中执行b.sh时：

```bash
# a.sh中的$0和BASH_SOURCE
$0 --> "a.sh"
${BASH_SOURCE[0]} -> "a.sh"

# b.sh中的$0和BASH_SOURCE
$0 --> "b.sh"
${BASH_SOURCE[0]} -> "b.sh"
```

当在a.sh中source b.sh时：

```bash
# a.sh中的$0和BASH_SOURCE
$0 --> "a.sh"
${BASH_SOURCE[0]} -> "a.sh"

# b.sh中的$0和BASH_SOURCE
$0 --> "a.sh"
${BASH_SOURCE[0]} -> "b.sh"
${BASH_SOURCE[1]} -> "a.sh"
```

当在a.sh中source b.sh时，如果b.sh中还执行了source c.sh，那么：

```bash
# a.sh中的$0和BASH_SOURCE
$0 --> "a.sh"
${BASH_SOURCE[0]} -> "a.sh"

# b.sh中的$0和BASH_SOURCE
$0 --> "a.sh"
${BASH_SOURCE[0]} -> "b.sh"
${BASH_SOURCE[1]} -> "a.sh"

# c.sh中的$0和BASH_SOURCE
$0 --> "a.sh"
${BASH_SOURCE[0]} -> "c.sh"
${BASH_SOURCE[1]} -> "b.sh"
${BASH_SOURCE[2]} -> "a.sh"
```

使用脚本来验证一下`BASH_SOURCE`和`$0`。在x.sh中source y.sh，在y.sh中source z.sh：

```bash
#~ /tmp/x.sh
cat >/tmp/x.sh<<'eof'
source y.sh
echo "x.sh: ${BASH_SOURCE[@]}"
eof

#~ /tmp/y.sh
cat >/tmp/y.sh<<'eof'
source z.sh
echo "y.sh: ${BASH_SOURCE[@]}"
eof

#~ /tmp/z.sh
cat >/tmp/z.sh<<'eof'
echo "z.sh: ${BASH_SOURCE[@]}"
eof
```

执行x.sh输出结果：
```bash
$ bash /tmp/x.sh
z.sh: z.sh y.sh /tmp/x.sh
y.sh: y.sh /tmp/x.sh
x.sh: /tmp/x.sh
```


