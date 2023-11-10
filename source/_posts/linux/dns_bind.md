---
title: DNS和bind从基础到深入
p: linux/dns_bind.md
date: 2021-04-15 14:17:18
tags: Linux
categories: Linux
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# DNS和bind从基础到深入

DNS是Domain name system的简称，有些地方也称为Domain name server，这东西是一个很大的话题。如果不是要配置DNS服务，只需要理解DNS的解析流程和DNS有关的基本知识即可。如果要配置DNS服务，则可以看完全文。

推荐阅读书籍：《DNS \& bind》，第四版有中文版，第五版目前只有英文版。

## DNS必懂基础

DNS主要是用于将域名解析为IP地址的协议，有时候也用于将IP地址反向解析成域名，所以DNS可以实现双向解析。

DNS可以使用TCP和UDP的53端口，基本使用UDP协议的53端口。

### 域的分类

域是分层管理的，就像中国的行政级别。

最高层的域是根域(root)"."，就是一个点，它就像国家主席一样。全球只有13个根域服务器，基本上都在美国，中国一台根域服务器都没有。

根域的下一层就是第二层次的顶级域（TLD）了，那么它就是各省省长了。顶级域一般两种划分方法：按国家划分和按组织性质划分。

```
按国家划分：.cn(中国)、.uk(英国)、.us(美国)。基本都是两个字母的。
按组织性质划分：.org、.net、.com、.edu、.gov、.cc等。
反向域：arpa。这是反向解析的特殊顶级域。
```

顶级域下来就是普通的域，公司或个人在互联网上注册的域名一般都是这些普通的域，如baidu.com。

![](/img/linux/1699366820951.png)

### 主机名、域名、FQDN

以百度(www.baidu.com)和百度贴吧(tieba.baidu.com)来举例。

- 域名：不论是www.baidu.com还是tieba.baidu.com，它们的域名都是baidu.com，严格地说是"baidu.com."。这是百度所购买的com域下的一个子域名。

- 主机名：对于www.baidu.com来说，主机名是www，对于tieba.baidu.com来说，主机名是tieba。其实严格来说，www.baidu.com和tieba.baidu.com才是主机名，它们都是baidu.com域下的主机。一个域下可以定义很多主机，只需配置好它的主机名和对应主机的IP地址即可。

- FQDN：FQDN是Fully Qualified Domain Name的缩写，称为完全合格域名，是指包含了所有域的主机名，其中包括根域。FQDN可以说是主机名的一种完全表示形式，它从逻辑上准确地表示出主机在什么地方。例如www.baidu.com的FQDN是"www.baidu.com."，com后面还有个点，这是根域；tieba.baidu.com的FQDN是"tieba.baidu.com."。

### 域的分层授权

域是从上到下授权的，每一层都只负责自己的直辖下层，而不负责下下层。例如根域给顶级域授权，顶级域给普通域授权，但是根域不会给普通域授权。和现实中的行政管理不一样，域的授权和管理绝对不会向下越级，因为它根本不知道下下级的域名是否存在。

### DNS解析流程

![](/img/linux/1699366929052.png)

以访问www.baidu.com为例。

(1).客户端要访问www.baidu.com，首先会查找本机DNS缓存，再查找自己的hosts文件，还没有的话就找DNS服务器（这个DNS服务器就是计算机里设置指向的DNS）。

![](/img/linux/1699369125707.png)

(6).DNS服务器把得到的www.baidu.com的IP结果告诉客户端，并缓存一份结果在自己机器中(默认会缓存，因为该服务器允许为客户端递归，否则不会缓存非权威数据)。

(7).客户端得到回答的IP地址后缓存下来，并去访问www.baidu.com，然后www.baidu.com就把页面内容发送给客户端，也就是百度页面。

最后要说明的是：

1.本机查找完缓存后如果没有结果，会先查找hosts文件，如果没有找到再把查询发送给DNS服务器，但这仅仅是默认情况，这个默认顺序是可以改变的。在/etc/nsswitch.conf中有一行`hosts: files dns`就是定义先查找hosts文件还是先提交给DNS服务器的，如果修改该行为`hosts: dns files`则先提交给DNS服务器，这种情况下hosts文件几乎就不怎么用的上了。

2.由于缓存是多层次缓存的，所以真正的查询可能并没有那么多步骤，上图的步骤是完全没有所需缓存的查询情况。假如某主机曾经向DNS服务器提交了www.baidu.com的查询，那么在DNS服务器上除了缓存了www.baidu.com的记录，还缓存了".com"和"baidu.com"的记录，如果再有主机向该DNS服务器提交ftp.baidu.com的查询，那么将跳过"."和".com"的查询过程直接向baidu.com发出查询请求。

### /etc/resolv.conf文件

