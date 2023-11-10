## 字符映射转换：tr和y///

除了可用使用`s///`来替换字符串，Perl还支持`y///`，它和sed命令的`y///`作用类似，用于字符映射转换。

Perl中，还支持tr，tr是和`y///`等价的别名。

tr的语法：
```
tr/SEARCH/REPLACEMENT/cdsr
y/SEARCH/REPLACEMENT/cdsr
```

tr或y用于将SEARCH中出现的字符(或指定修饰符c表示未出现的字符)逐字符转换为REPLACEMENT中的字符，根据指定的修饰符不同，转换操作要么是替换(默认是替换)，要么是删除(d修饰符)。

tr默认返回替换成功或删除成功的字符数量，如果指定修饰符r，则拷贝一份数据，对数据进行转换并返回，原数据保持不变。

其中：  
- c：取search的补集，将search中未找到的字符全都替换成replacement的最后一个字符
- d：删除search中出现的字符
- s：压缩重复字符，仅仅只需要压缩不需要替换时，可将replacement指定为空
- r：返回的不是替换成功的数量，而是替换成功后的内容，原数据不变，和s///的r修饰符是一样的

`y///`或`tr///`的斜线分隔符可以替换为其他符号，例如`tr### tr%%% tr===` 或括号方式`tr{}{} tr()() tr{}[]`以及它们的混合方式`tr{}// tr()##`。

### tr的基本用法

下面是tr的一些基本用法。

**1.tr的映射功能**

将小写字母e替换为大写字母E。  
```perl
my $str = "abcdef";
$str =~ y/e/E/;
say $str;
```

将小写字母全替换为大写字母。  
```perl
my $str="abcdef";
$str =~ y/a-z/A-Z/;
say $str;
```

如果对同一个字母指定不同的映射集，那么第一个映射关系将生效。
```perl
my $str="aaa ddd eee fff";
$str =~ tr/aaa/xyz/;  # a被映射为x
say $str;   # 输出xxx ddd eee fff
```

如果未指定Replacement部分，则拷贝Search部分作为Replacement，相当于不做任何替换，但却执行了tr操作，因此有返回值：

```perl
say tr/abcd//;   # 等价于say tr/abcd/abcd/
```

如果SEARCH部分比REPLACEMENT部分更长，则REPLACEMENT部分以最后一个字符补齐：

```perl
tr/abcd/AB/;  # 等价于tr/abcd/ABBB/
```

**2.s修饰符压缩连续字符**

使用s修饰符，可将SEARCH中出现的字符进行压缩，然后映射替换或删除。

例如：

```perl
my $str = "aabbccdd";
$str =~ tr/a//s;  # 连续的a替换为单个a，结果abbccdd
$str =~ tr/b/b/s; # 连续的b替换为单个b，结果abccdd
$str =~ tr/c/*/s; # 连续的c替换为单个*，结果ab*d
```

再例如：

```perl
my $str="abc    ddd eee    fff";
$str =~ tr/ //s;   # 压缩连续空格，结果：abc ddd eee fff
$str =~ tr/d/x/s;  # 压缩连续字符d，然后替换，结果：abc x eee fff
$str =~ tr/ef/y/s; # 压缩连续e和f，然后替换，结果：abc x y y
say $str;
```

可以压缩换行符：
```perl
my $str1="abc\n\nddd\neee    fff";
$str1 =~ tr/\n //s;  # abc\nddd\neee fff
```

**3.d修饰符删除SEARCH中未参与替换的字符**

例如，不指定replacement，search中的字符都被删除。用于直接删除某些字符。
```perl
my $str="abc ddd eee fff";
$str =~ y/de//d;    # abc   fff
```

指定replacement时，将先映射替换，再删除其余未参与替换的字符：

```perl
my $str="abc ddd eee fff";
$str =~ y/de/x/d;   # d替换为x，e被删除，结果abc xxx  fff
$str =~ y/abcf/mn/d; # a->m b->n，cf被删除，结果"mn xxx  "
```

**4.c修饰符取补集，将search中未指定的字符全部替换为replacement中的最后一个字符**  

c修饰符的作用，是将除search中指定字符之外的所有字符，都看作同一类字符。

