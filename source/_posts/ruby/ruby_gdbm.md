---
title: Ruby使用GDBM
p: ruby/ruby_gdbm.md
date: 2020-05-17 22:55:05
tags: Ruby
categories: Ruby
---

------

**[回到Ruby系列文章](/ruby/index/)**

------

# DBM简介

![](/img/ruby/1589766485125.png)

## DBM数据存储原理

1. dbm的数据存取基于key-value的hash格式  
2. dbm/ndbm中，key单独存放在一个文件中，key+value存放在另一个文件中。对于gdbm，则是key作为索引数据单独存放在db文件中的一个地方(索引区)，key+value存放在db文件中的另一个地方(数据区)  
3. 为了高效率查询，除了key作为索引单独存放外，还额外存放key+value在db文件中的偏移位置以及大小，使得可以直接seek()跳转到指定位置处读取指定大小的数据  
4. 删除记录时，只是删除索引区的key，数据区的key+value不方便删除也没必要删除，数据区这段孤儿空间称为保留空间或碎片空间，可以作为空闲空间留待后续复用  
5. gdbm在执行读取操作之后会将数据缓存下来，因此，第一次读取可能速度慢，但是第二次速度将非常快。keys、values等操作的的结果也都会被缓存下来  
6. 因为碎片空间可被复用，所以dbm还会记录所有的碎片空间的位置以及大小，gdbm中以链表方式记录之   
7. 因删除记录不会释放空间，所以db文件大小不会减小。换句话说，dbm的文件会随着时间的推移不断增大，除非重组dbm，重组时，将根据索引区存在的key找到数据区所有对应的key+value数据，并将它们写入临时文件，最后重命名覆盖原db文件  
8. 插入数据时，如果没有碎片空间，默认将插入在尾部，如果中间有碎片空间，则判断待写入数据的大小是否能够插入在碎片空间中  
9. 更新数据时，如果更新后的数据变大，且该数据后面没有碎片空间，则直接原地移除并在文件尾部插入更新后的数据，如果有足够的空间存放更新后的数据，则原地更新  
10. dbm只能存储字符串，数值、布尔、对象等都不能直接存储  

## Ruby使用gdbm

Ruby中要使用gdbm，它依赖于gdbm扩展库和头文件，所以需先安装：

```bash
# sudo yum install gdbm-devel
# Windows：
#   ridk exec uname -a确定32位还是64位，
#   然后ridk exec pacman -S mingw-w64-<$arch>-gdbm
sudo apt install libgdbm-dev
gem install gdbm
```

使用类方法`GDBM.new()`或者`GDBM.open()`可打开gdbm来操作db文件。

```ruby
require 'gdbm'

gdbm = GDBM.new("/tmp/lang.db")
gdbm["perl"] = "Perl"
gdbm["shell"] = "Shell"
gdbm["php"] = "PHP"
gdbm.close
```

查看其文件内容：

```bash
$ ls -l /tmp/lang.db
-rw-rw-rw- 1 longshuai longshuai 8192 May 17 21:22 /tmp/lang.db

$ cat /tmp/lang.db
P |...x...l9php...rdshe...}N;iperl...perlPerlshellShellphpPHP
```

其中...表示的是乱码部分。

注意其大小为8K，且数据区默认在db文件的尾部，包含了key和value。

从db中检索数据：

```ruby
gdbm = GDBM.open("/tmp/lang.db")
pp gdbm["perl"]
pp gdbm["php"]
gdbm.close
```

## new()、open()

new()或open()：open()可给定语句块，语句块退出时自动关闭IO流，未给定语句块时，open()等价于new()。

```
new(filename, mode = 0666, flags = nil)
open(filename, mode = 0666, flags = nil)
open(filename, mode = 0666, flags = nil) { |gdbm| ... }
```

当指定要操作的db文件不存在时，会创建文件，可指定创建文件时的权限。此外，flag参数接受如下值：

