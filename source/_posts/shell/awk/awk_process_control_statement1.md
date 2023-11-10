---
title: 精通awk系列(18)：awk流程控制：if、while、switch、for
p: shell/awk/awk_process_control_statement1.md
date: 2020-02-27 10:38:35
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------

# 流程控制语句

注：awk中语句块没有作用域，都是全局变量。

```
if (condition) statement [ else statement ]
expr1?expr2:expr3
while (condition) statement
do statement while (condition)
for (expr1; expr2; expr3) statement
for (var in array) statement
break
continue
next
nextfile
exit [ expression ]
{ statements }
switch (expression) {
    case value|regex : statement
    ...
    [ default: statement ]
}
```

## 代码块

```
{statement}
```

## if...else

```
# 单独的if
if(cond){
    statements
}

# if...else
if(cond1){
    statements1
} else {
    statements2
}

# if...else if...else
if(cond1){
    statements1
} else if(cond2){
    statements2
} else if(cond3){
    statements3
} else{
    statements4
}
```

搞笑题：妻子告诉程序员老公，去买一斤包子，如果看见卖西瓜的，就买两个。结果是买了两个包子回来。

```
# 自然语言的语义
买一斤包子
if(有西瓜){
    买两个西瓜
}

# 程序员理解的语义
if(没有西瓜){
    买一斤包子
}else{
    买两个包子
}
```

```
awk '
  BEGIN{
    mark = 999
    if (mark >=0 && mark < 60) {
      print "学渣"
    } else if (mark >= 60 && mark < 90) {
      print "还不错"
    } else if (mark >= 90 && mark <= 100) {
      print "学霸"
    } else {
      print "错误分数"
    }
  }
'
```

## 三目运算符?:

```
expr1 ? expr2 : expr3

if(expr1){
    expr2
} else {
    expr3
}
```

```
awk 'BEGIN{a=50;b=(a>60) ? "及格" : "不及格";print(b)}'
awk 'BEGIN{a=50; a>60 ? b="及格" : b="不及格";print(b)}' 
```


## switch...case

```
switch (expression) {
    case value1|regex1 : statements1
    case value2|regex2 : statements2
    case value3|regex3 : statements3
    ...
    [ default: statement ]
}
```

awk 中的switch分支语句功能较弱，只能进行等值比较或正则匹配。

各分支结尾需使用break来终止。

```
{
    switch($1){
        case 1:
            print("Monday")
            break
        case 2:
            print("Tuesday")
            break
        case 3:
            print("Wednesday")
            break
        case 4:
            print("Thursday")
            break
        case 5:
            print("Friday")
            break
        case 6:
            print("Saturday")
            break
        case 7:
            print("Sunday")
            break
        default:
            print("What day?")
            break
    }
}
```

分支穿透：
```
{
    switch($1){
        case 1:
        case 2:
        case 3:
        case 4:
        case 5:
            print("Weekday")
            break
        case 6:
        case 7:
            print("Weekend")
            break
        default:
            print("What day?")
            break
    }
}
```


## while和do...while

```
while(condition){
    statements
}

do {
    statements
} while(condition)
```

while先判断条件再决定是否执行statements，do...while先执行statements再判断条件决定下次是否再执行statements。

```
awk 'BEGIN{i=0;while(i<5){print i;i++}}'
awk 'BEGIN{i=0;do {print i;i++} while(i<5)}'
```

多数时候，while和do...while是等价的，但如果第一次条件判断失败，则do...while和while不同。
```
awk 'BEGIN{i=0;while(i == 2){print i;i++}}'
awk 'BEGIN{i=0;do {print i;i++} while(i ==2 )}'
```

所以，while可能一次也不会执行，do...while至少会执行一次。

一般用while，do...while相比while来说，用的频率非常低。


## for循环

```
for (expr1; expr2; expr3) {
    statement
}

for (idx in array) {
    statement
}
```