这个文件主要用于定义dns指向，即查询主机名时明确指定使用哪个dns服务器。该文件的详细说明见[Linux网络管理之：/etc/resolv.conf](/linux/linux_network#blog322)。

例如此文件中指定了`nameserver 8.8.8.8`，则每当要查询主机名时，都会向8.8.8.8这台dns服务器发起递归查询，这台dns服务器会帮忙查找到最终结果并返回给你。

当然，在后文的实验测试过程中，使用了另一种方式指定要使用的dns服务器：dig命令中使用`@dns_server`。

## DNS术语

### 递归查询和迭代查询

例如A主机要查询C域中的一个主机，A所指向的DNS服务器为B，递归和迭代查询的方式是这样的：

递归查询：`A --> B --> C --> B --> A`

迭代查询：`A --> B       A --> C --> A`

将递归查询和迭代查询的方式放到查询流程中，就如下图所示。(未标出Client指向的DNS服务器)

![](/img/linux/1699367252511.png)

也就是说，**递归的意思是找了谁谁就一定要给出答案。那么允许递归的意思就是帮忙去找位置，如A对B允许递归，那么B询问A时，A就去帮忙找答案，如果A不允许对B递归，那么A就会告诉B的下一层域的地址让B自己去找。**

可以想象，如果整个域系统都使用递归查询，那些公共的根域和顶级域会忙到死，因此更好的方案就是把这些压力分散到每个个人定制的DNS服务器。

所以DNS的解析流程才会如下图。并且在客户端到DNS服务器端的这一阶段是递归查询，从DNS服务器之后的是迭代查询。也就是说，顶级域和根域出于性能的考虑，是不允许给其他任何机器递归的。

![](/img/linux/1699367270888.png)

为什么客户端到DNS服务器阶段是递归查询？因为客户端本身不是DNS服务器，它自己是找不到互联网上的域名地址的，所以只能询问DNS服务器，最后一定由DNS服务器来返回答案，所以**DNS服务器需要对这个客户端允许递归。因此，dns解析器(nslookup、host、dig等)所发出的查询都是递归查询。**

### 权威服务器和(非)权威应答

![](/img/linux/1699369177552.png)

因此如果解析www.baidu.com时要获得权威应答，应该将DNS指向baidu.com这个域内负责解析的DNS服务器。

只有权威服务器直接给出的答案才是永远正确的，通过缓存得到的答案基本都是非权威应答。当然这不是一定的，因为权威服务器给的答案也是缓存中的结果，但是这是权威答案。DNS服务器缓存解析的数据库时间长度是由权威服务器决定的。

### DNS缓存

在Client和DNS服务器这些个人订制的DNS解析系统中都会使用缓存来加速解析以减少网络流量和查询压力，就算是解析不到的否定答案也会缓存。

但是要访问的主机IP可能会改变，所有使用缓存得到的答案不一定是对的，因此缓存给的答案是非权威的，只有对方主机的上一级给的答案才是权威答案。缓存给的非权威答案应该设定缓存时间，这个缓存时间的长短由权威者指定。

另外访问某个域下根本不存在的主机，这个域的DNS服务器也会给出答案，但是这是否定答案，否定答案也会缓存，并且有缓存时间。例如某个Client请求51cto.com域下的ftp主机，但是实际上51cto.com下面可能根本没有这个ftp主机，那么51cto.com就会给否定答案，为了防止Client不死心的访问ftp搞破坏，51cto.com这个域负责解析的DNS服务器有必要给Client指定否定答案的缓存时间。

### 主、从dns服务器

dns服务器也称为name server，每个域都必须有dns服务器负责该域相关数据的解析。但dns服务器要负责整个域的数据解析，压力相对来说是比较大的，且一旦出现问题，整个域都崩溃无法向外提供服务，这是非常严重的事。所以，无论是出于负载均衡还是域数据安全可用的考虑，两台dns服务器已经是最低要求了，多数时候应该配置多台dns服务器。

多台dns服务器之间有主次之分，主dns服务器称为master，从dns服务器称为slave。slave上的域数据都是从master上获取的，这样slave和master就都能向外提供名称解析服务。

### 资源记录(Resource Record,RR)

对于提供DNS服务的系统(DNS服务器)，域名相关的数据都需要存储在文件(区域数据文件)中。这些数据分为多种类别，每种类别存储在对应的资源记录(resource record,RR)中。也就是说，资源记录既用来区分域数据的类型，也用来存储对应的域数据。

DNS的internet类中有非常多的资源记录类型。常用的是SOA记录、NS记录、A记录(IPV6则为AAAA记录)、PTR记录、CNAME记录、MX记录等。

其中：(以下内容如果不了解，可以先跳过，在配置区域数据文件时回头来看)

**(1).SOA记录：start of authority，起始授权机构。该记录存储了一系列数据，若不明白SOA记录，请结合下面的NS记录，SOA更多的信息见"子域"部分的内容。格式如下：**

```
longshuai.com.  IN  SOA dnsserver.longshuai.com.    mail.longshuai.com. (
                            1     
                            3h    
                            1h    
                            1w    
                            1h )
```

第四列指定了"dnsserver.longshuai.com."为该域的master DNS服务器。

第五列是该域的管理员邮箱地址，但注意不能使用@格式的邮箱，而是要将@符号替换为点"."，正如上面的例子"mail.longshuai.com."，其实际表示的是"mail@longshuai.com"。

第六列使用括号将几个值包围起来。第一个值是区域数据文件的序列编号serial，每次修改此区域数据文件都需要修改该编号值以便让slave dns服务器同步该区域数据文件。第二个值是刷新refresh时间间隔，表示slave dns服务器找master dns服务器更新区域数据文件的时间间隔。第三个值是重试retry时间间隔，表示slave dns服务器找master dns服务器更新区域数据文件时，如果联系不上master，则等待多久再重试联系，该值一般比refresh时间短，否则该值表示的重试就失去了意义。第四个值是过期expire时间值，表示slave dns服务器上的区域数据文件多久过期。第五个值是negative ttl，表示客户端找dns服务器解析时，否定答案的缓存时间长度。这几个值可以分行写，也可以直接写在同一行中使用空格分开，所以，上面的SOA记录可以写成如下格式：

```
longshuai.com.  IN  SOA dnsserver.longshuai.com.  mail.longshuai.com. ( 1 3h 1h 1w 1h )
```

前三列是声明性的语句，表示"longshuai.com."这个域内的起始授权机构为第四列的值"dnsserver.longshuai.com."所表示的主机。第五列和第六列是SOA的附加属性数据。

每个区域数据文件中都有且仅能有一个SOA记录，且一般都定义为区域数据文件中的资源记录。

注意，资源记录的作用之一是存储域相关的对应数据，所以第4、5、6列表示的是该SOA记录所存储的相关值。

**(2).NS记录：name server，存储的是该域内的dns服务器相关信息。即NS记录标识了哪台服务器是DNS服务器。格式如下：**

```
longshuai.com.  IN  NS  dnsserver.longshuai.com.
```

前三列仍然是声明性语句，表示"longshuai.com."域内的DNS服务器(name server)为第四列值所表示的"dnsserver.longshuai.com."主机。

如果一个域内有多个dns服务器，则必然有主次之分，即master和slave之分。但在NS记录上并不能体现主次关系。例如：

```
longshuai.com.    IN  NS  dnsserver1.longshuai.com.
longshuai.com.    IN  NS  dnsserver2.longshuai.com.
```

表示主机"dnsserver1.longshuai.com."和主机"dnsserver2.longshuai.com."都是域"longshuai.com."内的dns服务器，但没有区分出主次dns服务器。

不少朋友搞不懂SOA记录，也很容易混淆SOA和NS记录。其实，仅就它们的主要作用而言，NS记录仅仅只是声明该域内哪台主机是dns服务器，用来提供名称解析服务，NS记录不会区分哪台dns服务器是master哪台dns服务器是slave。而SOA记录则用于指定哪个NS记录对应的主机是master dns服务器，也就是从多个dns服务器中挑选一台任命其为该域内的master dns服务器，其他的都是slave，都需要从master上获取域相关数据。由此，SOA的名称"起始授权机构"所表示的意思也就容易理解了。

**(3).A记录：address，存储的是域内主机名所对应的ip地址。格式如下：**

```
dnsserver.longshuai.com.    IN  A   172.16.10.15
```

客户端之所以能够解析到主机名对应的ip地址，就是因为dns服务器中的有A记录存储了主机名和ip的对应关系。
AAAA记录存储的是主机名和ipv6地址的对应关系。

**(4).PTR记录：pointer，和A记录相反，存储的是ip地址对应的主机名，该记录只存在于反向解析的区域数据文件中(并非一定)。格式如下：**

```
16.10.16.172.in-addr.arpa.  IN  PTR  www.longshuai.com.
```

表示解析172.16.10.16地址时得到主机名"www.longshuai.com."的结果。

**(5).CNAME记录：canonical name，表示规范名的意思，其所代表的记录常称为别名记录。之所以如此称呼，就是因为为规范名起了一个别名。什么是规范名？可以简单认为是fqdn。格式如下：**

```
www1.longshuai.com.     IN  CNAME  www.longshuai.com.
```

最后一列就是规范名，而第一列是规范名即最后一列的别名。当查询"www1.longshuai.com."，dns服务器会找到它的规范名"www.longshuai.com."，然后再查询规范名的A记录，也就获得了对应的IP地址并返回给客户端。

CNAME记录非常重要，很多时候使用CNAME可以解决很复杂的问题。而且目前常用的CDN技术有一个步骤就是在dns服务器上设置CNAME记录，将客户端对资源的请求引导到与它同网络环境(电信、网通)以及地理位置近的缓存服务器上。关于CDN的简介，见下文[CDN和DNS的关系](#cdn_dns)。

(6).MX记录：mail exchanger，邮件交换记录。负责转发或处理该域名内的邮件。和邮件服务器有关，且话题较大，所以不多做叙述，如有深入的必要，请查看《dns \& bind》中"Chapter 5. DNS and Electronic Mail"。

关于资源记录，最需要明确的概念就是它不仅仅用来区分和标识区域数据的类型，还用来存储对应的域数据。

## 安装DNS

Linux上搭建DNS服务的软件有bind9、NSD(Name server domain)和unbound，其中bind9是市场占有率最高的软件。本文也以此软件来介绍DNS服务。当前CentOS7.2上使用yum安装的bind为bind 9.9版本，bind实现DNS的最佳最详细说明文档是"DNS \& BIND"和"Bind97 Manual"，网上都有中文版本的资源。

```
[root@xuexi ~]# yum -y install bind
```

以下是bind生成的命令程序。

```
[root@xuexi ~]# rpm -ql bind | grep sbin
/usr/sbin/arpaname
/usr/sbin/ddns-confgen
/usr/sbin/dnssec-checkds
/usr/sbin/dnssec-coverage
/usr/sbin/dnssec-dsfromkey
/usr/sbin/dnssec-importkey
/usr/sbin/dnssec-keyfromlabel
/usr/sbin/dnssec-keygen
/usr/sbin/dnssec-revoke
/usr/sbin/dnssec-settime
/usr/sbin/dnssec-signzone
/usr/sbin/dnssec-verify
/usr/sbin/genrandom
/usr/sbin/isc-hmac-fixup
/usr/sbin/lwresd
/usr/sbin/named
/usr/sbin/named-checkconf
/usr/sbin/named-checkzone
/usr/sbin/named-compilezone
/usr/sbin/named-journalprint
/usr/sbin/nsec3hash
/usr/sbin/rndc
/usr/sbin/rndc-confgen
```

其中加粗标红的工具在后文可能都是要用的，它们是做什么的，可以直接man以下看看描述。其中named是提供DNS服务的主程序，它默认会读取配置文件/etc/named.conf。

安装bind后生成了以下几个配置文件。

```
[root@xuexi ~]# rpm -ql bind | grep etc
/etc/logrotate.d/named
/etc/named
/etc/named.conf
/etc/named.iscdlv.key
/etc/named.rfc1912.zones
/etc/named.root.key
/etc/rndc.conf
/etc/rndc.key
/etc/rwtab.d/named
/etc/sysconfig/named
/usr/share/doc/bind-9.9.4/sample/etc
/usr/share/doc/bind-9.9.4/sample/etc/named.conf
/usr/share/doc/bind-9.9.4/sample/etc/named.rfc1912.zones
```

其中named.conf是named工具默认的配置文件，它的配置指令项非常多。

## 配置和使用DNS

### named.conf配置简要说明

下文将以longshuai.com域以及这个域中的主机的定义来说明相关配置方法。

named.conf是named默认加载的配置文件，该配置文件中使用`#`或`/**/`或`//`作为注释符号，每个非注释语句都必须使用分号`;`结束。

该配置文件中只能有一个options，在这里面用于配置全局项。其中下例中的directory指令定义区域数据文件的存放目录。

```
options {
    directory "/var/named";
};
```

除了option，还必需有区域的配置。zone关键字后面接的是域和类，域是自定义的域名，IN是internet的简称，是bind 9中的默认类，所以可以省略。type定义该域的类型是`master|slave|stub|hint|forward`中的哪种，file定义该域的区域数据文件(区域数据文件的说明见下文)，因为这里是相对路径db.longshuai.com，它的相对路径是相对于/var/named的，也可以指定绝对路径/var/named/db.longshuai.com。

```
zone "longshuai.com" IN{
    type master;
    file "db.longshuai.com"
};
```

除此外，在每个named.conf中还应该配置几个必须的区域。

(1).根域名"."的区域配置。

```
zone "." IN {
    type hint;
    file named.ca;
}
```

type hint表示该区域"."类型为hint。回顾dns解析流程，在客户端让dns服务器迭代查询时，迭代查询的第一步就是让dns服务器去找根域名服务器。但是dns服务器如何知道根域名服务器在哪里？这就是hint类型的作用，它提示dns服务器根据其区域数据文件named.ca中的内容去获取根域名地址，并将这些数据缓存起来，下次需要根域名地址时直接查找缓存即可。

由于根域名地址也是会改变的，有了根区域的提示，就可以永远能获取到最新的根区域地址。其实也可以手动下载这些数据，地址为：<https://www.internic.net/domain/named.root>。

因此，只有根区域"."才会设置为hint类型。

(2)."localhost"域名(用于解析localhost为127.0.0.1)和127.0.0.1的方向查找区域。这两个没有定义在named.conf中，而是定义在/etc/named.rfc1912.zones中，然后在named.conf中使用include指令将其包含进来。

```
include "/etc/named.rfc1912.zones";
```

其中named.rfc1912.zones中部分内容：

```
zone "localhost" IN {
        type master;
        file "named.localhost";
        allow-update { none; };
};
 
zone "1.0.0.127.in-addr.arpa" IN {
        type master;
        file "named.loopback";
        allow-update { none; };
};
```

当然，反向查找区域可以定义为域而不是直接定义成主机。例如：

```
zone "0.0.127.in-addr.arpa" IN {
        type master;
        file "named.loopback";
        allow-update { none; };
};
```

但这样的话，就需要相对应地修改/var/named/named.loopback文件。

其实根域名"."和"localhost"以及"1.0.0.127.in-addr.arpa"完全没必要去改变，照搬就是了。

所以，/etc/named.conf的内容如下：

```
[root@xuexi ~]# cat /etc/named.conf
options {
    directory "/var/named";
};
 
zone "longshuai.com" {
    type master;
    file "db.longshuai.com";
};
 
zone "." IN {
    type hint;
    file "named.ca";
};
 
include "/etc/named.rfc1912.zones";
```

到此并没有结束，因为`/etc/named*`的属组都是named，且权限要为640。如下：

```
[root@xuexi ~]# ls -l /etc/named*
-rw-r--r-- 1 root root   205 Aug 12 21:58 /etc/named.conf
-rw-r----- 1 root named 1705 Mar 22  2016 /etc/named.conf.bak
-rw-r--r-- 1 root named 3923 Jul  5 18:15 /etc/named.iscdlv.key
-rw-r----- 1 root named  931 Jun 21  2007 /etc/named.rfc1912.zones
-rw-r--r-- 1 root named 1587 May 22 21:10 /etc/named.root.key
```

所以，对于新建的/etc/named.conf应该要改变属组和权限。

```
[root@xuexi ~]# chown root:named /etc/named.conf
[root@xuexi ~]# chmod 640 /etc/named.conf
```

然后使用/usr/sbin/named-checkconf命令来检查下/etc/named.conf文件的配置是否正确，如果不返回任何信息，则表示配置正确。

```
[root@xuexi ~]# named-checkconf
```

配置好配置文件后，接下来要书写域相关数据库——区域数据文件。

### 区域数据文件配置说明

区域数据文件即named.conf中zone关键字定义的域的数据文件，由zone中的file指令指定文件名，例如上面定义的"db.longshuai.com"，说明longshuai.com这个zone的区域数据文件为/var/named/db.longshuai.com。

区域数据文件中很多地方都可以使用缩写，如果不使用缩写的书写方式，那么每一条记录中的域名或主机名都要写全，即要写到根域"."。

假设longshuai.com域内有3台主机www、ftp和mydb，它们的ip分别为`172.16.10.{16,17,18}`，还有一台DNS服务器主机名为dnsserver，其ip为172.16.10.15。

![](/img/linux/1699367549668.png)

于是，longshuai.com这个区域的数据文件可以如下书写：区域数据文件中使用分号";"来注释。

```
$TTL 6h
longshuai.com.              IN  SOA dnsserver.longshuai.com.   xyz.longshuai.com. (
                       1     ; serial num
                       3h    ; refresh time 
                       1h    ; retry time 
                       1w    ; expire time
                       1h )  ; negative time

longshuai.com.              IN  NS  dnsserver.longshuai.com.

dnsserver.longshuai.com.    IN  A   172.16.10.15
www.longshuai.com.          IN  A   172.16.10.16
ftp.longshuai.com.          IN  A   172.16.10.17
mydb.longshuai.com.         IN  A   172.16.10.18

www1.longshuai.com.         IN  CNAME  www.longshuai.com.
```

其中第一行的`$TTL 6h`表示缓存周期，即查询该域中记录时肯定答案的缓存时间长度。例如本地查询www.baidu.com时，本地将缓存baidu.com域的相关查询结果，缓存时间长度由baidu.com这个域的区域数据文件中定义的`$TTL`的值决定。

在区域数据文件中，`$TTL`的定义表示其后的记录都以此TTL为准，直到遇到下一个`$TTL`。也就是说，两个`$TTL`之间的所有记录都以前面的`$TTL`为准。不过大多数时候，一个区域数据文件中只会在第一行定义一个`$TTL`值，表示该文件中所有记录都使用该缓存周期值。

第二行定义的是SOA记录，每个区域文件中的第一个资源记录定义行都要是SOA记录。在该行中除了指定了master dns服务器，还指定了一些附加属性，包括序列号为1，还有各种时长等信息。

第二个资源记录行定义的是NS记录，NS记录行表示该行定义的主机"dnsserver.longshuai.com."是这个域"longshuai.com."内的dns服务器。

接下来的几个资源记录行定义的都是A记录，分别定义了`{dnsserver,www,ftp,mydb}.longshuai.com.`这几个主机对应的ip地址为`172.16.10.{15,16,17,18}`。A记录中存储的是主机名和IP地址之间的映射关系，在为外界主机提供主机名查询服务时，就是从A记录中获取对应的ip地址。

最后一个资源记录行定义的是CNAME记录，它存储的是canonical name所对应的别名。例如此处定义的是"www.longshuai.com."的别名为"www1.longshuai.com."。

然后修改/var/named/db.longshuai.com文件的属组和权限。

```
[root@xuexi ~]# chmod 640 /var/named/db.longshuai.com 
[root@xuexi ~]# chown root:named /var/named/db.longshuai.com
```

然后使用named-checkzone命令检查区域数据文件是否书写正确。例如，要检查"longshuai.com"区域，其区域数据文件为/var/named/named.conf。

```
[root@xuexi ~]# named-checkzone longshuai.com /var/named/db.longshuai.com
zone longshuai.com/IN: loaded serial 1
OK
```

实际上，应该对所有的区域都进行检查，使用named-checkzone一个一个进行手动检查，显得非常繁琐。在CentOS 6上，支持使用"service named configtest"来检查，CentOS 7上没有对应的功能。但无论是CentOS 6还是CentOS 7上，在启动named服务的时候，都会自动检查配置文件的正确性。

回到区域数据文件的配置说明上。在上面的配置过程中，完全没有使用缩写，但其实很多地方都能使用缩写的书写方式。以下是区域数据文件中的一些书写规则，包括缩写规则：

**(1).`$`符号：定义宏。最常见的是`$TTL`、`$ORIGIN`。origin的的意思是起点，可以自行定义`$ORIGIN`的值。**

例如：`$ORIGIN`表示定义的`$ORIGIN`的起点值为根域符号，`$ORIGIN longshuai.com.`表示`$ORIGIN`的值为"longshuai.com."，未定义`$ORIGIN`时，值为zone "domain"的domain部分。

**(2).fqdn自动补齐：在区域数据文件中，没有使用点号"."结尾的，在实际使用的时候都会自动补上域名，使其变为fqdn。**

例如区域"longshuai.com."，以下是完全格式的资源记录：

```
dnsserver.longshuai.com.    IN  A   172.16.10.15
```

可以缩写为：

```
dnsserver    IN    A   172.16.10.15
```

因为dnsserver后没有点，所以会补齐整个域名"longshuai.com."。

实际上，自动补齐的部分是`$ORIGIN`的值，只不过默认没定义`$ORIGIN`时，`$ORIGIN`的值为zone定义的域名，所以默认是自动补齐域名。

**(3)."@"符号：可以使用@符号来缩写$ORIGIN的值。**

由于自定义的区域数据文件中，一般不会主动定义`$ORIGIN`的值，而第一个资源记录一般都是SOA记录，所以此时SOA记录中的第一列就可以使用`@`符号，其它地方只要值为`$ORIGIN`，都可以使用@符号缩写。例如：

```
@     IN  SOA  dnsserver  xyz. ( 1 3h 1h 1w 1h )
@     IN  NS   dnsserver
```

**(4).重复最近一个名称：区域数据文件中的第一列可以使用空格或制表符使该列继承上一行的第一列的值。**

例如第一行定义的是SOA记录，第一列是"longshuai.com."，那么第二行定义的NS记录中，其第一列就可以留空来继承第一行第一列的"longshuai.com."。不止第一行和第二行，第三行也可以继承第二行的第一列，第n+1行也可以继承第n行的第一列，只要它们的值一样即可。

所以，/var/named/db.longshuai.com这个文件完全缩写后的结果如下：

```
[root@xuexi named]# vim /var/named/db.longshuai.com 
$TTL 6h
@         IN  SOA    dnsserver   xyz ( 1 3h 1h 1w 1h )
@         IN  NS     dnsserver
 
dnsserver IN  A      172.16.10.15
www       IN  A      172.16.10.16
ftp       IN  A      172.16.10.17
mydb      IN  A      172.16.10.18
 
www1      IN  CNAME  www
```

是否缩写正确，可以使用named-checkzone来检查下，还可以使用named-compilezone命令对区域数据文件进行编译，并输出编译后的结果。

```
[root@xuexi named]# named-compilezone  -o  -  longshuai.com  /var/named/db.longshuai.com 
zone longshuai.com/IN: loaded serial 1
longshuai.com.            21600 IN SOA    dnsserver.longshuai.com. xyz.longshuai.com. 1 10800 3600 604800 3600
longshuai.com.            21600 IN NS     dnsserver.longshuai.com.
dnsserver.longshuai.com.  21600 IN A      172.16.10.15
ftp.longshuai.com.        21600 IN A      172.16.10.17
mydb.longshuai.com.       21600 IN A      172.16.10.18
www.longshuai.com.        21600 IN A      172.16.10.16
www1.longshuai.com.       21600 IN CNAME  www.longshuai.com.
OK
```

"-o"选项表示将编译后的结果输出到指定文件中，"-"表示输出到标准输出。

由上面编译的结果可以看出，缩写后的结果并没有任何错误。

在/etc/named.conf中，还定义了3个区域，分别是"."、"localhost"和"1.0.0.127.in-addr.arpa"，它们的区域数据文件分别是`/var/named/{named.ca,named.localhost,loopback}`，这几个文件的内容可以自行解读。

```
[root@xuexi ~]# cat /var/named/named.ca 
; <<>> DiG 9.9.4-RedHat-9.9.4-38.el7_3.2 <<>> +bufsize=1200 +norec @a.root-servers.net
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17380
;; flags: qr aa; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1472
;; QUESTION SECTION:
;.                              IN      NS

;; ANSWER SECTION:
.                  518400  IN      NS      a.root-servers.net.
.                  518400  IN      NS      b.root-servers.net.
.                  518400  IN      NS      c.root-servers.net.
.                  518400  IN      NS      d.root-servers.net.
.                  518400  IN      NS      e.root-servers.net.
.                  518400  IN      NS      f.root-servers.net.
.                  518400  IN      NS      g.root-servers.net.
.                  518400  IN      NS      h.root-servers.net.
.                  518400  IN      NS      i.root-servers.net.
.                  518400  IN      NS      j.root-servers.net.
.                  518400  IN      NS      k.root-servers.net.
.                  518400  IN      NS      l.root-servers.net.
.                  518400  IN      NS      m.root-servers.net.

;; ADDITIONAL SECTION:
a.root-servers.net.  3600000 IN  A       198.41.0.4
a.root-servers.net.  3600000 IN  AAAA    2001:503:ba3e::2:30
b.root-servers.net.  3600000 IN  A       192.228.79.201
b.root-servers.net.  3600000 IN  AAAA    2001:500:84::b
c.root-servers.net.  3600000 IN  A       192.33.4.12
c.root-servers.net.  3600000 IN  AAAA    2001:500:2::c
d.root-servers.net.  3600000 IN  A       199.7.91.13
d.root-servers.net.  3600000 IN  AAAA    2001:500:2d::d
e.root-servers.net.  3600000 IN  A       192.203.230.10
e.root-servers.net.  3600000 IN  AAAA    2001:500:a8::e
f.root-servers.net.  3600000 IN  A       192.5.5.241
f.root-servers.net.  3600000 IN  AAAA    2001:500:2f::f
g.root-servers.net.  3600000 IN  A       192.112.36.4
g.root-servers.net.  3600000 IN  AAAA    2001:500:12::d0d
h.root-servers.net.  3600000 IN  A       198.97.190.53
h.root-servers.net.  3600000 IN  AAAA    2001:500:1::53
i.root-servers.net.  3600000 IN  A       192.36.148.17
i.root-servers.net.  3600000 IN  AAAA    2001:7fe::53
j.root-servers.net.  3600000 IN  A       192.58.128.30
j.root-servers.net.  3600000 IN  AAAA    2001:503:c27::2:30
k.root-servers.net.  3600000 IN  A       193.0.14.129
k.root-servers.net.  3600000 IN  AAAA    2001:7fd::1
l.root-servers.net.  3600000 IN  A       199.7.83.42
l.root-servers.net.  3600000 IN  AAAA    2001:500:9f::42
m.root-servers.net.  3600000 IN  A       202.12.27.33
m.root-servers.net.  3600000 IN  AAAA    2001:dc3::35

;; Query time: 18 msec
;; SERVER: 198.41.0.4#53(198.41.0.4)
;; WHEN: Po kvě 22 10:14:44 CEST 2017
;; MSG SIZE  rcvd: 811
```

该文件中记录了获取根区域的方法：`dig +bufsize=1200 +norec @a.root-servers.net`。

```
[root@xuexi ~]# cat /var/named/named.{localhost,loopback}   
$TTL 1D
@   IN SOA  @ rname.invalid. (
                              0       ; serial
                              1D      ; refresh
                              1H      ; retry
                              1W      ; expire
                              3H )    ; minimum
    NS      @
    A       127.0.0.1
    AAAA    ::1

$TTL 1D
@  IN SOA  @ rname.invalid. (
                             0       ; serial
                             1D      ; refresh
                             1H      ; retry
                             1W      ; expire
                             3H )    ; minimum
   NS      @
   A       127.0.0.1
   AAAA    ::1
   PTR     localhost.
```

 至此，配置文件和区域数据文件都已经配置结束了，可以启动named服务。

```
[root@xuexi named]# systemctl restart named.service 
[root@xuexi named]# netstat -tnlup | grep named     
tcp   0  0 172.16.10.15:53  0.0.0.0:*   LISTEN  66248/named 
tcp   0  0 127.0.0.1:53     0.0.0.0:*   LISTEN  66248/named 
tcp   0  0 127.0.0.1:953    0.0.0.0:*   LISTEN  66248/named 
tcp6  0  0 ::1:953          :::*        LISTEN  66248/named 
udp   0  0 172.16.10.15:53  0.0.0.0:*           66248/named 
udp   0  0 127.0.0.1:53     0.0.0.0:*           66248/named 
```

从结果中看到，named默认监听在所有接口的tcp和udp的53端口上，但还监听了环回地址的tcp的953端口，这是named为rndc提供的控制端口，rndc是named的远程控制工具，在后文会专门介绍其用法。

### 测试DNS的解析

任找一台机器(也可以是dns服务器本身)，将其dns指向dns服务器的监听地址。例如，直接在上文中dns服务器172.16.10.15上设置。

```
[root@xuexi ~]# vim /etc/resolv.conf
search localdomain longshuai.com
nameserver 172.16.10.15
```

然后使用nslookup命令或host命令或bind-utils包中提供的dig命令来测试能否解析longshuai.com这个域相关数据。建议使用dig命令，因为host太简陋，nslookup在某些情况下有缺陷，dig命令则相对更完美些。dig命令的用法请参见man文档。

例如，解析longshuai.com域中的主机www的A记录。

```
[root@xuexi ~]# dig -t a www.longshuai.com
; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> -t a www.longshuai.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8670
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.longshuai.com.             IN      A

;; ANSWER SECTION:
www.longshuai.com.      21600   IN      A       172.16.10.16

;; AUTHORITY SECTION:
longshuai.com.          21600   IN      NS      dnsserver.longshuai.com.

;; ADDITIONAL SECTION:
dnsserver.longshuai.com. 21600  IN      A       172.16.10.15

;; Query time: 0 msec
;; SERVER: 172.16.10.15#53(172.16.10.15)
;; WHEN: Sat Aug 12 23:38:17 CST 2017
;; MSG SIZE  rcvd: 102
```

在结果中：

(1)."QUESTION SECTION"表示所发起的查询，表示要查询"www.longshuai.com."的A记录。

(2)."ANSWER SECTION"表示对查询的回复。回复的结果是"www.longshuai.com."的A记录值为"172.16.10.16"，这正是dig所期望的结果。

(3)."AUTHORITY SECTION"表示该查询是权威服务器给的答案，并给出了权威服务器的ns记录。在此例中，"www.longshuai.com"主机所在的域"longshuai.com"的权威服务器为"dnsserver.longshuai.com."。如果没有该段，表示非权威应答。

(4)."ADDITIONAL SECTION"段是额外的回复，回复的内容是权威服务器的A记录。

还可以测试ns记录、soa记录、cname记录等。

不过需要注意的是soa记录和ns记录查询的对象是域名而不是主机名，而CNAME记录的对象则必须是主机名。

```
[root@xuexi ~]# dig -t ns longshuai.com
[root@xuexi ~]# dig -t soa longshuai.com
[root@xuexi ~]# dig -t cname www1.longshuai.com
```

## 配置反向查找区域

反向查找是根据ip地址查找其对应的主机名。在/etc/named.conf中，需要定义`zone *.in-addr.arpa`，其中`*`是点分十进制ip的反写，可以是反写ip后的任意一段长度，例如127.0.0.1反写后是1.0.0.127，所以zone所定义的可以是"1.0.0.127"、"0.0.127"、"0.127"，甚至是"127"，长度位数不同，在区域数据文件中需要补全的数值就不同。

另外，反向查找区域的各种缓存时间可以都设置长一点，因为用的不多。

就以127.0.0.1解析为localhost为例。

在/etc/named.rfc1912.zones中已有这样一段：

```
zone "1.0.0.127.in-addr.arpa" IN {
       type master;
       file "named.loopback";
       allow-update { none; };
};
```

其区域数据文件/var/named/named.loopback内容如下：

```
[root@xuexi ~]# cat /var/named/named.loopback 
$TTL 1D
@  IN SOA  @ rname.invalid. (
                             0       ; serial
                             1D      ; refresh
                             1H      ; retry
                             1W      ; expire
                             3H )    ; minimum
   NS      @
   A       127.0.0.1
   AAAA    ::1
   PTR     localhost.
```

将/etc/named.rfc1912.zones中将1.0.0.127.in-addr.arpa区域的配置注释掉，改为如下内容：

```
zone "0.127.in-addr.arpa" IN {
        type master;
        file "named.loopback.test";
        allow-update { none; };
};
```

然后书写其区域数据文件/var/named/named.loopback.test。

```
$TTL 1D
@    IN  SOA  1.0    rname.invalid. ( 0 1D 1H 1W 3H )  
         NS   1.0.0.127.in-addr.arpa.
1.0      A    127.0.0.1
1.0      PTR  local.
```

在上述配置中，特意将1.0的ptr记录写成了"local."，这样查询1.0.0.127的主机名时，将得到local而不是localhost。

修改属组、权限后重启named，并使用`dig -x`测试，dig的"-x"选项专门用于反向查找。

```
[root@xuexi ~]# chown root:named /var/named/named.loopback.test
[root@xuexi ~]# chmod 640 /var/named/named.loopback.test
[root@xuexi ~]# systemctl restart named
[root@xuexi ~]# dig -x 127.0.0.1

; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> -x 127.0.0.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51058
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;1.0.0.127.in-addr.arpa.            IN   PTR

;; ANSWER SECTION:
1.0.0.127.in-addr.arpa. 86400   IN  PTR  local.

;; AUTHORITY SECTION:
0.127.in-addr.arpa.     86400   IN  NS   1.0.0.127.in-addr.arpa.

;; ADDITIONAL SECTION:
1.0.0.127.in-addr.arpa. 86400   IN  A    127.0.0.1

;; Query time: 0 msec
;; SERVER: 172.16.10.15#53(172.16.10.15)
;; WHEN: Sun Aug 13 04:58:47 CST 2017
;; MSG SIZE  rcvd: 100
```

现在，可以配置longshuai.com中主机的反向解析区域，由于在实验过程中，该域中的主机地址都是172.16.10.0/24网段内的主机，所以只需要配置一个10.16.172.in-addr.arpa反向查找区域即可，如果域内主机跨了多个网段，例如172.16.10.0/24和192.168.100.0/24，则需要配置两个反向查找区域。

以下是配置结果：

```
[root@xuexi ~]# cat /etc/named.conf
options {
    directory "/var/named";
};

zone "longshuai.com" {
    type master;
    file "db.longshuai.com";
};

zone "10.16.172.in-addr.arpa" in {
    type master;
    file "db.10.16.172";
};

zone "." IN {
    type hint;
    file "named.ca";
};

include "/etc/named.rfc1912.zones";
[root@xuexi ~]# vim /var/named/db.10.16.172
$TTL 1D
@    IN   SOA   15                         xyz.longshuai.com. ( 0 1D 1H 1W 3H )
     IN   NS    dnsserver.longshuai.com.
15   IN   PTR   dnsserver.longshuai.com.
16   IN   PTR   www.longshuai.com.
17   IN   PTR   ftp.longshuai.com.
18   IN   PTR   mydb.longshuai.com.
```

重启named，然后使用`dig -x`测试。

```
[root@xuexi ~]# chown root:named /var/named/db.10.16.172
[root@xuexi ~]# chmod 640 /var/named/db.10.16.172   
[root@xuexi ~]# systemctl restart named
[root@xuexi ~]# dig -x 172.16.10.16

; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> -x 172.16.10.16
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60502
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;16.10.16.172.in-addr.arpa.     IN   PTR

;; ANSWER SECTION:
16.10.16.172.in-addr.arpa. 86400 IN  PTR www.longshuai.com.

;; AUTHORITY SECTION:
10.16.172.in-addr.arpa. 86400   IN   NS  dnsserver.longshuai.com.

;; ADDITIONAL SECTION:
dnsserver.longshuai.com. 21600  IN   A   172.16.10.15

;; Query time: 0 msec
;; SERVER: 172.16.10.15#53(172.16.10.15)
;; WHEN: Sun Aug 13 05:18:35 CST 2017
;; MSG SIZE  rcvd: 125
```

## 配置"仅缓存"dns服务器

仅用作提供缓存的dns服务器，当有客户端请求该dns服务器帮忙解析某个地址时，它不会直接为外界主机提供dns解析，而是自己去找其他dns服务器解析，并将结果缓存在本地，并将缓存结果提供给客户端。

也就是说，仅缓存dns服务器其实扮演的角色和客户端一样，只不过它还未其他客户端提供解析查询而已。

要配置仅缓存dns服务器，只要配置3个任何时候都必要的域：根域"."、"localhost"域和"1.0.0.127.in-addr.arpa"。也就是说，任何一台完整的dns服务器，至少都是"缓存"服务器。

所以，仅缓存dns服务器配置如下：

```
[root@xuexi ~]# vim /etc/named.conf
options {
    directory "/var/named";
};

#zone "longshuai.com" {
#    type master;
#    file "db.longshuai.com";
#};

zone "." IN {
    type hint;
    file "named.ca";
};

include "/etc/named.rfc1912.zones";
```

其他的采用默认即可。

dig测试时，如何区分是否是由缓存给答案还是解析后给答案，有些时候不是很好判断，但如果仅仅是测试仅缓存服务器的缓存效果，方法还是很简单的。例如，将上述dns服务器指向8.8.8.8，另找一台机器，执行dig命令，并明确指定使用该缓存dns服务器来帮忙解析。例如，dig一下www1.baidu.com的别名记录。

```
[root@xuexi ~]# dig -t cname www1.baidu.com @172.16.10.15

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.62.rc1.el6 <<>> -t cname www1.baidu.com @172.16.10.15
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43817
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 5, ADDITIONAL: 5

;; QUESTION SECTION:
;www1.baidu.com.                        IN      CNAME

;; ANSWER SECTION:
www1.baidu.com.     7200    IN   CNAME   www.baidu.com.

;; AUTHORITY SECTION:
baidu.com.         86400   IN   NS  ns2.baidu.com.
baidu.com.         86400   IN   NS  dns.baidu.com.
baidu.com.         86400   IN   NS  ns3.baidu.com.
baidu.com.         86400   IN   NS  ns4.baidu.com.
baidu.com.         86400   IN   NS  ns7.baidu.com.

;; ADDITIONAL SECTION:
ns4.baidu.com.     172800  IN   A   220.181.38.10
ns3.baidu.com.     172800  IN   A   220.181.37.10
ns2.baidu.com.     172800  IN   A   61.135.165.235
dns.baidu.com.     172800  IN   A   202.108.22.220
ns7.baidu.com.     172800  IN   A   119.75.219.82

;; Query time: 527 msec
;; SERVER: 172.16.10.15#53(172.16.10.15)
;; WHEN: Thu Aug 10 04:53:07 2017
;; MSG SIZE  rcvd: 220
```

查询消耗了527毫秒。再次执行上面的dig，发现查询时间一定是0毫秒，因为是缓存给的答案。

由于缓存在于多个域空间层次上，我们个人能控制缓存的机器只有执行dig命令的客户端和仅缓存dns服务器172.16.10.15，所以可以在两台机器上依次执行"rndc flush"命令来清空缓存测试dig命令，看什么时候query time变回几百几千毫秒，这时就说明dig的答案不是缓存提供的。

## 配置dns转发服务器

配置成了转发服务器，named.conf里所有的zone都将失效(除非配置转发区和空转发区)，也就是不会再做任何解析（包括对根的查询），收到的解析请求全都提交给转发选项里指定的机器，转发选项所指定的机器称为转发器。转发器还可以指定给上一层。

例如，新添加一台dns服务器172.16.10.9，设置其为转发者，其转发器为172.16.10.15，如下图。

![](/img/linux/1699368065208.png)

配置转发器的方式极其简单，只需在172.16.10.9这台dns服务器上/etc/named.conf中的options中使用一个forwarders指令。其实还需加上forward指令，只不过现在采取其默认值，所以省略它。/etc/named.conf内容如下：

```
options {
    directory "/var/named";
    forwarders { 172.16.10.15; }; 
};
include /etc/named.rfc1912.zones;
```

这表示将172.16.10.9收到的所有查询请求都交给172.16.10.15这台转发器，由这台转发器帮忙查询相关请求，然后回复给转发者，由转发者回复给客户端。如果要指定多个转发器，这使用分号分隔各转发器地址，如`forwarders { 172.16.10.15;172.16.10.11; }`。

例如，前文在172.16.10.15上已经配置了longshuai.com域的域数据，例如www.longshuai.com主机的A记录。此时找台客户端，让其dns指向转发者172.16.10.9，然后发起www.longshuai.com的A记录查询请求，如果能返回得到172.16.10.15主机上所配置的域数据，则表明转发成功。

```
[root@xuexi ~]# dig -t a www.longshuai.com @172.16.10.9

; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7 <<>> -t a www.longshuai.com @172.16.10.9
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64375
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.longshuai.com.             IN  A

;; ANSWER SECTION:
www.longshuai.com.      21600   IN  A   172.16.10.16

;; AUTHORITY SECTION:
longshuai.com.          21600   IN  NS  dnsserver.longshuai.com.

;; ADDITIONAL SECTION:
dnsserver.longshuai.com. 21600  IN  A   172.16.10.15

;; Query time: 4 msec
;; SERVER: 172.16.10.9#53(172.16.10.9)
;; WHEN: Sun Aug 13 22:56:32 CST 2017
;; MSG SIZE  rcvd: 102
```

考虑一种特殊情况，如果转发器无法查询到结果，也就是说转发者获取不到来自转发器的答案，则转发者自己也会去查询。也就是说，如果转发器无法提供答案时，转发者中间会间隔一段等待时间。

当然，还存在一种限制性极大的"仅转发"dns服务器，它只转发，即使获取不到转发器的回复，也不会自己去查询。要配置成"仅转发"dns服务器，只需加上"forward only"指令即可。而前面所说的情况则是"forward first"，先转发，转发失败时自行查询，只不过这是默认值，可以省略。

```
options {
    directory "/var/named";
    forwarders { 172.16.10.15; };
    forward only;
};
include /etc/named.rfc1912.zones;
```

其实，配置成仅转发dns服务器时，除了forward和forwarder指令，其他的一切指令都可以省略，即使是direcotry指令。因为仅转发dns服务器根本用不上任何区域，所有的查询请求包括localhost这样的请求都会转发。所以上述配置可简化为：

```
options {
    forwarders { 172.16.10.15; };
    forward only;
};
```

也许你已经感受到了，几乎不会去配置仅转发dns服务器。实际上，操作系统中设置dns指向的地方，就是在设置仅转发功能，例如/etc/resolv.conf文件中所设置的nameserver，这就是仅转发功能。

需要注意的一点是，**转发者转发给转发器的查询是递归查询，转发器必须要亲自回复转发者**，也就是说转发者其实就相当于一个客户端。

其实，上述所述的情况是传统的转发功能：配置在options中的转发指令，只能作用于全局，也就是说无论对哪个域的查询，都全部转发出去。在bind 9.1之后，**可以配置新的转发功能——转发区，只有指定的区域的查询请求才会转发出去**。

转发区的配置方式也很简单，不用在options中指定forward，而是在指定的区域内设置其type为forward。例如在172.16.10.9上：

```
options {
    directory "/var/named";
};

zone "longshuai.com" IN {
    type forward;
    forwarders { 172.16.10.15; };
};

zone "." IN {
    type hint;
    file "named.ca";
};

include /etc/named.rfc1912.zones;
```

这样，只有查询"longshuai.com"结尾的记录都将转发到172.16.10.15上。但要注意，是longshuai.com结尾，而不是限制其域为"longshuai.com"，所以如果发出png.img.longshuai.com的查询请求，也会被转发，但实际上这是longshuai.com的子域img.longshuai.com中的主机。

![](/img/linux/1699369235438.png)

```
options {
    directory "/var/named";
    forwarders { 172.16.10.15; };
};

zone "longshuai.com" IN {
    type master;
    forwarders {};
};

zone "." IN {
    type hint;
    file "named.ca";
};

include /etc/named.rfc1912.zones;
```

注意，在options中设置了转发目标，在longshuai.com中设置了type master，也接受slave、stub类型的区，最重要的是设置了目标为空的forwarders指令。这样longshai.com结尾的查询请求都不会转发走，包括子域内主机png.img.longshuai.com的查询。

## ACL

![](/img/linux/1699369265957.png)

## 递归查询详述

先回顾下递归查询和迭代查询的概念。dns服务器接收到递归查询请求时，它需要帮忙去找答案，并亲自回复请求者，如果收到的是迭代查询请求，则将自己知道的消息(一般是自己负责的域信息，所以是权威消息)告诉请求者，让请求者亲自去查询。

由于**dns解析器发起的查询都是递归查询，所以一般客户端配置DNS指向谁就表示找谁帮忙做递归查询。如果dns服务器接受它的递归查询请求，则它会去帮助查询，如果dns服务器不接受它的递归查询请求，则会将递归查询当成迭代查询看待，让请求者自己去查询。**

**另外，允许递归查询的服务器，由于要帮忙查询，所以在递归查询服务器上总是缓存了一些非权威数据，如果是非递归查询服务器，则不用缓存任何数据，只需返回其负责的域的权威数据即可，这对减轻压力的作用是非常大的。**正如根域"."服务器，它接受来自世界各地的查询请求，如果允许递归，不仅其压力极大，且缓存也会极大。所以正常情况下，根域名服务器是不允许递归查询的。

dns服务器对不是自己负责的域，不应该帮别人递归。例如，远程client访问longshuai.com域，因为这就是dnsserver.longshuai.com负责的域，而且又必须给答案，所以应该给递归。但是client通过dnsserver.longshuai.com来查询sohu.com，那就不应该给递归，否则dns服务器会产生更多流量和压力。

默认情况下，/etc/named.conf中隐含了选项`recursion yes"和"allow-recursion {any;}`，也就是说允许递归查询，且是允许所有人发起的递归查询。

如果想明确指定不给哪些主机递归，则指定allow-recursion来覆盖默认的`allow-recursion {any;}`。例如，不给192.168.100.0/24递归。

```
[root@xuexi ~]# vi /etc/named.conf
options {
    recursion yes;
    allow-recursion {192.168.100.0/24;};
};
```

如果不想给所有人递归，则将recursion设置为no，关闭递归功能即可。

另外，**不要将非递归查询dns服务器设置为转发器，因为转发者转发给转发器的查询是递归查询**。

 

除了可以控制递归查询外，还可以更直接地控制是否允许查询。使用`allow-query {}`指令可以指定允许查询的主机列表，列表外的主机都不允许查询，不仅不允许为客户端做递归查询，也不允许为客户端做迭代查询。

## 明确指定不查询的dns服务器

自设的dns服务器帮忙查询答案时，有些远程dns服务器会返回一些错误、过时或者蓄意欺骗的信息。对于这些有问题的远程dns服务器，自设dns服务器不应该去查询它们。

为了让dns服务器明确不查询指定的远程dns服务器，可以使用：

```
server 10.0.0.2 {
    bogus yes;
};
```

对于大量远程dns服务器黑名单，可以将这些主机放进blackhole。

```
options {
    blackhole {
        10/8;
        172.16/12;
        192.168/16;
    };
};
```

## 配置主、从dns服务器

在配置/etc/named.conf的zone区域时，会指定type是什么类型的，常见的有master、slave、stub、forward、hint等。forward和hint在前文已经介绍过了，stub在后文介绍子域的时候介绍，此处先介绍主从dns服务器，即master和slave类型。

在只有一台dns服务器时，所有的dns解析过程都由这台dns服务器负责，压力极大。而且极不安全，因为这台dns服务器一垮掉，所有的解析服务都停止，整个网站也就垮了。

无论是出于负载均衡考虑，还是数据安全可靠的考虑，至少都应该配置2台或2台以上的dns服务器，其中必须有一台是master服务器，其余的是slave服务器。

master和slave服务器都可以配置成向外提供解析服务。但slave上的区域数据从何而来？不是自行书写区域数据文件而来，而是从master服务器上复制而来。从master复制区域数据到slave的过程，dns术语称之为"区域传送"。

### 主、从初体验

配置主从的方式很简单，仅仅只需在待设置为slave的服务器上设置zone类型为slave，并给定一个masters指令指向其所属master即可。例如在172.16.10.9上安装好bind软件后，配置其/var/named.conf，加上以下一项zone配置：

```
zone "longshuai.com" IN {
    zone slave;
    masters { 172.16.10.15; };
    file "slaves/db.longshuai.com.bak";
};
```

这表示172.16.10.9是172.16.10.15上"longshuai.com"的slave。

注意，上面的示例中使用file指令，但实际上**slave是可以不要区域数据文件的**，它从master上传送复制区域数据后会将其缓存下来，并从缓存中提供查询解析服务。**如果在slave区域内指定file指令，则表示在区域传送时还将备份一份数据到file指定的文件中**，所以该文件对named组要求有写权限。在安装bind后，在/var/named目录下自动生成了一个/var/named/slaves目录，其属组和权限已经设置好，正适合放置区域传送的备份文件。如果自定义存放备份文件的路径，则其存放目录属组要求为named，且属组有rwx权限。

重启slave服务器。

```
[root@xuexi ~]# service named restart
```

**slave每次重启都会自动完成一次区域传送。**查看master和slave上的日志信息：

master上：

```
[root@xuexi ~]# tail /var/log/message
Aug 14 11:03:29 xuexi named[69500]: client 172.16.10.9#37352 (longshuai.com): transfer of 'longshuai.com/IN': AXFR started
Aug 14 11:03:29 xuexi named[69500]: client 172.16.10.9#37352 (longshuai.com): transfer of 'longshuai.com/IN': AXFR ended
```

AXFR表示完全区域传送，还有增量区域传送IXFR。**完全区域传送是传送整个区域数据文件，增量区域传送则只传送区域数据文件中发生改变的部分。**对于完全区域传送和增量区域传送，在bind9上可以完全不用配置，一切都可采用默认值，它会自动决定何时完全、何时增量传送。

slave上：

```
[root@xuexi ~]# tail /var/log/message
Aug 14 11:03:29 xuexi named[5113]: running
Aug 14 11:03:29 xuexi named[5113]: zone longshuai.com/IN: Transfer started.
Aug 14 11:03:29 xuexi named[5113]: transfer of 'longshuai.com/IN' from 172.16.10.15#53: connected using 172.16.10.9#37352
Aug 14 11:03:29 xuexi named[5113]: zone longshuai.com/IN: transferred serial 1
Aug 14 11:03:29 xuexi named[5113]: transfer of 'longshuai.com/IN' from 172.16.10.15#53: Transfer completed: 1 messages, 8 records, 227 bytes, 0.001 secs (227000 bytes/sec)
```

需要说明的是，这样配置的主从服务器只会完全区域传送。原因稍后解释。

### 配置完整的主从服务器

既然slave也是域数据解析的负责人(尽管它是从的)，因此**在master上的区域数据文件中应该添加slave的ns记录，否则外部没法找到slave这个不完整的dns服务器来帮忙解析，且master也无法找到slave**。

另外，不在master区域数据文件中添加slave的ns记录，其只能进行完全区域传送，无法增量区域传送。因为不在主服务器的区域文件中添加从服务器的ns记录，将不表明这台slave是DNS服务器。之所以不添加ns记录时slave也会进行完全区域传送，仅仅是因为slave服务器做了指向master的配置，所以只能每次slave上启动named时才会实现完全区域传送，master重启named时是不会进行区域传送的(因为master找不到slave)。

所以，要配置完整的主从服务器，只需在前面的基础上，再在master的区域数据文件中加上slave的ns记录和对应的A记录即可，别忘了修改SOA记录的序列版本号。

```
[root@xuexi ~]# vim /var/named/db.longshuai.com
$TTL 6h
@            IN  SOA    dnsserver   xyz ( 111 3h 1h 1w 1h )
@            IN  NS     dnsserver
@            IN  NS     dnsserver1
 
dnsserver    IN  A      172.16.10.15
dnsserver1   IN  A      172.16.10.9
www          IN  A      172.16.10.16
ftp          IN  A      172.16.10.17
mydb         IN  A      172.16.10.18
 
www1         IN  CNAME  www
```

重启master dns服务以重新编译区域数据文件。重启master时，master会发送notify信息给slave，通知slave来复制区域数据。

```
[root@xuexi ~]# systemctl restart named    
[root@xuexi ~]# tail /var/log/messages
Aug 14 11:53:28 xuexi named[69794]: running
Aug 14 11:53:28 xuexi named[69794]: zone 10.16.172.in-addr.arpa/IN: sending notifies (serial 0)
Aug 14 11:53:28 xuexi named[69794]: zone longshuai.com/IN: sending notifies (serial 111)
Aug 14 11:53:28 xuexi systemd: Started Berkeley Internet Name Domain (DNS).
Aug 14 11:53:28 xuexi named[69794]: client 172.16.10.9#34264 (longshuai.com): transfer of 'longshuai.com/IN': AXFR-style IXFR started
Aug 14 11:53:28 xuexi named[69794]: client 172.16.10.9#34264 (longshuai.com): transfer of 'longshuai.com/IN': AXFR-style IXFR ended
```

由于配置了完整的主从结构，且修改了master区域数据文件上的序列号，所以区域传送的类型是增量传送。

回到slave上，查看区域数据文件和日志信息：

```
[root@xuexi ~]# cat /var/named/slaves/db.longshuai.com.bak   
$ORIGIN .
$TTL 21600      ; 6 hours
longshuai.com           IN SOA  dnsserver.longshuai.com. xyz.longshuai.com. (
                                111        ; serial
                                10800      ; refresh (3 hours)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                3600       ; minimum (1 hour)
                                )
                        NS      dnsserver.longshuai.com.
                        NS      dnsserver1.longshuai.com.
$ORIGIN longshuai.com.
dnsserver               A       172.16.10.15
dnsserver1              A       172.16.10.9
ftp                     A       172.16.10.17
mydb                    A       172.16.10.18
www                     A       172.16.10.16
www1                    CNAME   www
[root@xuexi ~]# tail /var/log/messages
Aug 14 11:53:28 xuexi named[5279]: client 172.16.10.15#7274: received notify for zone 'longshuai.com'
Aug 14 11:53:28 xuexi named[5279]: zone longshuai.com/IN: Transfer started.
Aug 14 11:53:28 xuexi named[5279]: transfer of 'longshuai.com/IN' from 172.16.10.15#53: connected using 172.16.10.9#34264
Aug 14 11:53:28 xuexi named[5279]: zone longshuai.com/IN: transferred serial 111
Aug 14 11:53:28 xuexi named[5279]: transfer of 'longshuai.com/IN' from 172.16.10.15#53: Transfer completed: 1 messages, 10 records, 268 bytes, 0.001 secs (268000 bytes/sec)
Aug 14 11:53:28 xuexi named[5279]: zone longshuai.com/IN: sending notifies (serial 111)
```

还可以配置slave从另一个slave上复制区域数据，这种情况下，master是slave1的主，slave1是slave2的主。如下图：

![](/img/linux/1699368247466.png)

配置方式一样很简单，只需将slave2主机的named.conf中的zone区域配置成如下格式：

```
zone "domain" IN {
    type slave;
    masters { slave1_ip };
};
```

当然，master的区域数据文件中应该制定slave2的ns记录和A记录。

### 何时进行区域传送

有3种情况会进行区域传送动作：

(1).slave重启。此时slave主动去master上获取区域数据。

(2).master重启。此时master会发送notify消息给slave。但要求区域数据文件中定义了slave的ns记录及其A记录，否则找不到slave，也就联系不上slave。

(3).区域数据文件SOA记录中定义的refresh时长到了。也就是说正常运行时，每隔一段时间，slave都会主动去联系master，并从其上复制区域数据。

在named.conf中可以定义notify发送给谁以及接收谁的notify，这个细节和过程篇幅稍大，下一节说明。此处先说明"allow-transfer"指令的用法。例如：

```
zone "movie.edu" {
     type master;
     file "db.movie.edu";
     allow-transfer { 192.249.249.1; 192.253.253.1; 192.249.249.9; 192.253.253.9; };
};
```

默认情况下，allow-transfer的值为any，表示允许任何人都可以从此主机上执行区域传送。实际上，**应该设置主dns服务器只允许slave服务器来区域传送，并设置slave服务器不允许任何人区域传送**，这样就最大程度保证了区域数据不泄漏。

可以手动使用dig命令强制区域传送，只需使用-t指定区域传送的类型即可，如下：

```
dig -t AXFR
dig -t ixfr=N
```

在指定增量区域传送时，需要指定序列号，只有比N大的序列号才会传送。

### notify通知

主DNS服务器可以通知从DNS服务器进行区域传送。

notify是这样工作的：**当主DNS服务器重启了DNS服务时，则通知所有slave DNS服务器来更新区域数据。**(有些地方模糊地说SOA序列号发生改变也会发送notify，其实不然，因为区域数据文件需要编译加载到内存，不重启named，即使改变了序列号也无效)

它是这样判断哪些是主DNS服务器哪些是从DNS服务器的：**找到区域数据文件中的所有NS记录，并排除本机以及SOA记录中MNAME的那个服务器(SOA记录的MNAME列即IN关键字后的一列，一般就是主DNS服务器)，剩余的都是从DNS服务器。**

主DNS为每个自定义的区都发送notify声明给所有从服务器，告知其哪些区改变了，slave服务器接收到notify声明后响应主DNS服务器告知它已经收到了通知，然后slave服务器向主DNS服务器发起查询，以确定notify声明中所通告的区的SOA记录是否真的发生了改变，如果SOA发生了改变，则进行区域传送，如果没有改变则不进行传送。

为什么从DNS服务器接收到notify声明后还要再次查询主服务器上SOA记录来确认呢？第一是因为要比较序列号，决定是否要传送，以及要完全传送还是增量区域传送，第二是因为有些人可能会发送假冒的notify声明给从DNS服务器，从而导致多余的区域传送。

在bind 9中，**从DNS服务器收到notify声明后并进行了区域传送后，从DNS服务器之间还会各自发送notify给对方，因为有些更低层次的从服务器不会接受主DNS服务器发送的notify。**

例如下图中，movie.edu的主DNS服务器为terminator.movie.edu，还有两台从服务器分别为wormhole.movie.edu和zardoz.movie.edu，它们都是movie.edu域的权威服务器，在该域的区域文件中都有NS记录。

![](/img/linux/1699368270165.png)

当terminator.movie.edu重启了named服务时，该机器会发送notify声明给wormhole.movie.edu和zardoz.movie.edu，当这两台从服务器进行了区域传送后，它们还会互发notify声明给对方，但是由于它们都不是对方的主DNS服务器，所以会忽略对方发送的notify声明。

在以前的bind 8中，以下为terminator.movie.edu机器syslog的日志记录，第一条信息表示terminator服务器发送了notify给两台从DNS服务器(2 NS), 它通知从服务器要更新的区域为movie.edu，序列号为"2000010958"。

```
Oct 14 22:56:34 terminator named[18764]: Sent NOTIFY for "movie.edu IN SOA
2000010958" (movie.edu); 2 NS, 2 A
```

下面两行表示两台从服务器收到notify后给出的notify响应。但是在bind 9中好像不会自动记录notify信息。

```
Oct 14 22:56:34 terminator named[18764]: Received NOTIFY answer (AA) from 192.249.249.1 for "movie.edu IN SOA"
Oct 14 22:56:34 terminator named[18764]: Received NOTIFY answer (AA) from 192.249.249.9 for "movie.edu IN SOA"
```

更形象的例子如下：a是某个区域的主DNS服务器，b和c都是该区域的从DNS服务器，但是b是a的从，c是b的从，所以在这里，SOA记录中mname列代表的机器是a。

![](/img/linux/1699368322413.png)

当a重启named时，发送notify声明给除了本机和SOA中的mname代表的机器即b和c， b和c都会收到notify，但是c会忽略a的notify信息，因为a不是它的主DNS服务器，它只会接受b发送的notify。而b收到a的notify后会从a上进行区域传送，传送完毕后发送notify给除了本机和a(因为它是SOA记录中mname列的值)之外的机器，即只发送notify给c，c收到b的notify后向b发起查询确认是否需要进行传送，传送完毕后继续发送notify给除了本机和a之外的机器即b机器，b收到c的notify信息后对其忽略，因为c不是它的主DNS服务器。

在bind 8和bind 9中，notify默认是打开的，注意notify是写在主DNS服务器的named.conf中的，它的作用对象默认是所有的从服务器。使用下面的语句可以关闭：

```
options {
    notify no;
};
```

也可以将notify的开和关字句写在某个区域中，这样该设置会覆盖全局配置。

```
zone "fx.movie.edu" {
    type master;
    file "db.fx.movie.edu";
    notify no;
};
```

还可以使用also-notify来定义notify列表，这样除了会发送notify通知给从服务器还会发送给列表中定义的机器。also-notify字句可以写在某个区中，也可以写在全局配置options字句中。

```
zone "fx.movie.edu" {
    type slave;
    file "bak.fx.movie.edu";
    notify yes;
    also-notify { 15.255.152.4; };
};
```

默认从服务器只接受来自其主DNS服务器的notify信息，非主DNS服务器的信息都会忽略，但是可以使用allow-notify字句定义可以接受其他从服务器的notify信息。例如a是b和c的主，如果在c上定义`allow-notify { b_IP; };`，那么它也会接受b的notify信息。

### 区域文件备份的重要性

在前面定义slave时说过，其实zone中的file指令是可以省略的，因为slave从master复制的区域数据是存放在缓存中的。如果明确指定了file指令，则表示复制区域数据时，还将其备份到指定文件中。

这个区域数据备份文件非常重要，其重要性体现在slave联系不上master时。当slave重启了named服务或者refresh时间到了，但又联系不上master时，slave就缺少了区域数据，也就没法向外提供解析服务。这意味着，dns服务完全垮了。

### rndc控制dns服务器

rndc是远程控制DNS服务器的工具，默认在装bind时已经装好。

控制dns服务器实际上是通过特殊的通道给dns服务程序发送控制消息，在bind9上只支持tcp端口号类型的控制通道，所以待控制的dns服务器上需要开启好一个端口号。这个端口号是通过/etc/named.conf中的controls指令来设置的。

rndc有几个很好的命令工具，如清空DNS缓存的命令rndc flush，该命令还可以指定清空哪个域的缓存。此外还有rndc dumpdb可以查询缓存等功能，这些功能无需配置就可直接使用。但有些rndc的功能需要配置后才能使用，例如控制dns服务器。

![](/img/linux/1699368358912.png)

### named.conf中的controls指令

在/etc/named.conf中，可以使用controls指令来设置接收控制消息的通道，以及允许控制本机的控制者。定义方式如下：

```
controls {
    inet local_ip port PORT_NUM  allow { control_ip_list; } keys { "rndc-key"; };
};
```

在上述格式中：

(1).local\_ip和PORT\_NUM设置的是dns服务器开启的tcp通道，表示监听在本机某个IP地址local\_ip上的PORT\_NUM端口上。

(2).allow关键字定义允许连接本通道的主机列表，也就是限定谁能控制本机dns服务器。

(3).keys关键字定义的是连接本通道时需要进行密钥认证，只有认证通过的才能成功连接通道。这个key在后面介绍rndc时会说明。

其中local\_ip可以使用`*`表示监听在本机所有地址上，port可以省略，默认监听端口为953。另外，key可以放在某个文件中，然后使用include指令导入到/etc/named.conf中。

### 控制端生成rndc.conf

使用工具rndc-confgen可以生成rndc的配置文件rndc.conf，同时也会生成一个md5的key。

```
[root@xuexi ~]# rndc-confgen >/etc/rndc.conf    
```

这个命令在远程连接工具上执行可能会很慢很慢，在Linux机器本身执行会很快。解决办法是创建一个随机数文件，然后使用`rndc-confgen -r`指定随机数文件。

如下：/tmp/b.ran中是一堆很长的随机输入的字母数字等。

```
[root@xuexi ~]# cat /tmp/b.ran
adsfajklgfadjgiowjkdsnmcajsdjfljkasljdfqiojfkljdasldfnvuaehiqohhjxahqpewipqnvxjlka
```

当然，也可以使用安装bind时生成的genrandom工具创建随机数文件。如下，其中2表示b.ran的大小为2kB

```
[root@xuexi ~]# genrandom 2 /tmp/b.ran
```

然后再生成rndc.conf就很快了。

```
[root@xuexi ~]# rndc-confgen -r /tmp/b.ran >/etc/rndc.conf
```

查看rndc.conf里的内容。

```
[root@xuexi ~]# cat /etc/rndc.conf
# Start of rndc.conf      # rndc.conf开始
key "rndc-key" {
        algorithm hmac-md5;
        secret "QDCyDaU8El7quzv3vB3z9A==";   # rndc-confgen生成的密钥
};

options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};
# End of rndc.conf         # rndc.conf结束
# 上面一段是/etc/rndc.conf的配置，下面的全是注释了的，要放入待控制主机的named.conf中进行配对

# 下面一段是复制段，需要复制到待控制的dns服务器的named.conf中，并按需修改controls指令
# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
#       algorithm hmac-md5;
#       secret "QDCyDaU8El7quzv3vB3z9A==";   #这个和上面都密钥是相同的，能够配对
# };
# 
# controls {
#       inet 127.0.0.1 port 953
#               allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf
```

上面的/etc/rndc.conf文件中分为两段，一段是保留在rndc.conf中的，另一段是需要复制到待控制机器的named.conf中的。为了方便称呼，下文将以"复制段"来表示需要复制的段。

以下是/etc/rndc.conf的配置说明：

(1).options段用于配置默认项。可以配置的指令包括default-server、default-port、default-key，表示当没有任何地方指定这些项的值时将采用这些默认值。

(2).server段用于配置待控制的dns服务器。server关键字后接的是dns服务器的主机名或ip地址。在此段落内，可以配置的指令包括：key、port、addresses。其中key表示连接此server时将使用该key进行配对，port表示要连接的dns服务器的rndc端口号，addresses指定要连接的dns服务器地址，当使用了该指令时将替代server关键字后的主机名或ip地址，addresses后可以紧跟着端口号。

(3).key段定义key值。只有两个指令，一个是algorithm，目前只支持hmac-md5，另一个指令是secret，表示该key段所使用的key。secret段加密的key可以使用rndc-confgen生成，只需使用不同的随机数即可。

以下是在172.16.10.16上设置的rndc.conf，用于控制172.16.10.9和172.16.10.15这两台dns服务器。

```
key "rndc-key" {
        algorithm hmac-md5;
        secret "QDCyDaU8El7quzv3vB3z9A==";
};

options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};

server localhost {
    key    "rndc-key";
};

server 172.16.10.9 {
    key  "rndc-key";
    port 953;
};

server 172.16.10.15 {
    key "rndc-key";
    port 953;
}; 

# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
#       algorithm hmac-md5;
#       secret "QDCyDaU8El7quzv3vB3z9A==";
# };
# 
# controls {
#       inet 127.0.0.1 port 953
#               allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf
```

然后只需将上述加粗标红的部分(注意要取消注释)复制到172.16.10.9和172.16.10.15主机的/etc/named.conf中，并稍作修改，如下：

```
key "rndc-key" {
      algorithm hmac-md5;
      secret "QDCyDaU8El7quzv3vB3z9A==";
};

controls {
      inet * port 953
              allow { 172.16.10.16; } keys { "rndc-key"; };
};
```

然后重启待控制端的named服务。另外，为了控制本地主机，还应该复制到本机的/etc/named.conf中并做修改，然后重启named，当然，如果不用控制本机则无需配置。

然后使用rndc去测试是否能控制远程dns服务器。

```
[root@xuexi ~]# rndc -c /etc/rndc.conf -s 172.16.10.15 status
version: 9.9.4-RedHat-9.9.4-50.el7_3.1 <id:8f9657aa>
CPUs found: 4
worker threads: 4
UDP listeners per interface: 4
number of zones: 103
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is OFF
recursive clients: 0/0/1000
tcp clients: 0/100
server is up and running
```

这表示控制成功。如果连接失败，可能会给出如下信息：

```
`rndc: connection to remote host closed``This may indicate that``* the remote server is using an older version of the ``command` `protocol,``* this host is not authorized to connect,``* the clocks are not synchronized, or``* the key is invalid.`
```

给出了几个原因：

(1).版本不一样。这个也会出问题的，亲测CentOS 7上的bind rndc无法连接CentOS 6上装的bind dns。

(2).key无法配对。

(3).时间未同步。

(4).key已经失效。

### rndc命令

 rndc命令功能非常强大，有很多很好用的功能。此处只列出它的命令列表并标出几个常用的命令，更多的用法可以自行探索。

```
[root@xuexi ~]# rndc
Usage: rndc [-b address] [-c config] [-s server] [-p port]
        [-k key-file ] [-y key] [-V] command

command is one of the following:

  reload        Reload configuration file and zones.
  reload zone [class [view]]
                Reload a single zone.
  refresh zone [class [view]]
                Schedule immediate maintenance for a zone.
  retransfer zone [class [view]]
                Retransfer a single zone without checking serial number.
  freeze        Suspend updates to all dynamic zones.
  freeze zone [class [view]]
                Suspend updates to a dynamic zone.
  thaw          Enable updates to all dynamic zones and reload them.
  thaw zone [class [view]]
                Enable updates to a frozen dynamic zone and reload it.
  notify zone [class [view]]
                Resend NOTIFY messages for the zone.
  reconfig      Reload configuration file and new zones only.
  sign zone [class [view]]
                Update zone keys, and sign as needed.
  loadkeys zone [class [view]]
                Update keys without signing immediately.
  stats         Write server statistics to the statistics file.
  querylog      Toggle query logging.
  dumpdb [-all|-cache|-zones] [view ...]
                Dump cache(s) to the dump file (named_dump.db).
  secroots [view ...]
                Write security roots to the secroots file.
  stop          Save pending updates to master files and stop the server.
  stop -p       Save pending updates to master files and stop the server
                reporting process id.
  halt          Stop the server without saving pending updates.
  halt -p       Stop the server without saving pending updates reporting
                process id.
  trace         Increment debugging level by one.
  trace level   Change the debugging level.
  notrace       Set debugging level to 0.
  flush         Flushes all of the server's caches.
  flush [view]  Flushes the server's cache for a view.
  flushname name [view]
                Flush the given name from the server's cache(s)
  status        Display status of the server.
  recursing     Dump the queries that are currently recursing (named.recursing)
  tsig-list     List all currently active TSIG keys, including both statically
                configured and TKEY-negotiated keys.
  tsig-delete keyname [view]
                Delete a TKEY-negotiated TSIG key.
  validation newstate [view]
                Enable / disable DNSSEC validation.
  addzone ["file"] zone [class [view]] { zone-options }
                Add zone to given view. Requires new-zone-file option.
  delzone ["file"] zone [class [view]]
                Removes zone from given view. Requires new-zone-file option.
  *restart      Restart the server.
```

## 子域

### 子域的原理分析

顶级域".com"是根域"."的子域，"baidu.com."又是".com."的子域。为何必须要向isc申请才能注册顶级域名下的子域如"longshuai.com"呢？当有了"longshuai.com"后，如何创建和设置它的子域"video.longshuai.com"呢？

不管是不是子域，只要它是一个域，就必须要有dns服务器来负责该域的解析。域的灵魂在于其区域数据文件(不是named.conf，它只是配置named程序工作的文件)，只有区域数据文件中才提供了域所需的所有数据，**要让dns服务器能够正常工作，域数据必须要完整且正确，完整的域数据至少要存储SOA记录，ns记录以及ns对应的a记录。**如果没有ns记录，则表示该域缺少dns服务器，这是不可能的。而缺少ns对应的a记录则能知道dns服务器的存在以及其主机名，却找不到dns服务器，因为没有它的ip地址。更重要的是缺少了soa记录后，它指定不了哪个dns服务器是主dns服务器，以及一系列的附加属性(序列号、各种重试缓存时间等)。更关键的是soa是起始授权机构，**缺少了soa记录表明父域没有对该域授权，该域没有自主权，它只是父域下的一部分(可能只是主机名上多了一截，例如wuda.video.longshuai.com)，所有的解析工作都需要由父域来完成，只有有了soa记录，才表明父域授权了该域，该域可以享受起始授权的权利，也就是具有自主权，可以独立完成解析工作。**子域，同样是域的概念，因此它的区域数据文件也一样要满足这些不是条件的条件。

再回顾下dns解析流程，客户端向dns服务器发送递归查询后，dns服务器会查询根域，根域会告诉dns服务器负责解析顶级域的服务器地址，dns服务器再向顶级域服务器发起查询，顶级域服务器再告诉dns服务器再下一层次域的服务器地址，依次下去，直到找到最终主机地址并返回给客户端。在这个解析流程中，父域总是将子域dns服务器的地址告诉dns服务器。所以，父域是知道子域dns服务器的ip地址的，因此**在父域的区域数据文件中，必然要指定子域dns服务器的A记录**。另外，父域如何其内主机是普通的主机还是子域dns服务器？总不能让子域dns服务器被父域当成普通主机吧？区分的方法是**在父域的区域数据文件中使用NS记录来存储子域dns服务器信息**。这样一来，子域dns服务器不仅在父域的区域数据文件中有了NS记录，还有了A记录，父域就能将子域信息返回给查询发起者。

其实在理论上，父域的区域数据文件中是可以不存储子域dns服务器A记录的，因为NS记录已经区分了它是dns服务器，只不过只有NS记录没有A记录是找不到该子域dns服务器的。

根据上述分析，应该就能明白在互联网上申请域名并向外界提供解析时，实际上是申请机构的区域数据文件中添加NS记录和NS对应的A记录，仅此而已。由此也知，根域名和顶级域名的区域数据文件是无比巨大的，在其中存放了无数的NS记录和A记录。

### 创建子域

根据上面的分析，在已有"longshuai.com"域时，假设要创建其子域"video.longshuai.com"，需要满足以下两个条件：

(1)."longshuai.com"的区域数据文件中，需要添加子域"video.longshuai.com"的dns服务器的NS记录和A记录，如果该子域有多个dns服务器，则需要添加多个NS记录和对应的A记录。

(2).子域master dns服务器上的区域数据文件中，需要书写SOA记录，NS记录和NS对应的A记录。

所以，配置子域的过程其实很简单。但实际上，要创建子域是一个比较精细的活，不仅要考虑子域的域名(要让子域名有意义、易读、没有二义性等，而不是随便取的)，还要考虑创建多少个子域，如何划分子域(是按地理位置划分、部门划分、功能划分还是其他方式划分)，当然这几个都不是技术层面的问题。就技术层面上考虑，创建子域时，需要考虑父域数据到子域的迁移，还要考虑反向查找区域数据文件的问题(反向查找区域的父域总是in-addr.arpa，所以根本不需要考虑子域反向查找区域数据文件的特殊性)。

在真正开始配置子域之前，仍然需要重复一遍：子域的区域数据文件中是可以没有SOA记录的，只不过此时子域没有得到父域授权，也就没有自主权，无法提供解析功能。这意味着这不是一个子域，而是父域的一部分，就相当于在父域的区域数据文件中使用`$INCLUDE`一样。可以认为授权了的子域是父域将自己的孩子送到了它们该到的地方，它们自己能够独挡一面，没有授权的子域实际上只是住在了父域的隔壁，它没有独立解析的能力，一切问题仍然需要父域来负责解答。如下图：

![](/img/linux/1699368536149.png)

现在要创建子域"video.longshuai.com"，新提供一台负责子域解析的dns服务器，结构如上图右边。

先修改父域的区域数据文件，加上子域的NS记录和NS对应的A记录。

```
[root@xuexi ~]# cat /var/named/db.longshuai.com    
$TTL 6h
@            IN  SOA    dnsserver   xyz ( 111 3h 1h 1w 1h ) 
@            IN  NS     dnsserver
@            IN  NS     dnsserver1

dnsserver    IN  A      172.16.10.15
dnsserver1   IN  A      172.16.10.9
www          IN  A      172.16.10.16
ftp          IN  A      172.16.10.17
mydb         IN  A      172.16.10.18

www1         IN  CNAME  www

video        IN  NS     ns1.video
ns1.video    IN  A      172.16.10.20
```

重启父域named服务，或者使用`rndc reload zone`文件。

```
[root@xuexi ~]# rndc -c /etc/rndc.conf reload longshuai.com
zone reload up-to-date
```

再配置子域的named.conf以及区域数据文件。注意子域的区域数据文件中，要配置SOA、NS以及NS对应的A记录。

```
[root@xuexi ~]# cat /etc/named.conf
options {
        directory       "/var/named";
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "video.longshuai.com" IN {
    type master;
    file "db.video.longshuai.com";
};

include "/etc/named.rfc1912.zones";
[root@xuexi ~]# cat /var/named/db.video.longshuai.com 
$TTL 6h
@       IN    SOA   ns1    xyz    ( 1 6h 3h 1d 1h )
        IN    NS    ns1

ns1     IN    A     172.16.10.20
```

重启子域dns服务器的named服务器。

```
[root@xuexi ~]# systemctl restart named
```

至此，子域就授权完成了，于是可以测试查询子域中的主机时，是由子域的dns服务器而不是父域来解析的。测试的方法是查询子域中任意主机，如果在dig结果中的"AUTHORITY SECTION"段给出的是父域的dns服务器，则表明是父域进行解析的，如果给出的是子域的dns服务器，则表明是子域进行解析的。例如，dig一个子域中没有的主机：

```
[root@xuexi ~]# dig -t a xyz.video.longshuai.com. @172.16.10.15

; <<>> DiG 9.9.4-RedHat-9.9.4-37.el7 <<>> -t a xyz.video.longshuai.com. @172.16.10.15
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 52731
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;xyz.video.longshuai.com.       IN      A

;; AUTHORITY SECTION:
video.longshuai.com.    3394    IN      SOA     ns1.video.longshuai.com. xyz.video.longshuai.com. 1 21600 10800 86400 3600

;; Query time: 0 msec
;; SERVER: 172.16.10.15#53(172.16.10.15)
;; WHEN: Tue Aug 15 04:03:25 CST 2017
;; MSG SIZE  rcvd: 92
```

上面结果中的权威答案是video.longshuai.com给的，说明子域配置是完整、正确的。

## 智能DNS——视图view

智能dns?仅就bind来说，其视图特性提供的就是智能dns的功能。使用view可以实现根据客户端来源为不同来源的客户端展现同一个区域的不同配置，不同来源的客户端解析同一个区域也可能得到不同的结果。例如，公司两台web服务器web1和web2（它们是完全相同的内容），使用电信网的客户端对web的请求让其访问到web1上，使用联通网的客户端对web的请求让其访问到web2上。通过判断网络来源，让其选择合适的线路访问可以加快访问速度，这是常用的功能。

要使用view功能，最好配合acl来制定什么客户端解析到哪去。acl指令是仅有的几个不能定义在view中的指令之一。

```
[root@xuexi ~]# vim /etc/named.conf
acl telecom { 172.16.10.0/24;127.0.0.0/8; };
options {
        directory "/var/named";
};
view telecom {
        match-clients {telecom;};
        zone "longshuai.com" IN {
                type master;
                file "telecom.longshuai.com.zone";
        };
};
view unicom {
        match-clients {any;};
        zone "longshuai.com" IN {
                type master;
                file "unicom.longshuai.com.zone";
        };
};
[root@xuexi named]# vim telecom.longshuai.com.zone 
$TTL 43200
@       IN      SOA     ns1  admin    ( 1 1H 5M 2D 6H )
        IN      NS      ns1
ns1     IN      A       172.16.10.15
www     IN      A       172.16.10.4         ; 电信的是10.4
[root@xuexi named]# chgrp named telecom.longshuai.com.zone 
[root@xuexi named]# chmod 640 telecom.longshuai.com.zone 
[root@xuexi named]# cp -a telecom.longshuai.com.zone unicom.longshuai.com.zone
[root@xuexi named]# vim unicom.longshuai.com.zone 
$TTL 43200
@       IN      SOA     ns1  admin    ( 1 1H 5M 2D 6H )
        IN      NS      ns1
ns1     IN      A       172.16.10.3
www     IN      A       172.16.10.8         ; 网通的是10.8
[root@xuexi named]# systemctl restart named
```

然后进行测试即可。

注意事项：

(1).所有的zone都必须要定义在view中，尽管默认的named.conf根本没有定义view，但是此时所有的zone都定义在了一个隐含的默认的视图中。

(2). view中match-clients指令的匹配方式是从前向后匹配的，如果在第一个view中匹配了，则后面定义的view将不会生效，所以定义的view的先后顺序是很重要的。

(3).绝大多数named.conf中的指令都能写在view中，只有很少量的指令不允许，例如acl指令。对于本该封装在options中的指令，如果想定义在view中，则不应该在view中使用options，因为options定义的是全局默认值，配置文件中只能出现一次，所以可以直接在view中写指令，这样会覆盖全局options。

(4).不同的view中定义的相同的zone，它们使用的区域文件一般不同（并非必须不同），否则就没有自定义view的必要。

<a name="cdn_dns"></a>

## CDN和DNS的关系

下图是加入了CDN节点的资源请求过程图。

![](/img/linux/1699368627243.png)

大致过程如下：

(1).客户端向A公司总部的DNS服务器发起www.a.com的查询请求。

(2).DNS服务器对www.a.com设置了CNAME记录指向CDN节点www.a.cdn.com，于是客户端再去查找www.a.cdn.com的IP地址。注意，域名已经变成了cdn.com。

(3).根据cdn.com的域名，客户端最终会查找到CDN的DNS服务器上。该DNS服务器通过智能DNS(例如BIND的视图功能)为A公司按网络类型(电信、网通)、地理位置远近设置了不同的区域数据文件，每个区域数据文件中设置了对应地区、网络类型的一条或多条A记录指向A公司部署在该地区的缓存服务器。

(4).客户端根据CDN-DNS返回的A记录IP地址，直接进行访问。这时访问的一般是缓存服务器，如果缓存未命中，则缓存服务器负责从总部服务器中请求数据并返回给客户端，同时缓存下来。

甚至，A公司总部的DNS服务器上，还可以直接智能选择，并CNAME到不同区域的的服务器节点上，如下图，注意区域配置文件的设置内容包括CNAME和A记录。这时就无需向服务商购买CDN服务。

![](/img/linux/1699368646265.png)

## DNS日志系统

DNS的日志系统有两个概念：通道(channel)和类别(category)。

channel定义日志向哪里发送：是发送到syslog日志系统中，还是发送到某个自定义的文件，抑或是发送到named的标准输出，还可以发送到位存储桶(bit bucket)。

category定义哪些类的信息需要写入日志。

每种类别的数据可以被发送给一个或多个通道。例如统计类别的信息发送到系统日志通道且发送到自定义日志文件通道，而查询类别只发送到自定义日志文件通道。

![](/img/linux/1699368672301.png)

通道允许你记录不同级别的日志信息，从严重到不严重有七种级别：critical;error;warring;notice;info;debug[level]和dynamic。其中前5种级别和系统日志syslog系统相同，后两种(debug和dynamic) 是BIND独有，且bebug还可以按照level进行细分。

但是写日志是一项非常消耗性能的操作，所以默认都是定义在info级别上。在正常使用环境中，除了调试时可能需要记录debug或者dynamic级别信息，其余至少都记录到info级别甚至更严格的级别。

以下是named.conf中的一个日志定义示例。

```
loggin {
    channel my_syslog {          /* 定义日志通道my_syslog */ 
        syslog daemon;           /* my_syslog通道的日志写入到系统日志syslog，并指定使用daemon工具记录 */ 
        severity warning;        /* 该通道只记录严重级别为warning及以上的信息 */ 
    };
    channel my_file {           /* 定义第二个日志通道my_file */ 
        file "log.msgs";        /* my_file通道的日志写入到自定义的文件log.msgs中 */ 
        severity info;          /* 该通道记录日志级别为info 及以上的信息 */ 
    };
    category statistics {my_syslog;my_file;};  /* 统计类别statistics的信息使用my_syslog和my_file通道记录 */ 
    category queries {my_file;};               /* 查询类别queries的信息记录使用my_file通道记录 */ 
    category default {null;};                  /* 除了上述两个自定义的类别，其余绝大部分的类别的信息都丢弃 */   
//category default { my_file; };           /* 或者不丢弃，将其记录到my_file通道指定的文件也可以 */ 
};
```

在上述示例中，有几个关键字：

- loggin是named.conf中标记日志系统的关键字。

- channel是定义通道的关键字。

- severity是定义记录什么日志级别的关键字。

- category是定义信息类别使用哪个通道记录的关键字。

下图是DNS日志系统的语法。

![](/img/linux/1699368698355.png)

**(1).channel的解释**

在channel字句中，指定通道的路径（即日志记录的位置）和记录的日志级别，还可以指定在日志文件中是否也记录类别信息、日志级别以及该条日志的记录时间。

通道的有四种位置可选：

- syslog:使用系统日志来记录，同时需要指定使用哪种工具(facility)来记录。

  有kern、user、mail、daemon、auth、syslog、lpr、news、uucp、cron、authpriv、ftp、local0、local1、local2、local3、local4、local5、local6 和local7这么多种工具可选。默认是daemon，也建议使用daemon。

- file:使用自定义的文件路径。可以指定该日志文件有几个版本(versions)和多大(size)时就进行轮替，不指定大小限制时日志文件将无限制增长。

  例如以下定义方式：每个日志文件增长到10M大小就创建新的文件来记录新的日志，最多记录3个版本的日志文件，当第三个日志文件my_logs2也到10M后将删除最早的日志文件继续记录。

  ```
  channel my_file {
      file my_logs versions 3 size 10M;
      severity info;
  };
  ```

- null:使用null方式的通道将把使用该通道来记录的信息全部丢弃。

- stderr:使用stderr方式的通道将把使用该通道来记录的信息输出到stderr。

定义完通道路径后还需要指定记录的日志级别。级别有7种：`critical;error;warring;notice;info;debug[level]和dynamic`。其中前5种级别和系统日志syslog系统相同，后两种(debug和dynamic) 是BIND独有，且bebug还可以按照level进行细分。

但是写日志是一项非常消耗性能的操作，所以默认都是定义在info级别上。在正常使用环境中，除了调试时可能需要记录debug或者dynamic级别信息，其余至少都记录到info级别甚至更严格的级别。

一个通道必须定义的两项就是上面所说的通道路径和日志级别。还有附加可选项可以定义：print-time boolean;print-severity boolean和print-category boolean。分别表示在记录日志时是否同时记录日志的记录时间、日志的级别和日志的类别。但是要注意的是，如果通道使用syslog来记录，那么print-time boolean是件无意义的事，因为syslog系统自身会加入时间戳。

**(2).类别的解释**

BIND 9 中有很多种类别：
```
default :匹配没有自定义的绝大多数类别。但是还有一些信息是不属于任何类别的信息，
        :这部分信息无法使用default来匹配，而是用general类别来匹配。
general :匹配那些不明确属于哪个类别的信息。
client  :客户端请求信息。
config  :配置文件分析和处理信息。
database:
dnssec  :
lame-servers:发现错误授权信息。
network :网络操作信息。
notify  :主从DNS服务器区数据文件传输通知信息。
queries :查询信息。
resolver:名称解析信息，包括来自解析器（客户端）的递归查询处理信息。
security:认可或非认可的请求信息。
update  :动态更新事件信息。
xfer-in :远程DNS服务器到本地DNS服务器的区域传送信息。
xfer-out:从本地DNS服务器到远程DNS服务器的区域传送信息。
```

**(3).默认的channel和类别**

当没有在named.conf中定义任何loggin字句时，发现在/var/log/messages中还是记录了很多DNS相关的日志，为什么没定义还是有记录呢？因为有BIND默认定义的日志记录方式。

在BIND中默认定义了4个channel，分别使用4种通道路径。这些channel你无法重定义它们，即使你不想要它们、不书写它们，BIND还是会自己创建它们。只有一种方法：新添加通道并指定使用该通道的类别，使你想记录的日志不使用默认的定义。

以下是默认定义的通道：

```
channel default_syslog {
    syslog daemon;
    severity info;
};
channel default_debug {
    file "named.run";
    severity dynamic;
};
channel default_stderr {
    stderr;
severity info;
};
channel null {
    null;
};
```

而同时在BIND 9 中所指定的默认类别语句如下:

```
category default { default_syslog;default_debug; };
```

也就是说，对于你没有指定的绝大多数类别的信息都使用通道default_syslog和通道default_debug来记录。由于默认定义了null通道，所以你想把default匹配的类别信息全部丢弃的话，可以直接使用`categroy default { null; };`来丢弃。
