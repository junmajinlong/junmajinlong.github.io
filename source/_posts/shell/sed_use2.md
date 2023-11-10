---
title: sed从文件a判断是否删除文件b中的某些行
p: shell/sed_use2.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列和sed系列文章大纲](/shell/index#sed)**  

------

# sed从文件a判断是否删除文件b中的某些行

test.xml文件很大，内容结构如下：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>

<url>
    <loc>http://www.u1cat.net/index.php?ctl=register</loc>
    <lastmod>2016-10-31</lastmod>
    <changefreq>always</changefreq>
    <priority>aaa</priority>
</url>

<url>
    <loc>http://www.u2bat.cc/index.php?ctl=register</loc>
    <lastmod>2015-11-18</lastmod>
    <changefreq>always</changefreq>
    <priority>bbb</priority>
</url>
<url>
    <loc>http://www.u3bat.cc/index.php?ctl=register</loc>
    <lastmod>2015-11-18</lastmod>
    <changefreq>always</changefreq>
    <priority>ccc</priority>
</url>
<url>
    <loc>http://www.u4bat.cc/index.php?ctl=register</loc>
    <lastmod>2015-11-18</lastmod>
    <changefreq>always</changefreq>
    <priority>ddd</priority>
</url>
<url>
    <loc>http://www.u5bat.cc/index.php?ctl=register</loc>
    <lastmod>2015-11-18</lastmod>
    <changefreq>always</changefreq>
    <priority>ddd</priority>
</url>
......
```

list.txt文件较小，内容如下：
```
bbb
xxx
yyy
ccc
```

需求是，如果`<url>...</url>`中间包含了list.txt文件中的某一行，则删除这个`<url>...</url>`。

在这里需要说明下sed的局限性：  
(1).sed处理输入流是一次性的，只要某行被sed读取了，就一定不会再读取。因此，读取到某满足匹配要求的行时，无法定位到它前面的某行、某几行。  
(2).sed自身没有显式的循环结构，例如while、for、until。但是通过某些功能的结合，可以隐式地实现循环。据我总结，只有标签跳转和"NDP"才能实现这种隐式意义上的循环。  
(3).sed和system命令交互的局限性非常大。只有e命令和s命令的e修饰符才能执行system中的命令。  
正是这3个局限性，导致sed实现上面的需求非常困难。

以下是一种效率非常高的方法：只读取一次test.xml和list.txt文件，并在每次读取到`<url>...</url>`的时候判断是否需要删除这一段。

创建sed脚本文件a.sed：
```
#!/usr/bin/sed -nf

\%<url>%!p
1{s/.*/cat list.txt/e;h}

\%<url>%{
N;N;N;N;N;G;
\%<priority>(.*)</priority>.*\1.*%d
}

s%</url>.*%</url>%p
```

执行sed：
```
sed -rn -f a.sed test.xml
```

由于上面示例文件中`<priority>bbb</priority>`和`<priority>ccc</priority>`的bbb、ccc存在于list.txt文件中，因此这两个`<url>...</url>`段落要删除。执行结果为：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>

<url>
    <loc>http://www.u1cat.net/index.php?ctl=register</loc>
    <lastmod>2016-10-31</lastmod>
    <changefreq>always</changefreq>
    <priority>aaa</priority>
</url>

<url>
    <loc>http://www.u4bat.cc/index.php?ctl=register</loc>
    <lastmod>2015-11-18</lastmod>
    <changefreq>always</changefreq>
    <priority>ddd</priority>
</url>
<url>
    <loc>http://www.u5bat.cc/index.php?ctl=register</loc>
    <lastmod>2015-11-18</lastmod>
    <changefreq>always</changefreq>
    <priority>ddd</priority>
</url>
```

思路大致为：  
- (1).一开始就通过sed的e命令将list.txt文件读取到pattern space空间，并保存到hold space。  
- (2).每读取到`<url>`的时候就继续读取后面5行，正好读到`</url>`。  
- (3).读完了`</url>`后，把hold space中的内容追加回pattern space，并从`<priority>XXXXX<priority>`开始判断后面是否还有XXXXX，如果有就直接删除pattern space，否则就将追加回pattern space的list.txt内容删除，最后输出。  
- (4).这样的执行方式，只需读取一次test.xml和list.txt文件，效率很高。  