```
### 注意：writer方式可读可写
READER  - 以只读方式打开，即返回一个reader
WRITER  - 以可读写方式打开，即返回一个writer
WRCREAT - writer，如果数据库文件不存在，则创建
NEWDB   - writer，总是截断覆盖已存在的数据库文件

# 上面的三个writer可使用位或(|)的方式结合下面的选项：
SYNC   - 以sync模式写入数据库文件
NOLOCK - 打开时不锁定数据库文件
```

在未给定任何选项时，即默认情况下，总是先尝试以WRCREAT的方式打开，即以writer打开且文件不存在时创建。但如果打开失败(比如另一个进程已经打开且还未关闭)，则尝试使用reader方式打开。

reader和reader之间互相兼容，writer和writer之间以及writer和reader之间互斥。所以，在某一时刻，**允许同时有多个reader，但只能有一个writer**。

当打开gdbm实例后，它可以按照操作hash结构的方式去操作db，此外，**gdbm已经mix-in Enumerable模块**，所以可以直接使用该模块中的一些方法，比如find、grep、map等。

## gdbm方法

```
######### 查询、插入、更新 #########
["key"]
fetch(key [, default]) → value
检索指定的key。
使用`[]`检索时，如果key不存在将返回nil，
使用fetch检索时，如果key不存在则报错，或者返回指定的默认值

values_at(key, ...) → array
检索一个或多个key，并以数组方式返回对应的value

["key"]= value
store(key, value) → value
更新指定的key，如果key不存在则插入

########## 遍历 #########
each_pair { |key, value| block } → gdbm
each_key { |key| block } → gdbm
each_value { |value| block } → gdbm
分别根据key-value、key、value遍历db

######### 其它检索、筛选方式 #########
key(value) → key
根据value找到其key，如果有多个相同的value，返回第一个

keys → array
以数组方式返回db中所有的key

values → array
以数组方式返回所有value

select { |key, value| block } → array
筛选所有满足条件的key-value

######### 判断key或value是否存在 #########
has_key?(k) → true or false
include?(k) → true or false
key?(k) → true or false
member?(k) → true or false
判断key是否存在

has_value?(v) → true or false
value?(v) → true or false
判断指定的value是否存在

######### 删除 #########
delete(key) → value or nil
根据key移除key-value并返回被移除的Key-value，db若空，返回nil

shift → (key, value) or nil
移除指定的key-value，并以数组方式返回之，db若空，则返回nil

delete_if { |key, value| block } → gdbm
移除满足条件(语句块返回true)的key-value，直接修改gdbm

reject { |key, value| block } → hash
等价于delete_if，但不修改gdbm，而是以hash的方式返回

reject! { |key, value| block } → gdbm
等价于delete_if，直接修改gdbm

clear → gdbm
清空db中所有key-value

######## 大小判断 #########
empty? → true or false
db是否为空

length → fixnum
size → fixnum
等价，返回db中的key-value数量

####### 其它操作 #########
invert → hash
反转gdbm中key-value：key作为value，value作为key，并以hash的方式返回

close → nil
关闭已打开的db文件

closed? → true or false
判断db文件是否已关闭

replace(other) → gdbm
将另一个gdbm(即other)的内容覆盖替换到当前的gdbm

update(other) → gdbm
用另一个gdbm(即Other)合并到当前gdbm，若key冲突，则当前gdbm的key被覆盖

reorganize → gdbm
重组gdbm

cachesize = size → size
设置gdbm内部的hash桶缓存大小

######## gdbm模式 #########
sync → gdbm
将IO buffer中的数据刷入磁盘中的db文件，全部写入成功才返回
如果以SYNC标记打开，则无需sync()

fastmode = boolean → boolean
syncmode = boolean → boolean
打开或关闭sync模式。
sync模式下，写入操作需要写入磁盘db文件成功(或失败)后才返回，
非sync模式下，只需写入io buffer即可返回。
syncmode方法在gdbm >= 1.8才可用，在此版本之前，使用方法fastmode=

######### 转换 #########
to_a → array
to_hash → hash
转换为数组、转换为hash
```