```perl
my $str="aaa bbb ccc ddd";
$str =~ y/ab/xy/c;   # aaaybbbyyyyyyyy
```
注意上面，除a和b外，其余字符全都替换为y，replacement中的x字符被忽略。

c修饰符的作用，相当于将除了ab字符之外的其余字符，都看作另一种相同字符，然后进行下一步操作。比如上面示例，空格字符和cd字符都可以被看作同一类字符`A1 A2 A3`，然后它们都被替换为字符y。

如果replacement比search长，则仍然是取replacement的最后一个字符作为替换字符。所以下面的等价：
```perl
y/ab/xy/c;
y/ab/zxy/c
```
因此，replacement部分指定单个替换字符即可。

如果同时指定了s修饰符，则先补集替换，然后再压缩。
```perl
my $str="aaa bbb ccc ddd";
$str =~ y/ab/xy/sc;   # 先补集替换，得到aaaybbbyyyyyyyy
                      # 再压缩得到aaaybbby
```

因此，`sc`修饰符的效果是：除了search部分指定的字符外，其余字符全部被当成同一类字符，然后压缩。

如果同时指定`cd`修饰符，则删除所有未在search中的字符，也就是说replacement是多余的：

```perl
my $str="aaa bbb c d";
$str =~ y/ab/xy/dc; # 删除所有非ab字符，结果：aaabbb
$str =~ tr/0-9//cd; # 删除所有非数字字符
```

**5.使用r返回替换后的结果**

r修饰符使得处理数据前会先拷贝一份数据，然后对副本数据进行操作，所以原始数据会保持不变。
```perl
my $str="abcdef";
say $str =~ y/e/E/r;   # abcdEf
say $str;   # abcdef
```

### tr的一些细节

tr不指定操作对象时，默认操作`$_`。因此，下面是等价的：

```perl
$_ =~ tr/a/b/;
tr/a/b/;
```

tr默认返回被替换或被删除的字符数量，这可以作为获取字符串长度的一种技巧。但需要注意，和length获取字符长度一样，如果被操作的字符串中含有多字节字符，则应当`use utf8`，否则得到字节数而不是字符数。

```perl
say tr///c;  # 输出$_变量的字符数量

my $s="你好";
say $s =~ y///c;  # 6
{
  use utf8;
  say $s =~ y///c; # 2
}
```

使用r修饰符使得tr将拷贝一份数据，并对副本数据进行替换删除，原数据保持不变。因此，可将r修饰符操作后的数据赋值给变量，且有一些技巧可用：

```perl
($STR = $str) =~ tr/a/A/; # str不变，对STR进行转换
$STR = $str =~ tr/a/A/r;  # str不变，拷贝修改后赋值给STR

$STR = $str =~ tr/a/A/r   
            =~ s/:/ -p/r; # tr///r和s///r串联
```

tr总是在编译期间生成字符映射表，因此，无法在search部分和replacement部分使用变量。如果要使用变量，则应该使用eval来实现：

```perl
eval "tr/$oldlist/$newlist/";
```

tr有两种方式指定字符范围：`a-z0-9A-Z`，以及`[a-z0-9A-Z]`，前者不使用中括号，在编译期间可以精确知道哪些字符需要生成映射表，后者在编译后仍然是动态的字符类，因此无法对其生成映射表。使用不同的字符范围语法，可能会导致不同的结果。例如，在使用d修饰符删除时：

```perl
my $str="abc ddd eee fff";
# $str =~ y/d-f//d;     # e不会被保留，得到"abc   "
# $str =~ y/d-f/e/d;    # e被保留，得到"abc eee  "
# $str =~ y/[d-f]/e/d;  # e不会被保留，得到"abc   "
# $str =~ y/[def]/e/d;  # e不会被保留，得到"abc   "
# $str =~ y/[df]e/e/d;  # e不会被保留，得到"abc   "
```

除了`=~`，也可以使用`!~`操作符，这表示先进行转换，对转换后的结果布尔取反。也就是说，如果执行了转换操作，则返回布尔假，未执行转换操作，则返回布尔真。例如：

```perl
$str !~ tr/A//;   # 若$str中有A时返回假，没有A时返回真
```