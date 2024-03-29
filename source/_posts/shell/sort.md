---
title: 文本排序的王者：玩透sort命令
p: shell/sort.md
date: 2023-07-12 18:20:41
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------


# 文本排序的王者：玩透sort命令

sort是排序工具，它完美贯彻了Unix哲学："只做一件事，并做到完美"。它的排序功能极强、极完整，只要文件中的数据足够规则，它几乎可以排出所有想要的排序结果，是一个非常优质的工具。

虽然sort很强大，但它的选项很少，使用方法也很简单。更让人觉得它成功的地方在于：即使想要实现复杂、完整的sort功能，所使用的选项和一般使用时的选项没什么不同。只不过要实现复杂功能时，必须得理解sort是如何工作的。

也就是说，**没搞懂sort工作机制时，它也能完成任务，指哪就能打哪，但没被指到的地方难免会有所偏差和疑惑。只有搞懂了sort机制，才能真正的指哪打哪，结果中一丝偏差也没有，即使出现了偏差也知道是为什么。**

本文先解释sort命令的常用选项，再给出sort的简单使用示例，用于初步解释sort各选项，最后对sort深入说明。更完整的选项说明可参考info sort的译文：[sort命令中文手册(info sort翻译)](/shell/sort_trans)。

## sort选项

sort读取每一行输入，并按照指定的分隔符将每一行划分成多个字段，这些字段就是sort排序的对象。同时，sort可以指定按照何种排序规则进行排序，如按照当前字符集排序规则(这是默认排序规则)、按照字典排序规则、按照数值排序规则、按照月份排序规则、按照文件大小格式(k<M<G)。还可以去除重复行，指定降序或升序(默认)的排序方式。

默认的排序规则为字符集排序规则，通常几种常见字符的顺序为："空字符串<空白字符<数值<a<A<b<B<...<z<Z"，这也是字典排序的规则。

语法格式：

```
sort [OPTION]... [FILE]...
 
选项说明：
-c：检测给定的文件是否已经排序。如未排序，
    则会输出诊断信息，提示从哪一行开始乱序。
-C：类似于"-c"，只不过不输出任何诊断信息。
    可以通过退出状态码1判断出文件未排序。
-m：对给定的多个已排序文件进行合并。在合并过
    程中不做任何排序动作。
-b：忽略字段的前导空白字符。空格数量不固定时，
   该选项几乎是必须要使用的。"-n"选项隐含该选项。
-d：按照字典顺序排序，只支持字母、数值、空白。
    除了特殊字符，一般情况下基本等同于默认排序规则。
--debug：将显示排序的过程以及每次排序所使用的字段、
         字符。同时还会在最前几行显示额外的信息。
-f：将所有小写字母当成大写字母。例如，"b"和"B"是相
  ：同的。在和"-u"选项一起使用时，如果排序字段的比
    较结果相等，则丢弃小写字母行。
-k：指定要排序的key，key格式："POS1[,POS2]"，
    POS1为key起始字段，POS2为key结束字段。
-n：按数值排序。空字符串""或"\0"被当作空。该选项除
    了能识别负号"-"，其他所有非数字字符都不识别。当按
    数值排序时，遇到不识别的字符时将立即结束该key的排序。
-M：按字符串格式的月份排序。会自动转换成大写，并取缩写值。
    规则：unknown<JAN<FEB<...<NOV<DEC。
-o：将结果输出到指定文件中。
-r：默认是升序排序，使用该选项将得到降序排序的结果。
    注意："-r"不参与排序动作，只是操作排序完成后的结果。
-s：禁止sort做"最后的排序"。
-t：指定字段分隔符。
    对于特殊符号(如制表符)，可使用类似于-t$'\t'或
    -t'ctrl+v,tab'(先按ctrl+v，然后按tab键)的方法实现。
-u：只输出重复行的第一行。结合"-f"使用时，重复的小写行被丢弃。
```

## sort示例

此小节为sort的简单用法示例，也是平时最可能用上的示例。如果只是为了使用sort，而不是为了刨根问题，本小节已经足够。

假设当前已有文件system.txt，内容如下：其中空白部分为单个制表符。

```bash
[root@xuexi tmp]# cat system.txt
1       mac     2000    500
2       winxp   4000    300
3       bsd     1000    600
4       linux   1000    200
5       SUSE    4000    300
6       Debian  600     200
```

(1).不加任何选项时，将对整行从第一个字符开始依次向后直到行尾按照默认的字符集排序规则做升序排序。

