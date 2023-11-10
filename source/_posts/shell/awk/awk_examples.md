---
title: 18个awk的经典实战案例
p: shell/awk/awk_examples.md
date: 2019-11-27 10:37:36
tags: Awk
categories: Awk
---

--------

**回到：**  
- **[Linux系列文章](/linux/index)**  
- **[Shell系列文章](/shell/index)**  
- **[Awk系列文章](/shell/awk/index)**  

--------


# 介绍

这些案例是我收集起来的，大多都是我自己遇到过的，有些比较经典，有些比较具有代表性。

这些awk案例我也录了相关视频的讲解[awk 18个经典实战案例精讲](https://edu.51cto.com/sd/0dd13)，欢迎大家去瞅瞅。

## awk插入几个新字段

在"a b c d"的b后面插入3个字段`e f g`。

```shell
echo a b c d|awk '{$3="e f g "$3}1'
```

## awk格式化空白

移除每行的前缀、后缀空白，并将各部分左对齐。

```
      aaaa        bbb     ccc                 
   bbb     aaa ccc
ddd       fff             eee gg hh ii jj
```

```shell
awk 'BEGIN{OFS="\t"}{$1=$1;print}' a.txt
```

执行结果：

```
aaaa    bbb     ccc
bbb     aaa     ccc
ddd     fff     eee     gg      hh      ii      jj
```

## awk筛选IPv4地址

从ifconfig命令的结果中筛选出除了lo网卡外的所有IPv4地址。

![](/img/shell/awk/1588919571011.png)

## awk读取.ini配置文件中的某段

```
[base]
name=os_repo
baseurl=https://xxx/centos/$releasever/os/$basearch
gpgcheck=0

enable=1

[mysql]
name=mysql_repo
baseurl=https://xxx/mysql-repo/yum/mysql-5.7-community/el/$releasever/$basearch

gpgcheck=0
enable=1

[epel]
name=epel_repo
baseurl=https://xxx/epel/$releasever/$basearch
gpgcheck=0
enable=1
[percona]
name=percona_repo
baseurl = https://xxx/percona/release/$releasever/RPMS/$basearch
enabled = 1
gpgcheck = 0
```

![](/img/shell/awk/733013-20191127171517240-2019317102.jpg)


## awk根据某字段去重

去掉`uid=xxx`重复的行。

```
2019-01-13_12:00_index?uid=123
2019-01-13_13:00_index?uid=123
2019-01-13_14:00_index?uid=333
2019-01-13_15:00_index?uid=9710
2019-01-14_12:00_index?uid=123
2019-01-14_13:00_index?uid=123
2019-01-15_14:00_index?uid=333
2019-01-16_15:00_index?uid=9710
```

```shell
awk -F"?" '!arr[$2]++{print}' a.txt
```

结果：

```
2019-01-13_12:00_index?uid=123
2019-01-13_14:00_index?uid=333
2019-01-13_15:00_index?uid=9710
```

## awk次数统计

```
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
```

```shell
awk '{arr[$1]++}END{OFS="\t";for(idx in arr){printf arr[idx],idx}}' a.txt
```

## awk统计TCP连接状态数量

```
$ netstat -tnap
Proto Recv-Q Send-Q Local Address   Foreign Address  State       PID/Program name
tcp        0      0 0.0.0.0:22      0.0.0.0:*        LISTEN      1139/sshd
tcp        0      0 127.0.0.1:25    0.0.0.0:*        LISTEN      2285/master
tcp        0     96 192.168.2.17:22 192.168.2.1:2468 ESTABLISHED 87463/sshd: root@pt
tcp        0      0 192.168.2017:22 192.168.201:5821 ESTABLISHED 89359/sshd: root@no
tcp6       0      0 :::3306         :::*             LISTEN      2289/mysqld
tcp6       0      0 :::22           :::*             LISTEN      1139/sshd
tcp6       0      0 ::1:25          :::*             LISTEN      2285/master
```

统计得到的结果：
```
5: LISTEN
2: ESTABLISHED
```

![](/img/shell/awk/733013-20191127171805677-1264614779.jpg)

一行式：
```
netstat -tna | awk '/^tcp/{arr[$6]++}END{for(state in arr){print arr[state] ": " state}}'
netstat -tna | /usr/bin/grep 'tcp' | awk '{print $6}' | sort | uniq -c
```

## awk统计日志中各IP访问非200状态码的次数

日志示例数据：

```
111.202.100.141 - - [2019-11-07T03:11:02+08:00] "GET /robots.txt HTTP/1.1" 301 169 
```

统计非200状态码的IP，并取次数最多的前10个IP。

```shell
# 法一
awk '$8!=200{arr[$1]++}END{for(i in arr){print arr[i],i}}' access.log | sort -k1nr | head -n 10

# 法二：
awk '
    $8!=200{arr[$1]++}
    END{
        PROCINFO["sorted_in"]="@val_num_desc";
        for(i in arr){
            if(cnt++==10){exit}
            print arr[i],i
        }
}' access.log
```

## awk统计独立IP
​     url     访问IP       访问时间      访问人

```
a.com.cn|202.109.134.23|2015-11-20 20:34:43|guest
b.com.cn|202.109.134.23|2015-11-20 20:34:48|guest
c.com.cn|202.109.134.24|2015-11-20 20:34:48|guest
a.com.cn|202.109.134.23|2015-11-20 20:34:43|guest
a.com.cn|202.109.134.24|2015-11-20 20:34:43|guest
b.com.cn|202.109.134.25|2015-11-20 20:34:48|guest
```

需求：统计每个URL的独立访问IP有多少个(去重)，并且要为每个URL保存一个对应的文件，得到的结果类似：

```
a.com.cn  2
b.com.cn  2
c.com.cn  1
```
并且有三个对应的文件：
```
a.com.cn.txt
b.com.cn.txt
c.com.cn.txt
```

代码：

![](/img/shell/awk/733013-20191127172048213-524433672.jpg)

## 输出第二字段重复的所有整行

如下文本内容

```
1 zhangsan
2 lisi
3 zhangsan
4 lisii
5 a
6 b
7 c
8 d
9 a
10 b
```

要求：输出第二列重复的所有整行，即输出结果：

```
1 zhangsan
3 zhangsan
5 a
9 a
6 b
10 b
```

代码：

```awk
awk '{
    arr[$2]++;
    if(arr[$2]>1){
      if(arr[$2]==2){
        print first[$2]
      };
      print $0
    }else{
      first[$2]=$0
    }
}' a.txt
```


## 相邻重复行去重，并保留最后一行

根据字段进行比较，去除相邻的重复行，并保留重复行中的最后一行以及那些非重复行。

a.log内容：

```
TCP 10.33.4.149:19404 wrr
-> 10.27.4.197:19404 FullNat 10 2 0
TCP 10.33.4.150:19039 wrr
TCP 10.33.4.150:19089 wrr
-> 10.27.4.201:19089 FullNat 10 2 0
TCP 10.33.4.150:19094 wrr
TCP 10.33.4.150:19102 wrr
TCP 10.33.4.150:19107 wrr
-> 10.27.100.150:19107 FullNat 10 18 0
TCP 10.33.4.150:19111 wrr
TCP 10.33.4.150:19112 wrr
TCP 10.33.4.150:19113 wrr
TCP 10.33.4.150:19114 wrr
TCP 10.33.4.150:19207 wrr
-> 10.27.100.150:19207 FullNat 10 18 0
```

以第一字段判断重复，去重相邻行，最终输出结果：

```
TCP 10.33.4.149:19404 wrr
-> 10.27.4.197:19404 FullNat 10 2 0
TCP 10.33.4.150:19089 wrr
-> 10.27.4.201:19089 FullNat 10 2 0
TCP 10.33.4.150:19107 wrr
-> 10.27.100.150:19107 FullNat 10 18 0
TCP 10.33.4.150:19207 wrr
-> 10.27.100.150:19207 FullNat 10 18 0
```

方案：

```shell
# 用第一个字段比较。如果想按其他字段比较，换成$N，N为对应的字段号
awk '
{
  if($1!=prev){
    a[++n]=$0;
  }else{
    a[n]=$0
  }
}
{prev=$1}
END{
  for(i=1;i<=length(a);i++){print a[i]}
}
' a.log
```




## awk处理字段缺失的数据

```
ID  name    gender  age  email          phone
1   Bob     male    28   abc@qq.com     18023394012
2   Alice   female  24   def@gmail.com  18084925203
3   Tony    male    21                  17048792503
4   Kevin   male    21   bbb@189.com    17023929033
5   Alex    male    18   ccc@xyz.com    18185904230
6   Andy    female       ddd@139.com    18923902352
7   Jerry   female  25   exdsa@189.com  18785234906
8   Peter   male    20   bax@qq.com     17729348758
9   Steven          23   bc@sohu.com    15947893212
10  Bruce   female  27   bcbd@139.com   13942943905
```

当字段缺失时，直接使用FS划分字段来处理会非常棘手。gawk为了解决这种特殊需求，提供了FIELDWIDTHS变量。

FIELDWIDTH可以按照字符数量划分字段。

```shell
awk '{print $4}' FIELDWIDTHS="2 2:6 2:6 2:3 2:13 2:11" a.txt
```

## awk处理字段中包含了字段分隔符的数据

下面是CSV文件中的一行，该CSV文件以逗号分隔各个字段。

```
Robbins,Arnold,"1234 A Pretty Street, NE",MyTown,MyState,12345-6789,USA
```

需求：取得第三个字段"1234 A Pretty Street, NE"。

当字段中包含了字段分隔符时，直接使用FS划分字段来处理会非常棘手。gawk为了解决这种特殊需求，提供了FPAT变量。

FPAT可以收集正则匹配的结果，并将它们保存在各个字段中。（就像grep匹配成功的部分会加颜色显示，而使用FPAT划分字段，则是将匹配成功的部分保存在字段`$1 $2 $3...`中）。

```shell
echo 'Robbins,Arnold,"1234 A Pretty Street, NE",MyTown,MyState,12345-6789,USA' |\
awk 'BEGIN{FPAT="[^,]+|\".*\""}{print $1,$3}'
```

## awk取字段中指定字符数量

```
16  001agdcdafasd
16  002agdcxxxxxx
23  001adfadfahoh
23  001fsdadggggg
```

得到：

```
16  001
16  002
23  001
23  002
```

```shell
awk '{print $1,substr($2,1,3)}'
awk 'BEGIN{FIELDWIDTH="2 2:3"}{print $1,$2}' a.txt
```

## awk行列转换

```
name age
alice 21
ryan 30
```

转换得到：

```
name alice ryan
age 21 30
```

```shell
awk '
    {
      for(i=1;i<=NF;i++){
        if(!(i in arr)){
          arr[i]=$i
        } else {
            arr[i]=arr[i]" "$i
        }
      }
    }
	END{
        for(i=1;i<=NF;i++){
            print arr[i]
        }
	}
' a.txt
```

## awk行列转换2

文件内容：
```
74683 1001
74683 1002
74683 1011
74684 1000
74684 1001
74684 1002
74685 1001
74685 1011
74686 1000
....
100085 1000
100085 1001
```

文件就两列，希望处理成
```
74683 1001 1002 1011
74684 1000 1001 1002
...
```

就是只要第一列数字相同， 就把他们的第二列放一行上，中间空格分开

```shell
{
  if($1 in arr){
    arr[$1] = arr[$1]" "$2
  } else {
    arr[$1] = $2
  }
  
}

END{
  for(i in arr){
    printf "%s %s\n",i,arr[i]
  }
}
```

## awk筛选给定时间范围内的日志

grep/sed/awk用正则去筛选日志时，如果要精确到小时、分钟、秒，则非常难以实现。

但是awk提供了mktime()函数，它可以将时间转换成epoch时间值。

```shell
# 2019-11-10 03:42:40转换成epoch
$ awk 'BEGIN{print mktime("2019 11 10 03 42 40")}'
1573328560
```

借此，可以取得日志中的时间字符串部分，再将它们的年、月、日、时、分、秒都取出来，然后放入mktime()构建成对应的epoch值。因为epoch值是数值，所以可以比较大小，从而决定时间的大小。

下面strptime1()实现的是将`2019-11-10T03:42:40+08:00`格式的字符串转换成epoch值，然后和which_time比较大小即可筛选出精确到秒的日志。

![](/img/shell/awk/733013-20191127172424429-1080986345.jpg)

下面strptime2()实现的是将`10/Nov/2019:23:53:44+08:00`格式的字符串转换成epoch值，然后和which_time比较大小即可筛选出精确到秒的日志。

```
BEGIN{
  # 要筛选什么时间的日志，将其时间构建成epoch值
  which_time = mktime("2019 11 10 03 42 40")
}

{
  # 取出日志中的日期时间字符串部分
  match($0,"^.*\\[(.*)\\].*",arr)
  
  # 将日期时间字符串转换为epoch值
  tmp_time = strptime2(arr[1])
  
  # 通过比较epoch值来比较时间大小
  if(tmp_time > which_time){
    print 
  }
}

# 构建的时间字符串格式为："10/Nov/2019:23:53:44+08:00"
function strptime2(str   ,dt_str,arr,Y,M,D,H,m,S) {
  dt_str = gensub("[/:+]"," ","g",str)
  # dt_sr = "10 Nov 2019 23 53 44 08 00"
  split(dt_str,arr," ")
  Y=arr[3]
  M=mon_map(arr[2])
  D=arr[1]
  H=arr[4]
  m=arr[5]
  S=arr[6]
  return mktime(sprintf("%s %s %s %s %s %s",Y,M,D,H,m,S))
}

function mon_map(str   ,mons){
  mons["Jan"]=1
  mons["Feb"]=2
  mons["Mar"]=3
  mons["Apr"]=4
  mons["May"]=5
  mons["Jun"]=6
  mons["Jul"]=7
  mons["Aug"]=8
  mons["Sep"]=9
  mons["Oct"]=10
  mons["Nov"]=11
  mons["Dec"]=12
  return mons[str]
}
```

## awk去掉`/**/`中间的注释

示例数据：

```
/*AAAAAAAAAA*/
1111
222

/*aaaaaaaaa*/
32323
12341234
12134 /*bbbbbbbbbb*/ 132412

14534122
/*
    cccccccccc
*/
xxxxxx /*ddddddddddd
    cccccccccc
    eeeeeee
*/ yyyyyyyy
5642341
```

![](/img/shell/awk/733013-20191127172624494-1597318447.jpg)


## awk前后段落关系判断

从如下类型的文件中，找出false段的前一段为i-order的段，同时输出这两段。

```
2019-09-12 07:16:27 [-][
  'data' => [
    'http://192.168.100.20:2800/api/payment/i-order',
  ],
]
2019-09-12 07:16:27 [-][
  'data' => [
    false,
  ],
]
2019-09-21 07:16:27 [-][
  'data' => [
    'http://192.168.100.20:2800/api/payment/i-order',
  ],
]
2019-09-21 07:16:27 [-][
  'data' => [
    'http://192.168.100.20:2800/api/payment/i-user',
  ],
]
2019-09-17 18:34:37 [-][
  'data' => [
    false,
  ],
]
```

```
BEGIN{
  RS="]\n"
  ORS=RS
}
{
  if(/false/ && prev ~ /i-order/){
    print tmp
    print
  }
  tmp=$0
}
```

递归正则搜索：

```bash
grep -Pz '(?s)\d+((?!2019).)*i-order(?1)+\d+(?1)+false(?1)+'
```

## awk两个文件的处理

有两个文件file1和file2，这两个文件格式都是一样的。

需求：先把文件2的第五列删除，然后用文件2的第一列减去文件一的第一列，把所得结果对应的贴到原来第五列的位置，请问这个脚本该怎么编写？

```
file1：
50.481  64.634  40.573  1.00  0.00
51.877  65.004  40.226  1.00  0.00
52.258  64.681  39.113  1.00  0.00
52.418  65.846  40.925  1.00  0.00
49.515  65.641  40.554  1.00  0.00
49.802  66.666  40.358  1.00  0.00
48.176  65.344  40.766  1.00  0.00
47.428  66.127  40.732  1.00  0.00
51.087  62.165  40.940  1.00  0.00
52.289  62.334  40.897  1.00  0.00
file2：
48.420  62.001  41.252  1.00  0.00
45.555  61.598  41.361  1.00  0.00
45.815  61.402  40.325  1.00  0.00
44.873  60.641  42.111  1.00  0.00
44.617  59.688  41.648  1.00  0.00
44.500  60.911  43.433  1.00  0.00
43.691  59.887  44.228  1.00  0.00
43.980  58.629  43.859  1.00  0.00
42.372  60.069  44.032  1.00  0.00
43.914  59.977  45.551  1.00  0.00
```

![](/img/shell/awk/733013-20191127172748233-790426259.jpg)

<a name="subarray"></a>

## 统计多项数据

如下内容，第一个字段是IP，第二个字段是每个访问的uri。

```
1.1.1.1 /index1.html
1.1.1.1 /index1.html
1.1.1.1 /index2.html
1.1.1.1 /index2.html
1.1.1.1 /index2.html
1.1.1.1 /index3.html
1.1.1.2 /index1.html
1.1.1.2 /index2.html
1.1.1.2 /index2.html
1.1.1.3 /index1.html
1.1.1.3 /index1.html
1.1.1.3 /index2.html
1.1.1.3 /index2.html
1.1.1.3 /index2.html
1.1.1.3 /index3.html
1.1.1.3 /index3.html
1.1.1.4 /index2.html
1.1.1.4 /index2.html
```

要求统计出每个ip访问的总次数，以及每个ip所访问的uri的次数。

期望的输出结果：

```
1.1.1.1 6 /index3.html 1
1.1.1.1 6 /index2.html 3
1.1.1.1 6 /index1.html 2
1.1.1.2 3 /index2.html 2
1.1.1.2 3 /index1.html 1
1.1.1.3 7 /index3.html 2
1.1.1.3 7 /index2.html 3
1.1.1.3 7 /index1.html 2
1.1.1.4 2 /index2.html 2
```

方法1，使用子数组。awk代码：

```bash
awk '
  {
    a[$1][$2]++
  }
  END{
    # 遍历数组，统计每个ip的访问总数
    for(ip in a){
      for(uri in a[ip]){
        b[ip] += a[ip][uri]
      }
    }
    
    # 再次遍历
    for(ip in a){
      for(uri in a[ip]){
        print ip, b[ip], uri, a[ip][uri]
      }
    }
  }
' a.log
```

方法2，使用复合索引的数组。awk代码：

```bash
awk '
  {
    a[$1]++
    b[$1"_"$2]++
  }
  END{
    for(i in b){
      split(i,c,"_");
      print c[1],a[c[1]],c[2],b[i]
    }
}' a.log
```

## 处理段落

文件内容如下：

```
{ "ent_id" : MinKey, "_id" : MinKey } -->> {
        "ent_id" : NumberLong("aaaaa"),
        "_id" : ObjectId("bbbbb")
} on : shard04 Timestamp(685, 0)
{
        "ent_id" : NumberLong("ccccc"),
        "_id" : ObjectId("ddddd")
} -->> {
        "ent_id" : NumberLong("eeeee"),
        "_id" : ObjectId("fffff")
} on : shard04 Timestamp(331, 1)
{
        "ent_id" : NumberLong("ggggg"),
        "_id" : ObjectId("hhhhh")
} -->> {
        "ent_id" : NumberLong("iiiii"),
        "_id" : ObjectId("jjjjj")
} on : shard04 Timestamp(680, 0)
```

期望结果：

```
MinKey,MinKey,NumberLong("aaaaa"),ObjectId("bbbbb"),shard04
NumberLong("ccccc"),ObjectId("ddddd"),NumberLong("eeeee"),ObjectId("fffff"),shard04
NumberLong("ggggg"),ObjectId("hhhhh"),NumberLong("iiiii"),ObjectId("jjjjj"),shard04
```

awk代码：

```awk
BEGIN{
  # 以Timestamp...为输入记录分隔符，一次读取一段
  RS=" Timestamp\\([0-9]+, [0-9]\\)"
}
{
  # 将一段中所有冒号后的内容保存到数组
  patsplit($0,arr,": ([0-9a-zA-Z\"\\(\\)])+")
  for(i in arr){
    # 移除冒号，并使用逗号分隔串联各元素
    str = str gensub(": ","","g",arr[i])","
  }
  # 移除尾部逗号
  print(substr(str,1,length(str)-1))
  str=""
}
```

使用Perl或Ruby则更简单：

```
perl -0nE 'BEGIN{$,=","}say $& =~ /: \K[^\s,]+/g while /{.*?} on : \S+/sg' test.log
ruby -ne 'BEGIN{$/=nil};$_.scan(/{.*?} on : \S+/m){|s|puts s.scan(/: \K[^\s,]+/).join(",")}' test.log
```

