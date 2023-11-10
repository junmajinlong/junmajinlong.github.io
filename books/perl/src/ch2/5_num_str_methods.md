## 数值和字符串常见操作函数

数值类的常用函数比较少，包括：  

- int()：截断为整数  
- rand()：返回指定范围内的随机数，不指定范围，则返回`[0,1)`之间的随机数  

例如：

```perl
say int(3.78);  # 3
say rand();     # 返回0到1之间的随机数：0.25324216212994
say rand(1.5);  # 返回0到1.5之间的随机数：1.21238530987939
say rand(10);   # 返回0到10之间的随机数：9.61505077404098

# 要获取随机整数，加上int()
say int(rand(10)); # 返回0到10之间的随机整数：8
```

字符串类的常用函数较多，包括但不限于如下所列函数：

```
# 大小写转换类函数
lc：(lower case)将后面的字母转换为小写，是\L的实现
uc：(uppercase)将后面的字母转换为大写，是\U的实现
fc：(foldcase)和lc基本等价，但fc可以处理UTF-8类的字母
lcfirst：将后面第一个字母转换为小写，是\l的实现
ucfirst：将后面第一个字母转换为大写，是\u的实现

# 进制、编码转换类函数
chr：ASCII码(或unicode码点)转换为对应字符
ord：字符转换为ASCII码(或unicode码点)
hex：十六进制字符串转换为十进制数值
oct：八进制字符串转换为十进制数值

# 字符串处理类函数
chomp：去除行尾换行符
chop：去除行尾字符
substr：从字符串中取子串
split：根据指定分隔符将字符串划分为列表

# 字符索引类函数
index：获取字符所在索引位置
rindex：从后向前搜索，获取字符所在索引位置

# 其他函数
length：字符串字符数
sprintf：返回格式化的字符串(自行查阅perldoc -f sprintf)
crypt：加密字符串(实际上是计算MD5摘要信息)
```

下面举一些示例简单演示其中一部分函数的用法。

### 大小写转换和编码、进制转换

**字符串大小写转化类函数**：

```perl
say lc("HELLO");      # hello
say ucfirst("hello"); # Hello
```

**chr和ord**：

```perl
say chr(65);    # A
say ord('A');   # 65
say ord('AB');  # 65
```

**hex**：将十六进制字符串转换为十进制数值。注意，如果给定的不是字符串，而是数值本身，则数值转换为十进制后被当作十六进制字符串进行处理。

```perl
say hex '0x32'; # 50

# 没有前缀0x，32也当作十六进制字符串
say hex '32';   # 50

# 给定数值作为参数
# 0x32对应数值50，hex将50当作待处理字符串
# 等价于hex '0x50'
say hex 0x32;   # 80
```

**oct**：进制字符串转换为十进制数值的通用函数。可以处理八进制、十六进制、二进制：

- 当给定字符串以0x或0X开头，等价于hex函数  
- 当给定字符串以0b或0B开头，将字符串当作二进制字符串转换为十进制数值  
- 当给定字符串以0开头或没有特殊前缀开头，将字符串当作八进制字符串转换为十进制数值  
- 当给定的是数值参数而非字符串，则数值转换为十进制后被当作八进制字符串处理  

```perl
say oct '0x32';   # 50，等价于hex '0x32'
say oct '0b11';   # 3
say oct '050';    # 40
say oct '50';     # 40
say oct 0x32;     # 40，0x32转换为十进制50，等价于oct '50'
```

如果想要将十进制数值转换为指定的进制字符串，使用printf或sprintf：

```perl
printf "%#o\n", 420;  # 0644
printf "%#b\n", 3;    # 0b11
printf "%#x\n", 50;   # 0x32

printf "%#B\n", 3;    # 0B11
printf "%#X\n", 50;   # 0X32

printf "%o\n", 420;  # 644
printf "%b\n", 3;    # 11
printf "%x\n", 50;   # 32
```

### 字符串处理函数

**chomp**：移除尾部换行符，如果尾部没有换行符，则不做任何事。

实际上，chomp移除的是`$/`变量值对应的字符，该变量表示输入记录分隔符，默认为换行符，因此默认移除字符串尾部换行符。注意：  

- chomp不能对字符串字面量进行操作  
- chomp可以对左值进行操作  
- chomp可以操作列表，表示移除列表每项元素的尾部换行符  
- chomp也可以操作hash，表示移除每项value的为尾部换行符  
- chomp返回成功移除的总次数  
  - 对于字符串，如果有换行符，则返回1，否则返回0，因此可通过该返回值判断字符串是否有结尾换行符  
  - 对于列表，可以通过返回值来判断总共操作了多少行数据  