```bash
[root@xuexi tmp]# sort system.txt
1       mac     2000    500
2       winxp   4000    300
3       bsd     1000    600
4       linux   1000    200
5       SUSE    4000    300
6       Debian  600     200
```

由于每行的第一个字符1<2<3<4<5<6，所以结果如上。

(2).以第三列为排序列进行排序。由于要划分字段，所以指定字段分隔符。指定制表符这种无法直接输入的特殊字符的方式是`$'\t'`。

```bash
[root@xuexi tmp]# sort -t $'\t' -k3 system.txt  
4       linux   1000    200
3       bsd     1000    600
1       mac     2000    500
2       winxp   4000    300
5       SUSE    4000    300
6       Debian  600     200
```

结果中虽然1000<2000<4000的顺序是对了，但600却排在最后面，因为这是按照默认字符集排序规则进行排序的，字符6大于4，所以排最后一行。

(3).对第三列按数值排序规则进行排序。

```bash
[root@xuexi tmp]# sort -t $'\t' -k3 -n system.txt
6       Debian  600     200
3       bsd     1000    600
4       linux   1000    200
1       mac     2000    500
2       winxp   4000    300
5       SUSE    4000    300
```

结果中600已经排在第一行。结果中第2行、第3行的第三列值均为1000，如何决定这两行的顺序？

(4).在对第3列按数值排序规则排序的基础上，使用第四列作为决胜属性，且是以数值排序规则对第四列排序。

```bash
[root@xuexi tmp]# sort -t $'\t' -k3 -k4 -n system.txt
6       Debian  600     200
4       linux   1000    200
3       bsd     1000    600
1       mac     2000    500
2       winxp   4000    300
5       SUSE    4000    300
```

如果想在第3列按数值排序后，以第2列作为决胜列呢？由于第2列为字母而非数值，所以下面的语句是错误的，虽然得到了期望的结果。

```bash
[root@xuexi tmp]# sort -t $'\t' -k3 -k2 -n system.txt
6       Debian  600     200
3       bsd     1000    600
4       linux   1000    200
1       mac     2000    500
2       winxp   4000    300
5       SUSE    4000    300
```


之所以最终得到了正确的结果，是因为**默认情况下，在命令行中指定的排序行为结束后，sort还会做最后一次排序，这最后一次排序是对整行按照完全默认规则进行排序的，也就是按字符集、升序排序。**由于1000所在的两行中的第一个字符3小于4，所以3排在前面。

之所以说上面的语句是错误的，是因为第2列第一个字符是字母而不是数值，**在按数值排序时，字母是不可识别字符，一遇到不可识别字符就会立即结束该字段的排序行为。**可以使用`--debug`选项来查看排序的过程和排序时所使用的列。注意，该选项只有CentOS 7上的sort才有。


```bash
[root@xuexi tmp]# sort --debug -t $'\t' -k3 -k2 -n system.txt
sort: using ‘en_US.UTF-8’ sorting rules
sort: key 1 is numeric and spans multiple fields
sort: key 2 is numeric and spans multiple fields
6>Debian>600>200
         ___            # 第1次排序行为，即对"-k3"排序，此次用于排序的字段为第3列
  ^ no match for key    # 第2次排序行为，即对"-k2"排序，但显示无法匹配排序key
________________        # 默认sort总会进行最后一次排序，排序对象为整行
3>bsd>1000>600
      ____
  ^ no match for key
______________
4>linux>1000>200
        ____
  ^ no match for key
________________
1>mac>2000>500
      ____
  ^ no match for key
______________
2>winxp>4000>300
        ____
  ^ no match for key
________________
5>SUSE>4000>300
       ____
  ^ no match for key
_______________
```

(5).在对第3列按数值排序规则排序的基础上，使用第2列作为决胜属性，且以默认排序规则对此列降序排序。


```bash
[root@xuexi tmp]# sort -t $'\t' -k3n -k2r system.txt
6       Debian  600     200
4       linux   1000    200
3       bsd     1000    600
1       mac     2000    500
2       winxp   4000    300
5       SUSE    4000    300
```


由于既要对第3列按数值升序排序，又要对第2列按默认规则降序排序，因此只能对每个字段单独分配选项。**注意，虽然"r"选项是降序结果，但它不影响排序过程，只影响最终排序结果。也就是说，在按照升序排序结束得到最终结果后，再反转第2列顺序，也就是得到了降序的结果。同样也说明，sort在排序的时候，一定且只能按照升序排序，只有排序动作结束了"r"选项才开始工作。**