```perl
my $name = "junmajinlong\n";
chomp $name;  # name变为"junmajinlong"
chomp $name;  # name仍然是"junmajinlong"
chomp(my $tmp = "junmajinlong\n"); # 对左值变量进行操作

# chomp操作数组或列表
my @arr = ("junma\n","gaoxiao\n");
chomp @arr;    # @arr=("junma","gaoxiao")

# chomp操作hash
my %lines = (
  line1 => "abc\n",
  line2 => "def\n",
);
chomp %lines;
say "$lines{line1}, $lines{line2}";  # abc def
```

**chop**：是chomp的通用版，移除尾部单个字符，等价于`s/.$//s`，但效率更高。

```perl
my $name1 = "junmajinlong\n";
my $name2 = "junmajinlong";
chop $name1;   # $name1 = "junmajinlong"
chop $name2;   # $name2 = "junmajinlon"
```

**substr**：从给定字符串中提取并返回一段子串。用法：

```
substr STRING,OFFSET,LENGTH,REPLACEMENT
substr STRING,OFFSET,LENGTH
substr STRING,OFFSET
```

其中：

- offset从0开始计算  
- offset为负数时，表示从尾部位移(从-1开始计算)往前位移  
- length如果忽略，则从offset处开始取到最尾部  
- length为正数时，从offset处开始取length个字符  
- length为负数时，length则表示从后往前的位移位置，所以将从offset开始提取，直到尾部保留-length个的字符  
- replacement替换string中提取出来的字串。需注意：加了replacement，将返回提取的子串，同时会修改源字符串STRING  

```perl
$str="love your everything";

say substr $str,5;     # 输出：your everything
say substr $str,-10;   # 从后往前取：everything
say substr $str,5,4;   # 从前往后取4个字符：your
say substr $str,5,-3;  # 从位移5取到位移-3(从后往前算)：your everyth

say substr $str,5,4,"fairy's";  # 替换源字符串，但返回提取子串：your
say $str;              # 源字符串已被替换：love fairy's everything
```

**substr函数可以作为左值**，这样可以修改源变量，就像给了replacement参数一样：

```perl
$str="love your everything";
substr($str,5,4) = "fairy's";
say $str;       # 源字符串已被替换：love fairy's everything
```

**index和rindex**：用来找出给定字符串中某个子串或某个字符的索引位置。

```
(r)index STRING,SUBSTR,POSITION
(r)index STRDING,SUBSTR
```

- index搜索STRING中第一次出现SUBSTR的位置，rindex则从后向前搜索第一次出现的位置，也就是从前向后最后一次出现的位置 
- 如果省略position，则从起始位置(从0开始计算)开始搜索第一次出现的子串  
- 给定position，则从position处开始搜索，如果是rindex，则是找position左边的  
- 如果STRING中找不到SUBSTR，则返回-1  

```perl
$str="love you and your everything";

say index $str,"you";     # 输出：5
say index $str,"yours";   # 输出：-1
say index $str,"you",136; # 输出：-1
say index $str,"you",6;   # 从offset=6之后搜索，输出：13
say rindex $str,"you";    # 输出：13
say rindex $str,"you",10; # 找出offset=10左边的最后一个you，输出：5
```

**length**：返回字符串字符数。不能用于数组和hash。注意，如果包含多字节字符，要加上`use utf8`才能计算出字符数量。

```perl
say length("abc");     # 3
say length("我爱你");  # 9

# 加上use utf8
use utf8;
say length("我爱你啊a");  # 5
```

**crypt**：加密字符串，实际上是计算MD5摘要值：第一个参数是待计算摘要的字符串，第二个参数是salt字符串，salt至少两个字符且只取前两个字符(salt允许使用的字符为`[0-9a-zA-Z./]`共64个字符)。

如果待计算的字符串相同，且salt相同，那么计算出来的摘要信息一定相同。计算的结果中，前两位是salt的前两位字符，后面11位是计算结果。

```perl
say crypt("hello", "world");  # woglQSsVNh3SM
say crypt("hello", "wo");     # woglQSsVNh3SM
say crypt("hello", "wx");     # wxNrzGG7p9cyw
```

和openssl命令计算的md5值是一样的：

```bash
$ echo "hello" | openssl passwd -stdin -salt "wo"
woglQSsVNh3SM
```

**split**：根据指定分隔符，将字符串划分为列表形式，参考[Perl split](../ch3/7_op_list.md#perl_split)。