紧跟在字段后的选项(如"-k3n"的"n"和"-k2r"的"r")称为私有选项，使用短横线写在字段外的选项(如"-n"、"-r")为全局选项。**当没有为字段分配私有选项时，该排序字段将继承全局选项。**当然，只有像"-n"、"-r"这样的排序性的选项才能继承和分配给字段，"-t"这样的选项则无法分配。

因此，`-n -k3 -k4`、`-n -k3n -k4`和`-k3n -k4n`是等价的，`-r -k3n -k4`和`-k3nr -k4r`是等价的。

实际上，上面的命令写法并不严谨。更标准的写法应该如下：

```bash
sort -t $'\t' -k3n -k2,2r system.txt
```

`-k2,2`表示排序对象从第2个字段开始到第2个字段结束，也就是限定了只对第二个字段排序。**它的格式为"POS1,POS2"，如果省略POS2，将自动扩展到行尾**，即`-k2`等价于`-k2,4`，也就是说，对整个第2列到第4列进行排序。

需要注意，由于上面的`-k2`继承了全局默认的排序规则，即按字符排序而非按数值排序，此时它能够等价于`-k2,4`，但如果是`-k2n`按照数值排序的话，它不等价于`-k2,4n`或`-k2n,4n`或`-k2n,4`(这3者为等价写法)，之所以不等价，是因为按数值排序时只能识别数字和负号"-"，当排序时遇到其他所有字符，都将立即结束此次排序。所以`-k2n`等价于`-k2,2n`或`-k2n,2`或`-k2n,2n`。

这些理论性的知识点，请参照下一小节sort的[理论内容：深入研究sort](#more_sort)。后文也不再解释理论性的内容，只是介绍命令使用方法。

(6).在对第3列按数值排序规则排序的基础上，使用第2列的第2个字符作为决胜属性，且以默认排序规则对此列升序排序。


```bash
[root@xuexi tmp]# sort -t $'\t' -k3n -k2.2,2.2 system.txt
6       Debian  600     200
4       linux   1000    200
3       bsd     1000    600
1       mac     2000    500
2       winxp   4000    300
5       SUSE    4000    300
```


其中`-k2.2,2.2`表示从第2个字段的第2个字符开始，到第2个字段的第2个字符结束，即严格限定为第2个字段第2个字符。如果需要对此字符降序排序，则`-k2.2,2.2r`。

(7).使用`-u`去除重复字段所在的行。例如第3列有两行1000，两行4000，去除字段重复的行时，将只保留排在前面的第一行。

```bash
[root@xuexi tmp]# sort -t $'\t' -k3n -u system.txt
6       Debian  600     200
3       bsd     1000    600
1       mac     2000    500
2       winxp   4000    300
```

由于需要去除重复字段的行，因此使用`-u`时将禁止sort做"最后一次排序"。至于字段重复的行中，如何判断哪一行是排在最前面的行，需要搞懂sort的整个工作机制，请通读本文。

`sort -u`和`sort | uniq`是等价的，但是如果多指定几个选项，它们将不等价。例如，`sort -n -u`只会检查排序字段数值部分的唯一性，但`sort -n | uniq`在sort对行中字段按数值排序后，uniq将检查整个行的唯一性。

(8).将排序结果保存到文件中。即可以使用重定向，也可以使用`-o`选项，但使用重定向不可保存到原文件，因为在sort开始执行前，原文件先被重定向截断。而使用`-o`则没有这样的问题，因为sort在打开文件前先完成数据的读取。但`-o`和`-m`一起使用时，同样不安全。

```bash
[root@xuexi tmp]# sort -t $'\t' -k3n -o system1.txt system.txt
```

(9).使用"-c"或"-C"检测文件是否排过序。如果已排序，则不返回任何信息，退出状态码为0。如果未排序，退出状态码为1，但"-c"会给出诊断信息，并指明从哪一行开始乱序，而"-C"不返回任何信息。

```bash
[root@xuexi tmp]# sort -c -k3n system.txt ;echo $?
sort: system.txt:3: disorder: 3 bsd     1000    600
1
```

说明system.txt中的第3行开始出现乱序，且退出状态码为1。

```bash
[root@xuexi tmp]# sort -C -k3n system.txt ;echo $?
1
```

<a name="more_sort"></a>

## 深入研究sort

咋一看上去，sort的使用方法很简单，不就是`sort -t DELIM -k POS1,POS2 file`吗，确实如此，它的man文档也才100来行，连info文档加上一堆废话也才500多行。但事实上，sort命令很难，也可以说很简单，简单是因为不管是复杂功能还是简单功能，用来用去就那么几个选项，难是因为没搞懂它的工作机制和细节时，有些时候的结果会比较出人意料，也不知道为什么会如此。

本小节主要讲理论和工作机制的细节，偶尔给出几个示例，所以遇到疑惑时请自行测试，当然也欢迎在博客下方留言。另外，`--debug`(CentOS7才支持该选项)选项对排疑解惑有极大帮助，所以应该善用该选项。

**(1).sort命令默认按照字符集的排序规则进行排序，可以指定"-d"选项按照字典顺序排序，指定"-n"按照数值排序，指定"-M"按照字符格式的月份规则排序，指定"-h"按照文件容量大小规则排序。**

字符集排序规则和字典排序规则对能识别的字符来说，顺序一般是一致的，几种常见字符的顺序为：`空字符串<空白字符<数值<a<A<b<B<...<z<Z`。

指定不同的排序规则，不仅改变排序时的依据，还间接影响排序时的行为，因为不同排序规则能够识别的字符类型不同。至于如何影响，见下面的(4)。

**(2).sort使用"-t"选项指定的分隔符对每行进行分割，得到多个字段，分隔符不作为字段的内容。默认的分隔符为空白字符和非空白字符之间的空字符，并非网上众多文章所说的空格或制表符**(原文：By default, fields are separated by the empty string between a non-blank character and a blank character.)**。**

例如，` foo bar`默认将分隔为两个字段` foo`和` bar`，而使用空格作为分隔符时将分隔为三个字段：第一个字段为空，第二个字段和第三个字段为`foo`和`bar`。使用下面三个sort语句可以验证默认的分隔符并非空格。

```bash
[root@xuexi ~]# echo -e " 234 bar\n 123 car" | sort -t ' ' -b -k3 
 234 bar
 123 car

[root@xuexi ~]# echo -e " 234 bar\n 123 car" | sort -b -k2
 234 bar
 123 car

# -k3指定的字段超出了范围，所以key为空
[root@xuexi ~]# echo -e " 234 bar\n 123 car" | sort -b -k3    
 123 car
 234 bar
```

需要注意的是，分割字段后，字段分隔符不包括在排序目标中。换句话说，**分割后两个字段A和B是紧靠在一起的。当排序的目标字段包含了B字段，那么sort会从字段左对齐处开始依次对字符排序。**实际上sort就是按照字符进行排序的(见下面的-k key的解释)，只不过通过字段进行分隔对人比较友好。

例如，下面的例子：

```bash
[root@xuexi ~]# cat sort.txt
11:1:2
1:1:2
12:1:1
1:1:0
[root@xuexi ~]# sort -t:  sort.txt
1:1:0
11:1:2
1:1:2
12:1:1
```

上面排序例子中，为什么`1:1:2`的1会在11和12中间，而`1:1:0`中的1却在11的前面？实际上，真正排序的时候，sort看到的内容是这样的：

```
1112
112
1211
110
```

这样一来，结果不言而明。**当明确指定排序字段时，sort会计算每一行的排序起始字符和排序结束字符**。例如：


```bash
[root@xuexi ~]# sort -t: -k1,1  sort.txt
1:1:0        # 取 1  作为排序目标
1:1:2        # 取 1  作为排序目标
11:1:2       # 取 11 作为排序目标
12:1:1       # 取 12 作为排序目标
[root@xuexi ~]# sort -t: -k1,2  sort.txt 
1:1:0        # 取 11  作为排序目标
1:1:2        # 取 11  作为排序目标
11:1:2       # 取 111 作为排序目标
12:1:1       # 取 121 作为排序目标
```


当然，默认情况下sort在对排序key指定的字符排完序后，再做一次最终排序。见下面的第(5)点说明。

**(3).使用"-k"选项指定排序的key。不指定排序key时，整行将成为排序key，即对整行进行排序。**

- key由字段组成，格式为`POS1,[POS2]`，表示每行排序的起始和终止位置。也就是说，key才是排序的对象。
- POS的格式为`F[.C][OPTS]`，其中F表示字段的序号，C表示该字段中字符的序号。字段和字符的位置都从1开始计算。如果POS2的字符位置指定为0，则表示POS2字段中的最后一个字符。如果POS1中省略".C"，则默认值为1(字段的起始字符)，如果POS2中省略".C"，默认值为0(字段的终止字符)。使用"-b"选项忽略前导空白字符时，C从第一个非空白字符开始计算。如果F或C超出了有效范围，则该key为空，例如一行只有3个字段，却指定了"-k4"，或者第2字段只有3个字符，却指定了`-k2.5`。
- 如果省略POS2，则key将自动扩展到行尾，即等价于`POS1,line_end`。如果不省略POS2，则该key可能会跨越多个字段。无论那种情况，跨越多个字段时，key中都不会保留字段间的分隔符。
- OPTS指定的是该key的选项，包括但不限于`bfnrhM`，它们的作用和全局选项`-b -f -n -r -h -M`相同。默认情况下，如果key中没有指定任何OPTS，则该key会继承全局选项。当key中单独指定了选项时，这些选项是该key的私有排序选项，将覆盖全局选项。除了"b"选项外，其余选项无论是指定在POS1还是POS2中都是等价的，对于"b"选项，指定在POS1则作用于POS1，指定在POS2则作用于POS2。如果继承了全局选项"-b"，则作用于POS1和POS2。
- 字段前数量不固定的前导空白字符，将使得字段混乱，因此强烈建议总是忽略前导空白字符。数值排序时(即"n"选项)隐含"b"选项。
- 可以使用多个"-k"选项指定多个key，排序时将按照key的顺序进行排序。第一个key通常称为主排序key(primary key)。第二个key将在第一个key排序的基础上排序，同理，第三个key将在第二个key的排序基础上进行排序。

以下是几个例子：例子中出现了选项"n"的，描述暂不严谨，但目前只能如此描述，在稍后的(4)中解释。

- `-k 2`：因为没有指定POS2，所以key扩展到了行尾。因此该key从第2字段第一个字符开始，到行尾结束。
- `-k 2,3`：该key从第2字段第一个字符开始到第3字段最后一个字符结束。
- `-k 2,2`：该key仅拥有第2字段。
- `-k 2,3n`和`-k 2n,3`和`-k 2n,3n`：这三者等价，因为除了"b"选项，OPTS指定在POS1或POS2的结果是一样的。
- `-k 2,3b`和`-k 2b,3`和`-k 2b,3b`：这三者互不等价。
- `-k 2n`：该key从第2字段开始直到行尾，都按数值排序。
- `-k 2.2b,3.2n`：该key从第2字段的第2个非空白字符开始，到第3字段第2字符(可能包含空白字符)结束，且该key按照数值排序。其实此处的b选项是多余的，因为n隐含了b选项。
- `-k 5b,5 -k 3,3n`：定义了两个排序key，主排序key为第5字段不包含空白字符的部分，副key为第三个字段。主key按照默认规则排序，副key按照数值排序。副key在主key排序后的基础上再排序。
- `-k 5,5n -k 3b,6b`：主key为第5字段，按照数值排序，副key从第3字段到第六字段，忽略前导空白字符，但是按照默认规则排序。副key在主key排序后的基础上再排序。

**(4).当排序规则选项(例如`n、d、M、h`)发现不识别的符号时，将立即结束当前key的排序。默认排序规则是字符集的排序规则，通常能识别所有字符，所以总会对整个key进行完整的排序。这是"何时跨字段、跨key比较？"的问题。**

例如，指定n选项按数值排序时，由于"n"选项只能识别数字和负号"-"，当排序时遇到无法识别字符时，将导致该key的排序立即结束。也就是说，对于`abc 123 456 abc`这样的输入，分隔符为空格，当指定`-k 2,3n`时，虽然排序key包括`123 456`，但由于中间的空白字符无法被n识别，使得在排完第2字段"123"时就立即结束该key的排序。

正因如此，使得n选项绝对不会跨字段、跨key进行比较。因此，`-k 2,3n`和`-k 2n`、`-k 2,2n`、`-k 2,4n`的结果是等价的，都只对第2字段按照数值进行排序。但默认的排序规则不会有这样的问题，因为默认排序规则能识别所有字符，也就是说`-k 2,3`、`-k 2`、`-k 2,2`、`-k 2,4`是互不等价的。

同理，"-d"的字典排序规则只能识别字母、数字和空白字符，所以遇到非这3类字符时也将立即结束当前key的排序。"-h"和"-M"也都有字符的识别限制，处理方式也一样。关于"-h"和"-M"选项的说明，见info sort。

需要特意说明的是：n同样不识别空字符串，发现空字符串时也结束排序。这可能会适得按数值排序的结果出人意料。例如：

```bash
[root@xuexi ~]# echo -e "b 100:200 200\na 110 300" | tr ':' '\0'|sort -t ' ' -k2n
b 100200 200
a 110 300
```

对于`b 100\0200 200`这样的行，`-k 2n`使得该key为`100\0200`。虽然结果看上去是100200，但却只对100进行排序，也就是说它小于110。这就造成了数值排序的假象，100200竟然比110小。

**(5).默认情况下，sort会进行一次"最后的排序"。使用"-s"选项将禁止"最后的排序"，"-u"选项隐含"-s"选项。**

考虑这样一种情况：两行在所有key的排序结果上都完全相同，应该如何决定这两行的先后顺序？

例如：

```bash
[root@xuexi ~]# echo -e "b 100 200\na 100 300" | sort -t ' ' -k2n
a 100 300
b 100 200
```

第一行为`b 100 200`，第二行为`a 100 300`。由于第2字段都是100，所以这两行在该key上的数值排序的结果相同，**于是sort采取最后的手段，完全按照默认规则(即按字符集排序规则升序排序)对整行进行一次排序，这次排序称为"最后的排序"**(info sort中称为last-resort comparison)。由于最后的排序过程中，第一个字符a<b，所以最终结果将是第二行`a 100 300`在第一行`b 100 200`的前面。

**禁止"最后的排序"后，对那些排序key相同的行，将保留被读取时相对顺序。**即，先读取的排在前面。

如果上面的例子中，第二字段不采用数值排序，而是默认排序规则排序呢？如下：

```bash
[root@xuexi ~]# echo -e "b 100 200\na 100 300" | sort -t ' ' -k2
b 100 200
a 100 300
```

由于默认的排序规则是按照字符集排序规则进行排序，它能识别所有的字符，所以会对"-k2"整个key进行排序，该key会自动扩展为第2字段和第3字段，由于第三字段的2小于3，所以结果中第一行排在第二行的前面。即使如此，sort还是进行了"最后的排序"，只不过"最后的排序"不影响排序结果。

如果未指定任何排序选项，其本身就是完全默认的，因此没必要再做最后的排序，所以将不会进行"最后的排序"。如果指定的是"-r"选项，由于"-r"是对最终结果进行反转排序，因此会影响这次的"最后的排序"的结果。

**(6).sort的使用建议。**

搞清楚了以上几点，是否感觉sort能实现几乎所有的排序需求呢？只要文件够规则，sort就能控制任何一列或多列的排序方式，并且可以设置出是否跨列、跨字符、跨key排序。

这里有几个sort使用建议，算是最后的补充。

- 任何时候想对单个字段或单个字符排序时，都建议写出POS2，且`POS2=POS1`，这样能严格排序key的范围只为那个字段或字符。例如，使用`-k2,2`取代`-k2`。
- 想对多个字段或字符排序时，建议使用多个"-k"选项指定多个key，并按需求为每个key分配私有选项。之所以要如此，是防止无意中忽视了扩展到行尾或者范围。例如，想对第2列、第3列按数值排序，应该指定`-k2n -k3n`，而不应该写成`-k2,3n`。
- 应该总是使用"-b"选项去掉前导空白字符面，防止字段分割时混乱。"-n"隐含了"-b"，所以对数值排序时，可以省略"-b"。
- 对于大文件，建议写出满足需求的所有排序命令，然后使用"-s"关闭"最后的排序"。因为"最后的排序"对每个整行进行排序，性能非常低。


最后，给出一个测试题：假设一些待排序的日志文件中的内容格式如下：
```
4.150.156.3 - - [01/Apr/2004:06:31:51 +0000] message 1
211.24.3.231 - - [24/Apr/2004:20:17:39 +0000] message 2
```

能否理解下面两条等价的命令？

```bash
sort -s -t ' ' -k 4.9n -k 4.5M -k 4.2n -k 4.14,4.21 file*.log | sort -s -t '.' -k 1,1n -k 2,2n -k 3,3n -k 4,4n

sort -s -t ' ' -k 4.9n -k 4.5M -k 4.2n -k 4.14,4.21 file*.log | sort -s -t '.' -n -k1 -k2 -k3 -k4
```

