---
title: rsync系列(2)：intofy+rsync和sersync的用法
p: linux/rsync_inotify_sersync.md
date: 2023-07-16 18:20:40
tags: Linux
categories: Linux
---

--------

**[回到Linux基础(rsync)系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# rsync系列(2): intofy+rsync和sersync的用法

## rsync+inotify

如果要实现定时同步数据，可以在客户端将rsync加入定时任务，但是定时任务的同步时间粒度并不能达到实时同步的要求。在Linux kernel 2.6.13后提供了inotify文件系统监控机制。通过rsync+inotify组合可以实现实时同步。

inotify实现工具有几款：inotify本身、sersync、lsyncd。其中sersync克服了inotify的缺陷，且提供了几个插件作为可选工具。此处先介绍inotify的用法以及它的缺陷，通过其缺陷引出sersync，并介绍其用法。

### 安装inotify工具inotify-tools

inotify由inotify-tools包提供。在安装inotify-tools之前，请确保内核版本高于2.6.13，且在`/proc/sys/fs/inotify`目录下有以下三项，这表示系统支持inotify监控，关于这3项的意义，下文会简单解释。
```bash
[root@node1 tmp]# ll /proc/sys/fs/inotify/
total 0
-rw-r--r-- 1 root root 0 Feb 11 19:57 max_queued_events
-rw-r--r-- 1 root root 0 Feb 11 19:57 max_user_instances
-rw-r--r-- 1 root root 0 Feb 11 19:57 max_user_watches
```
epel源上提供了inotify-tools工具，或者下载源码包格式进行编译。

inotify-tools源码包地址：<https://cloud.github.com/downloads/rvoicilas/inotify-tools/inotify-tools-3.14.tar.gz>

以下为编译安装过程：
```bash
tar xf inotify-tools-3.14.tar.gz
./configure --prefix=/usr/local/inotify-tools-3.14
make && make install
ln -s /usr/local/inotify-tools-3.14 /usr/local/inotify
```

inotify-tools工具只提供了两个命令。 
```bash
[root@xuexi ~]# rpm -ql inotify-tools | grep bin/
/usr/bin/inotifywait
/usr/bin/inotifywatch
```
其中inotifywait命令用于等待文件发生变化，所以可以可以实现监控(watch)的功能，该命令是inotify的核心命令。inotifywatch用于收集文件系统的统计数据，例如发生了多少次inotify事件，某文件被访问了多少次等等，一般用不上。

以下是inotify相关的内核参数。

- (1).`/proc/sys/fs/inotify/max_queued_events`：调用inotify_init时分配到inotify instance中可排队的event数的最大值，超出值时的事件被丢弃，但会触发队列溢出Q_OVERFLOW事件。
- (2).`/proc/sys/fs/inotify/max_user_instances`：每一个real user可创建的inotify instances数量的上限。
- (3).`/proc/sys/fs/inotify/max_user_watches`：每个inotify实例相关联的watches的上限，即每个inotify实例可监控的最大目录、文件数量。如果监控的文件数目巨大，需要根据情况适当增加此值。如：

```bash
[root@xuexi ~]# echo 30000000 > /proc/sys/fs/inotify/max_user_watches
```

<a name="inotifywait"></a>
### inotifywait命令以及事件分析

`inotifywait`命令的选项：
```
-m：表示始终监控，否则应该是监控到了一次就退出监控了
-r：递归监控，监控目录中的任何文件，包括子目录。递归监控可能会超出max_user_watches的值，需要适当调整该值
@<file>：如果是对目录进行递归监控，则该选项用于排除递归目录中不被监控的文件。file是相对路径还是绝对路径由监控目录是相对还是绝对来决定
-q：--quiet的意思，静默监控，这样就不会输出一些无关的信息
-e：指定监控的事件。一般监控的就delete、create、attrib、modify、close_write
--exclude <pattern>：通过模式匹配来指定不被监控的文件，区分大小写
--excludei <pattern>：通过模式匹配来指定不被监控的文件，不区分大小写
--timefmt：监控到事件触发后，输出的时间格式，可指定可不指定该选项，一般设置为[--timefmt '%Y/%m/%d %H:%M:%S']
--format：用户自定义的输出格式，如[--format '%w%f %e%T']
  %w：产生件的监控路径，不一定就是发生事件的具体文件，例如递归监控一个目录，该目录下的某文件产生事件，将输出该目录而非其内具体的文件
  %f：如果监控的是一个目录，则输出产生事件的具体文件名。其他所有情况都输出空字符串
  %e：产生的事件名称
  %T：以"--timefmt"定义的时间格式输出当前时间，要求同时定义"--timefmt"
```

`inotifywait -e`可监控的事件：
```
access：文件被访问
modify：文件被写入
attrib：元数据被修改。包括权限、时间戳、扩展属性等等
close_write：打开的文件被关闭，是为了写文件而打开文件，之后被关闭的事件
close_nowrite：read only模式下文件被关闭，即只能是为了读取而打开文件，读取结束后关闭文件的事件
close：是close_write和close_nowrite的结合，无论是何种方式打开文件，只要关闭都属于该事件
open：文件被打开
moved_to：向监控目录下移入了文件或目录，也可以是监控目录内部的移动
moved_from：将监控目录下文件或目录移动到其他地方，也可以是在监控目录内部的移动
move：是moved_to和moved_from的结合
moved_self：被监控的文件或目录发生了移动，移动结束后将不再监控此文件或目录
create：在被监控的目录中创建了文件或目录
delete：删除了被监控目录中的某文件或目录
delete_self：被监控的文件或目录被删除，删除之后不再监控此文件或目录
umount：挂载在被监控目录上的文件系统被umount，umount后不再监控此目录
isdir ：监控目录相关操作
```
以下是几个示例：
```bash
[root@xuexi ~]# mkdir /longshuai
# 以前台方式监控目录，由于没指定监控的事件，所以监控所有事件
[root@xuexi ~]# inotifywait -m /longshuai   
Setting up watches.
Watches established.
```
打开其他会话，对被监控目录进行一些操作，查看各操作会触发什么事件。
```bash
# 进入目录不触发任何事件
[root@xuexi ~]# cd  /longshuai
```
**(1).向目录中创建文件，触发create、open attrib、close_write和close事件。**
```bash
[root@xuexi longshuai]# touch a.log
/longshuai/ CREATE a.log
/longshuai/ OPEN a.log
/longshuai/ ATTRIB a.log
/longshuai/ CLOSE_WRITE,CLOSE a.log
```
如果是创建目录，则触发的事件则少的多。
```bash
[root@xuexi longshuai]# mkdir b
/longshuai/ CREATE,ISDIR b
```
ISDIR表示产生该事件的对象是一个目录。

**(2).修改文件属性，触发attrib事件。**
```bash
[root@xuexi longshuai]# chown 666 a.log
/longshuai/ ATTRIB a.log
```
**(3).cat查看文件，触发open、access、close_nowrite和close事件。**
```bash
[root@xuexi longshuai]# cat a.log
/longshuai/ OPEN a.log
/longshuai/ ACCESS a.log
/longshuai/ CLOSE_NOWRITE,CLOSE a.log
```
**(4).向文件中追加或写入或清除数据，触发open、modify、close_write和close事件。**
```bash
[root@xuexi longshuai]# echo "haha" >> a.log
/longshuai/ OPEN a.log
/longshuai/ MODIFY a.log
/longshuai/ CLOSE_WRITE,CLOSE a.log
```
**(5).vim打开文件并修改文件，中间涉及到临时文件，所以有非常多的事件。**
```bash
[root@xuexi longshuai]# vim a.log
/longshuai/ OPEN,ISDIR 
/longshuai/ CLOSE_NOWRITE,CLOSE,ISDIR 
/longshuai/ OPEN,ISDIR 
/longshuai/ CLOSE_NOWRITE,CLOSE,ISDIR 
/longshuai/ OPEN a.log
/longshuai/ CREATE .a.log.swp
/longshuai/ OPEN .a.log.swp
/longshuai/ CREATE .a.log.swx
/longshuai/ OPEN .a.log.swx
/longshuai/ CLOSE_WRITE,CLOSE .a.log.swx
/longshuai/ DELETE .a.log.swx
/longshuai/ CLOSE_WRITE,CLOSE .a.log.swp
/longshuai/ DELETE .a.log.swp
/longshuai/ CREATE .a.log.swp
/longshuai/ OPEN .a.log.swp
/longshuai/ MODIFY .a.log.swp
/longshuai/ ATTRIB .a.log.swp
/longshuai/ CLOSE_NOWRITE,CLOSE a.log
/longshuai/ OPEN a.log
/longshuai/ CLOSE_NOWRITE,CLOSE a.log
/longshuai/ MODIFY .a.log.swp
/longshuai/ CREATE 4913
/longshuai/ OPEN 4913
/longshuai/ ATTRIB 4913
/longshuai/ CLOSE_WRITE,CLOSE 4913
/longshuai/ DELETE 4913
/longshuai/ MOVED_FROM a.log
/longshuai/ MOVED_TO a.log~
/longshuai/ CREATE a.log
/longshuai/ OPEN a.log
/longshuai/ MODIFY a.log
/longshuai/ CLOSE_WRITE,CLOSE a.log
/longshuai/ ATTRIB a.log
/longshuai/ ATTRIB a.log
/longshuai/ MODIFY .a.log.swp
/longshuai/ DELETE a.log~
/longshuai/ CLOSE_WRITE,CLOSE .a.log.swp
/longshuai/ DELETE .a.log.swp 
```
其中有"ISDIR"标识的是目录事件。此外，需要注意到vim过程中，相应的几个临时文件(`.swp`、`.swx`和以`~`为后缀的备份文件)也产生了事件，这些临时文件的相关事件在实际应用过程中，其实不应该被监控。

**(6).向目录中拷入一个文件，触发create、open、modify和close_write、close事件。其实和新建文件基本类似。**
```bash
[root@xuexi longshuai]# cp /bin/find .
/longshuai/ CREATE find
/longshuai/ OPEN find
/longshuai/ MODIFY find
/longshuai/ MODIFY find
/longshuai/ CLOSE_WRITE,CLOSE find
```
**(7).向目录中移入和移除一个文件。**
```bash
[root@xuexi longshuai]# mv /tmp/after.log /longshuai
/longshuai/ MOVED_TO after.log
[root@xuexi longshuai]# mv /longshuai/after.log /tmp
/longshuai/ MOVED_FROM after.log
```
**(8).删除一个文件，触发delete事件。**
```bash
[root@xuexi longshuai]# rm -f a.log
/longshuai/ DELETE a.log
```

从上面的测试结果中可以发现，很多动作都涉及了close事件，且大多数情况都是伴随着`close_write`事件的。所以，大多数情况下在定义监控事件时，其实并不真的需要监控open、modify、close事件。特别是close，只需监控它的分支事件`close_write`和`close_nowrite`即可。由于一般情况下inotify都是为了监控文件的增删改，不会监控它的访问，所以一般只需监控`close_write`即可。

由于很多时候定义触发事件后的操作都是根据文件来判断的，例如a文件被监控到了变化(不管是什么变化)，就立即执行操作A，又由于对文件的一个操作行为往往会触发多个事件，例如cat查看文件就触发了open、access、close_nowrite和close事件，这样很可能会因为多个事件被触发而重复执行操作A。例如下面的示例，当监控到了/var/log/messages文件中出现了a.log关键字，就执行echo动作。
```bash
while inotifywait -mrq -e modify /var/log/messages; do
  if tail -n1 /var/log/messages | grep a.log; then
​    echo "haha"
  fi
done
```
综合以上考虑，建议对监控对象的close_write、moved_to、moved_from、delete和isdir(主要是create,isdir，但无法定义这两个事件的整体，所以仅监控isdir)事件定义对应的操作，因为它们互不重复。如有需要，可以将它们分开定义，再添加需要监控的其他事件。例如：
```bash
[root@xuexi tmp]# cat a.sh
#!/bin/bash
#
inotifywait -mrq -e delete,close_write,moved_to,moved_from,isdir /longshuai |\
while read line;do
   if echo $line | grep -i delete &>/dev/null; then
​       echo "At `date +"%F %T"`: $line" >>/etc/delete.log
   else
​       rsync -az $line --password-file=/etc/rsync_back.passwd rsync://rsync_backup@172.16.10.6::longshuai 
   fi
done
```

### inotify应该装在哪里

inotify是监控工具，监控目录或文件的变化，然后触发一系列的操作。

假如有一台站点发布服务器A，还有3台web服务器B/C/D，目的是让服务器A上存放站点的目录中有文件变化时，自动触发同步将它们推到web服务器上，这样能够让web服务器最快的获取到最新的文件。需要搞清楚的是，监控的是A上的目录，推送到的是B/C/D服务器，所以在站点发布服务器A上装好inotify工具。除此之外，一般还在web服务器BCD上将rsync配置为daemon运行模式，让其在873端口上处于监听状态。也就是说，对于rsync来说，监控端是rsync的客户端，其他的是rsync的服务端。

当然，这只是最可能的使用情况，并非一定需要如此。况且，inotify是独立的工具，它和rsync无关，它只是为rsync提供一种比较好的实时同步方式而已。

### inotify+rsync示例脚本

以下是监控/www目录的一个`inotify+rsync`脚本示例，也是网上流传的用法版本。但注意，该脚本非常烂，实际使用时需要做一些修改，此处仅仅只是示例(如果你不考虑资源消耗，那就无所谓)。
```bash
[root@xuexi www]# cat ~/inotify.sh
#!/bin/bash
 
watch_dir=/www
push_to=172.16.10.5
inotifywait -mrq -e delete,close_write,moved_to,moved_from,isdir \
            --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%e:%T' $watch_dir \
            --exclude=".*.swp" \
| while read line;do
    # logging some files which has been deleted and moved out
​    if echo $line | grep -i -E "delete|moved_from" &>/dev/null;then
​        echo "$line" >> /etc/inotify_away.log
​    fi
   # from here, start rsync's function
​    rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
​    if [ $? -eq 0 ];then
​        echo "sent $watch_dir success"
​    else
​        echo "sent $watch_dir failed"
​    fi
done
```

然后对上面的脚本赋予执行权限并执行。注意，该脚本是用来前台测试运行的，如果要后台运行，则将最后一段的if字句删掉。

该脚本记录了哪些被删除或从监控目录中移出的文件，且监控到事件后，触发的rsync操作是对整个监控目录`$watch_dir`进行同步，并且不对vim产生的临时文件进行同步。

前台运行该监控脚本。
```bash
[root@xuexi www]# ~/inotify.sh
```
然后测试分别向监控目录`/www`下做拷贝文件、删除文件等操作。

例如删除文件，会先记录删除事件到`/etc/inotify_away.log`文件中，再执行rsync同步删除远端对应的文件。
```bash
/www/yum.repos.d/base.repo:DELETE:2017-07-21 14:47:46
sent /www success
/www/yum.repos.d:DELETE,ISDIR:2017-07-21 14:47:46
sent /www success
```
再例如，拷入目录`/etc/pki`到`/www`下，会产生多个rsync结果。
```
sent /www success
sent /www success
sent /www success
sent /www success
sent /www success
sent /www success
......
```
显然，由于拷入了多个文件，rsync被触发了多次，但其实rsync只要同步一次`/www`目录到远端就足够了，多余的rsync操作完全是浪费资源。如果拷入少量文件，其实无所谓，但如果拷入成千上万个文件，将长时间调用rsync。例如：
```bash
[root@xuexi www]# cp -a /usr/share/man /www
```
拷入了15000多个文件到www目录下，该脚本将循环15000多次，且将调用15000多次rsync。虽然经过一次目录同步之后，rsync的性能消耗非常低(即使性能浪费少，但仍然是浪费)，但它消耗大量时间时间资源以及网络带宽。

### inotify的不足之处

虽然inotify已经整合到了内核中，在应用层面上也常拿来辅助rsync实现实时同步功能，但是inotify因其设计太过细致从而使得它配合rsync并不完美，所以需要尽可能地改进inotify+rsync脚本或者使用sersync工具。另外，inotify存在bug。

#### inotify的bug

当向监控目录下拷贝复杂层次目录(多层次目录中包含文件)，或者向其中拷贝大量文件时，**inotify经常会随机性地遗漏某些文件**。这些遗漏掉的文件由于未被监控到，所有监控的后续操作都不会执行，例如不会被rsync同步。

实际上，上面描述的问题不是inotify的缺陷，而是inotify-tools包中inotifywait工具的缺陷。inotifywait的man文档中也给出了这个bug说明。
```
BUGS
​    There are race conditions in the recursive directory watching code which can cause events to be missed if they occur in a directory immediately after that directory is created.  This is probably not fixable.
```
也就是说，那些直接发起inotify相关系统调用的上层工具(如sersync、lsyncd等)可能不会出现这个bug。

为了说明这个bug的影响，以下给出一些示例来证明。

以下是一个监控delete和close_write事件的示例，监控的是`/www`目录，该目录下初始时没有pki目录。
```bash
[root@xuexi ~]# inotifywait -mrq -e delete,close_write --format '%w%f:%e' /www
```
监控开始后，准备向该目录下拷贝`/etc/pki`目录，该目录有多个子目录，且有多个层次的子目录，有一些文件分布在各个子目录下。经过汇总，`/etc/pki`目录下共有30个普通文件。
```bash
[root@xuexi www]# find /etc/pki/ -type f | wc -l
30
```
另开一个终端，拷贝pki目录到`/www`下。
```bash
[root@xuexi www]# cp -a /etc/pki /www
```
于此同时，在监控终端上将产生一些监控事件。
```
/www/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7:CLOSE_WRITE,CLOSE
/www/pki/rpm-gpg/RPM-GPG-KEY-CentOS-Debug-7:CLOSE_WRITE,CLOSE
/www/pki/rpm-gpg/RPM-GPG-KEY-CentOS-Testing-7:CLOSE_WRITE,CLOSE
/www/pki/tls/certs/Makefile:CLOSE_WRITE,CLOSE
/www/pki/tls/certs/make-dummy-cert:CLOSE_WRITE,CLOSE
/www/pki/tls/certs/renew-dummy-cert:CLOSE_WRITE,CLOSE
/www/pki/tls/misc/c_info:CLOSE_WRITE,CLOSE
/www/pki/tls/misc/c_issuer:CLOSE_WRITE,CLOSE
/www/pki/tls/misc/c_name:CLOSE_WRITE,CLOSE
/www/pki/tls/openssl.cnf:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/README:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/ca-legacy.conf:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/extracted/java/README:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/extracted/java/cacerts:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/extracted/pem/email-ca-bundle.pem:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/extracted/pem/objsign-ca-bundle.pem:CLOSE_WRITE,CLOSE
/www/pki/ca-trust/source/README:CLOSE_WRITE,CLOSE
/www/pki/nssdb/cert8.db:CLOSE_WRITE,CLOSE
/www/pki/nssdb/cert9.db:CLOSE_WRITE,CLOSE
/www/pki/nssdb/key3.db:CLOSE_WRITE,CLOSE
/www/pki/nssdb/key4.db:CLOSE_WRITE,CLOSE
/www/pki/nssdb/pkcs11.txt:CLOSE_WRITE,CLOSE
/www/pki/nssdb/secmod.db:CLOSE_WRITE,CLOSE
```
数一数上面监控到的事件结果，总共有25行，也就是25个文件的拷贝动作被监控到，但实际上拷贝的总文件数(目录和链接文件不纳入计算)却是30个。换句话说，inotify遗漏了5个文件。

经过测试，遗漏的数量和文件不是固定而是随机的(所以运气好可能不会有遗漏)，且只有close_write事件会被遗漏，delete是没有问题的。为了证实这个bug，再举两个示例。

向监控目录`/www`下拷贝`/usr/share/man`目录，该目录下有15441个普通文件，且最深有3层子目录，层次上并不算复杂。
```bash
[root@xuexi www]# find /usr/share/man/ -type f | wc -l
15441
```
为了方便计算监控到的事件数量，将事件结果重定向到文件man.log中去。
```bash
[root@xuexi ~]# inotifywait -mrq -e delete,close_write,moved_to,moved_from,isdir --format '%w%f:%e' /www > /tmp/man.log
```
开始拷贝。
```bash
[root@xuexi www]# cp -a /usr/share/man /www
```
拷贝结束后，统计/tmp/man.log文件中的行数，也即监控到的事件数。
```bash
[root@xuexi www]# cat /tmp/man.log | wc -l
15388
```

显然，监控到了15388个文件，比实际拷入的文件少了53个，这也证明了inotify的bug。

但上述两个示例监控的都是close_write事件，为了保证证明的严格性，监控所有事件，为了方便后续的统计，将监控结果重定向到pki.log文件中，且在监控开始之前，先删除/www/pki目录。
```bash
[root@xuexi ~]# rm -rf /www/pki
[root@xuexi ~]# inotifywait -mrq --format '%w%f:%e' /www > /tmp/pki.log
```
向监控目录`/www`下拷贝`/etc/pki`目录。
```bash
[root@xuexi ~]# cp -a /etc/pki /www
```
由于监控了所有事件，所以目录相关事件"ISDIR"以及软链接相关事件也都被重定向到`/tmp/pki.log`中，为了统计监控到的文件数量，先把pki.log中目录相关的"ISDIR"行去掉，再对同一个文件进行去重，以便一个文件的多个事件只被统计一次，以下是统计命令。
```bash
[root@xuexi www]# sed /ISDIR/d /tmp/pki.log | cut -d":" -f1 | sort -u | wc -l
32
```
结果竟然比普通文件数30多2个，实际并非如此，因为pki.log中软链接文件也被统计了，但`/www/pki`目录下普通文件加上软链接文件总共有35个文件。
```bash
[root@xuexi www]# find /www/pki -type f -o -type l | wc -l
35
```
也就是说，即使监控的是全部事件，也还是出现了遗漏，所以我认为这是inotify的一个bug。

但是需要说明的是，只有拷贝多层次包括多文件的目录时才会出现此bug，拷贝单个文件或简单无子目录的目录时不会出现此bug。对于`inotify+rsync`来说，由于触发事件后常使用rsync同步整个目录而非单个文件，所以这个bug对rsync来说并不算严重。

#### inotify+rsync的缺陷

**由于inotify的bug，使用inotify+rsync时应该总是让rsync同步目录，而不是同步那些产生事件的单个文件，否则很可能会出现文件遗漏。另一方面，同步单个文件的性能非常差。**下面相关缺陷的说明将默认rsync同步的是目录。

使用inotify+rsync时，考虑两方面问题：
- (1).由于inotify监控经常会对一个文件产生多个事件，且一次性操作同一个目录下多个文件也会产生多个事件，这使得inotify几乎总是多次触发rsync同步目录，由于rsync同步的是目录，所以多次触发rsync完全没必要，这会浪费资源和网络带宽；如果是分层次独立监控子目录，则会导致同步无法保证实时性
- (2).vim编辑文件的过程中会产生.swp和.swx等临时文件，inotify也会监控这些临时文件，且临时文件会涉及多个事件，因此它们可能也会被rsync拷贝走，除非设置好排除临时文件，但无论如何，这些临时文件是不应该被同步的，极端情况下，同步vim的临时文件到服务器上可能是致命的。

**由于这两个缺陷，使得通过脚本实现的inotify+rsync几乎很难达到完美，即使要达到不错的完美度，也不是件容易的事(不要天真的认为网上那些inotify+rsync示例是完美的)。**总之，为了让`inotify+rsync`即能保证同步性能，又能保证不同步临时文件，认真设计`inotify+rsync`的监控事件、循环以及rsync命令是很有必要的。

在设计`inotify+rsync`脚本过程中，有以下几个目标应该尽量纳入考虑或达到：

- (1).每个文件都尽量少地产生监控事件，但又不能遗漏事件。
- (2).让rsync同步目录，而不是同步产生事件的单个文件。
- (3).一次性操作同步目录下的多个文件会产生多个事件，导致多次触发rsync。如果能让这一批操作只触发一次rsync，则会大幅降低资源的消耗。
- (4).rsync同步目录时，考虑好是否要排除某些文件，是否要加上`--delete`选项等。
- (5).为了性能，可以考虑对子目录、对不同事件单独设计`inotify+rsync`脚本。

以前文给出的示例脚本来分析。
```bash
[root@xuexi www]# cat ~/inotify.sh
#!/bin/bash
 
watch_dir=/www
push_to=172.16.10.5
inotifywait -mrq -e delete,close_write,moved_to,moved_from,isdir \
            --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%e:%T' $watch_dir \
            --exclude=".*.swp" \
| while read line;do
    # logging some files which has been deleted and moved out
​    if echo $line | grep -i -E "delete|moved_from" &>/dev/null;then
​        echo "$line" >> /etc/inotify_away.log
​    fi
    # from here, start rsync's function
​    rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
​    if [ $? -eq 0 ];then
​        echo "sent $watch_dir success"
​    else
​        echo "sent $watch_dir failed"
​    fi
done  
```
该脚本中已经尽量少地设置监控事件，使得它尽量少重复触发rsync。但需要明确的是，尽管设计的目标是尽量少触发事件，但应该以满足需求为前提来定义监控事件。如果不清楚如何选择监控事件，回看前文[inotify命令以及事件分析](#inotifywait)。另外，可以考虑对文件、目录、子目录单独定义不同的脚本分别监控不同事件。

该脚本的不足之处主要在于重复触发rsync。该脚本中rsync同步的是目录而非单个文件，所以如果一次性操作了该目录中多个文件，将会产生多个事件，也因此会触发多次rsync命令，在前文中给出了一个拷贝`/usr/share/man`的示例，它调用了15000多次rsync，其实只需同步一次即可，剩余的上万次同步完全是多余的。

因此，上述脚本的改进方向是尽量少地调用rsync，但却要保证rsync的实时性和同步完整性。使用sersync工具可以很轻松地实现这一点，也许sersync的作者开发该工具的最初目标也是为了解决这个问题。关于sersync的用法，留在后文介绍。

<a name="inotify_rsync"></a>
### inotify+rsync的最佳实现

在上面已经提过inotify+rsync不足之处以及改进的目标。以下是通过修改shell脚本来改进`inotify+rsync`的示例。
```bash
[root@xuexi tmp]# cat ~/inotify.sh

#!/bin/bash
 
watch_dir=/www
push_to=172.16.10.5
 
# The first thing: initial sync
rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
 
inotifywait -mrq -e delete,close_write,moved_to,moved_from,isdir \
            --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%e:%T' $watch_dir \
            --exclude=".*.swp" >>/etc/inotifywait.log &
 
while true;do
​     if [ -s "/etc/inotifywait.log" ];then
​        grep -i -E "delete|moved_from" /etc/inotifywait.log >> /etc/inotify_away.log
​        rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
​        if [ $? -ne 0 ];then
​           echo "$watch_dir sync to $push_to failed at `date +"%F %T"`,please check it by manual" |\
​           mail -s "inotify+Rsync error has occurred" root@localhost
​        fi
​        cat /dev/null > /etc/inotifywait.log
​        rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
​    else
​        sleep 1
​    fi
done
```
为了让一次性对目录下多个文件的操作只触发一次rsync，通过`while read line`这种读取标准输入的循环方式是不可能实现的。

本人的实现方法是将inotifywait得到的事件记录到文件`/etc/inotifywait.log`中，然后在死循环中判断该文件，如果该文件不为空则调用一次rsync进行同步，同步完后立即清空`inotifywait.log`文件，防止重复调用rsync。但需要考虑一种情况，inotifywait可能会不断地向`inotifywait.log`中写入数据，清空该文件可能会使得在rsync同步过程中被inotifywait监控到的文件被rsync遗漏，所以在清空该文件后应该再调用一次rsync进行同步，这也变相地实现了失败重传的错误处理功能。如果没有监控到事件，`inotifywait.log`将是空文件，此时循环将睡眠1秒钟，所以该脚本并不是百分百的实时，但1秒钟的误差对于cpu消耗来说是很值得的。

该脚本对每批事件只调用两次rsync，虽然无法像sersync一样只触发一次rsync，但差距完全可以忽略不计。

另外，脚本中inotifywait命令中的后台符号`&`绝不能少，否则脚本将一直处于inotifywait命令阶段，不会进入到下一步的循环阶段。也不能写成如下格式：但需要注意，脚本(子shell)中的后台进程在脚本结束的时候不会随之停止，而是挂靠在`pid=1`的`init/systemd`进程下，这种情况下可以直接使用 killall script_file 的方式来停止脚本，这样脚本中的后台也会中断。如果想直接在脚本中实现这样的功能，见：[如何让shell脚本自杀](http://www.cnblogs.com/f-ck-need-u/p/8661501.html)。

实际上，上面的脚本还远不算完美，更完美的做法是提供一个判断功能，如果监控的目录非常大(即文件数量非常多)，应该传输变化的单个文件而不是同步整个目录，如果监控目录不算大，可以考虑同步整个目录。sersync采用的方式是监控目录，但取出发生变化的单个文件并同步，并提供定时任务来决定多久整体同步一次，也就是整个目录同步。这一切都能通过shell脚本来实现，包括它的多线程也一样，如果有兴趣，可以自己写一写。
```bash
inotifywait -mrq -e delete,close_write,moved_to,moved_from,isdir \
            --timefmt '%Y-%m-%d %H:%M:%S' --format '%w%f:%e:%T' $watch_dir \
            --exclude=".*.swp" \
| tee /etc/inotifywait.log \
| while read line;do       # 或者while true;do**
​     if [ -s "/etc/inotifywait.log" ];then
​        grep -i -E "delete|moved_from" /etc/inotifywait.log >> /etc/inotify_away.log
​        rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
​        if [ $? -ne 0 ];then
​           echo "$watch_dir sync to $push_to failed at `date +"%F %T"`,please check it by manual" |\
​           mail -s "inotify+Rsync error has occurred" root@localhost
​        fi
​        cat /dev/null > /etc/inotifywait.log
​        rsync -az --delete --exclude="*.swp" --exclude="*.swx" $watch_dir $push_to:/tmp
​    else
​        sleep 1
​    fi
done 
```
因为这样的格式在进入循环后将无法回到inotifywait命令。由此，可以知道while循环在读取具有随机性的标准输入时，应该让产生标准输入的命令在后台运行。

## sersync

sersync类似于inotify，同样用于监控，但它克服了inotify的几个缺点。

前文说过，inotify最大的不足是会产生重复事件，或者同一个目录下多个文件的操作会产生多个事件(例如，当监控目录中有5个文件时，删除目录时会产生6个监控事件)，从而导致重复调用rsync命令。而且vim文件时，inotify会监控到临时文件的事件，但这些事件相对于rsync来说是不应该被监控的。

在上文已经给出了inotify+rsync的最佳实现脚本，它克服了上面两个问题。sersync也能克服这样的问题，且实现方式更简单，另外它还有多线程的优点。

sersync项目地址：<https://code.google.com/archive/p/sersync/>，在此网站中有下载、安装、使用等等详细的中文介绍。

sersync下载地址：<https://code.google.com/archive/p/sersync/downloads>

sersync优点：
- 1.sersync是使用c++编写，而且对linux系统文件系统产生的临时文件和重复的文件操作进行过滤，所以在结合rsync同步的时候，节省了运行时耗和网络资源。因此更快。
- 2.sersync配置很简单，其中bin目录下已经有静态编译好的2进制文件，配合bin目录下的xml配置文件直接使用即可。
- 3.sersync使用多线程进行同步，尤其在同步较大文件时，能够保证多个服务器实时保持同步状态。 
- 4.sersync有出错处理机制，通过失败队列对出错的文件重新同步，如果仍旧失败，则按设定时长对同步失败的文件重新同步。
- 5.sersync自带crontab功能，只需在xml配置文件中开启，即可按要求隔一段时间整体同步一次。无需再额外配置crontab功能。 
- 6.sersync可以二次开发。

简言之，sersync可以过滤重复事件减轻负担、自带crontab功能、多线程调用rsync、失败重传。

建议：
- (1)当同步的目录数据量不大时，建议使用rsync+inotify
- (2)当同步的目录数据量很大时（几百G甚至1T以上）文件很多时，建议使用rsync+sersync

实际上，在前面[inotify+rsync的最佳实现](#inotify_rsync)中的改进脚本，除了多线程功能，已经和sersync的核心功能接近了，即使是同步大量数据，性能也能接近sersync。

sersync工具包无需任何安装，解压即可使用。
```bash
[root@xuexi ~]# wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/sersync/sersync2.5.4_64bit_binary_stable_final.tar.gz
[root@xuexi ~]# tar xf sersync2.5.4_64bit_binary_stable_final.tar.gz
[root@xuexi ~]# cp -a GNU-Linux-x86 /usr/local/sersync
[root@xuexi ~]# echo "PATH=$PATH:/usr/local/sersync" > /etc/profile.d/sersync.sh
[root@xuexi ~]# source /etc/profile.d/sersync.sh
```

sersync目录`/usr/local/sersync`只有两个文件：一个是二进制程序文件，一个是xml格式的配置文件。
```bash
[root@xuexi ~]# ls /usr/local/sersync/
confxml.xml  sersync2
```
其中confxml.xml是配置文件，文件内容很容易理解。以下是示例文件内容说明。
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<head version="2.5">
​    <host hostip="localhost" port="8008"></host>
​    <debug start="false"/>           # 是否开启调试模式，下面所有出现false和true的地方都分别表示关闭和开启的开关
​    <fileSystem xfs="false"/>        # 监控的是否是xfs文件系统
​    <filter start="false">           # 是否启用监控的筛选功能，筛选的文件将不被监控
​        <exclude expression="(.*)\.svn"></exclude>
​        <exclude expression="(.*)\.gz"></exclude>
​        <exclude expression="^info/*"></exclude>
​        <exclude expression="^static/*"></exclude>
​    </filter>
​    <inotify>   # 监控的事件，默认监控的是delete/close_write/moved_from/moved_to/create folder
​        <delete start="true"/>
​        <createFolder start="true"/>
​        <createFile start="false"/>
​        <closeWrite start="true"/>
​        <moveFrom start="true"/>
​        <moveTo start="true"/>
​        <attrib start="false"/>
​        <modify start="false"/>
​    </inotify>
 
​    <sersync>       # rsync命令的配置段
​        <localpath watch="/www">    # 同步的目录或文件，同inotify+rsync一样，建议同步目录
​            <remote ip="172.16.10.5" name="/tmp/www"/>  # 目标地址和rsync daemon的模块名，所以远端要以daemon模式先运行好rsync
​            <!--remote ip="IPADDR" name="module"-->     # 除非下面开启了ssh start，此时name为远程shell方式运行时的目标目录
​        </localpath>
​        <rsync>                      # 指定rsync选项
​            <commonParams params="-az"/>
​            <auth start="false" users="root" passwordfile="/etc/rsync.pas"/>
​            <userDefinedPort start="false" port="874"/><!-- port=874 -->
​            <timeout start="false" time="100"/><!-- timeout=100 -->
​            <ssh start="false"/>      # 是否使用远程shell模式而非rsync daemon运行rsync命令
​        </rsync>
​        <failLog path="/tmp/rsync_fail_log.sh" timeToExecute="60"/><!--default every 60mins execute once-->  # 错误重传
​        <crontab start="false" schedule="600"><!--600mins-->    # 是否开启crontab功能
​            <crontabfilter start="false">       # crontab定时传输的筛选功能
​                <exclude expression="*.php"></exclude>
​                <exclude expression="info/*"></exclude>
​            </crontabfilter>
​        </crontab>
​        <plugin start="false" name="command"/>
​    </sersync>
 
​    <plugin name="command">
​        <param prefix="/bin/sh" suffix="" ignoreError="true"/>  <!--prefix /opt/tongbu/mmm.sh suffix-->
​        <filter start="false">
​            <include expression="(.*)\.php"/>
​            <include expression="(.*)\.sh"/>
​        </filter>
​    </plugin>
 
​    <plugin name="socket">
​        <localpath watch="/opt/tongbu">
​            <deshost ip="192.168.138.20" port="8009"/>
​        </localpath>
​    </plugin>
​    <plugin name="refreshCDN">
​        <localpath watch="/data0/htdocs/cms.xoyo.com/site/">
​            <cdninfo domainname="ccms.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>
​            <sendurl base="http://pic.xoyo.com/cms"/>
​            <regexurl regex="false" match="cms.xoyo.com/site([/a-zA-Z0-9]*).xoyo.com/images"/>
​        </localpath>
​    </plugin>
</head>
```
以上配置文件采用的是远程shell方式的rsync连接方式，所以无需在目标主机上启动rsync daemon。设置好配置文件后，只需执行sersync2命令即可。该命令的用法如下：
```bash
[root@xuexi sersync]# sersync2 -h
set the system param
execute：echo 50000000 > /proc/sys/fs/inotify/max_user_watches
execute：echo 327679 > /proc/sys/fs/inotify/max_queued_events
parse the command param
_______________________________________________________
参数-d:启用守护进程模式，让sersync2运行在后台
参数-r:在监控前，将监控目录与远程主机用rsync命令推送一遍，
       :即首先让远端目录和本地一致，以后再同步则通过监控实现增量同步
参数-n:指定开启守护线程的数量，默认为10个
参数-o:指定配置文件，默认使用confxml.xml文件
参数-m:单独启用其他模块，使用 -m refreshCDN 开启刷新CDN模块
参数-m:单独启用其他模块，使用 -m socket 开启socket模块
参数-m:单独启用其他模块，使用 -m http 开启http模块
不加-m参数，则默认执行同步程序
________________________________________________________________
```
由此可见，sersync2命令总是会先设置inotify相关的系统内核参数。

所以，只需执行以下简单的命令即可。
```bash
[root@xuexi ~]# sersync2 -r -d
set the system param
execute：echo 50000000 > /proc/sys/fs/inotify/max_user_watches
execute：echo 327679 > /proc/sys/fs/inotify/max_queued_events
parse the command param
option: -r      rsync all the local files to the remote servers before the sersync work
option: -d      run as a daemon
daemon thread num: 10
parse xml config file
host ip : localhost     host port: 8008
daemon start，sersync run behind the console 
config xml parse success
please set /etc/rsyncd.conf max connections=0 Manually
sersync working thread 12  = 1(primary thread) + 1(fail retry thread) + 10(daemon sub threads) 
Max threads numbers is: 22 = 12(Thread pool nums) + 10(Sub threads)
please according your cpu ，use -n param to adjust the cpu rate
------------------------------------------
rsync the directory recursivly to the remote servers once
working please wait...
execute command: cd /www && rsync -az -R --delete ./  -e ssh 172.16.10.5:/tmp/www >/dev/null 2>&1
run the sersync: 
watch path is: /www
```
上面粗体加红标注的即为rsync命令的运行参数，由于rsync执行前会cd到监控目录中，且rsync通过`-R`选项是以`/www`为根的相对路径进行同步的，所以监控目录自身不会被拷贝到远端，因此在confxml.xml中设置目标目录为`/tmp/www`，这样本地`/www`下的文件将同步到目标主机的`/tmp/www`目录下，否则将会同步到目标主机的/tmp目录下。

对于sersync多实例，也即监控多个目录时，只需分别配置不同配置文件，然后使用sersync2指定对应配置文件运行即可。

例如：
```bash
[root@xuexi ~]# sersync2 -r -d -o /etc/sersync.d/nginx.xml
```

## 总结说明

rsync功能非常强大，基本用法比较简单，配合inotify或者使用sersync也能较好地实现实时同步。但要想实现复杂的rsync功能，或者比较好地配合inotify却不是那么简单的事。复杂的rsync功能由非常多的可选选项来实现，但要读懂man rsync中大多数选项说明，必要的前提是要搞懂rsync算法原理和实现机制。对于inotify来说，设计一个较好的脚本需要考虑的事情可能也比较多，其中最重要的一点是尽量少地监控事件和尽量少地重复触发rsync。其实，如非特殊需求(如需要执行特定的rsync命令，而sersync中给出的选项无法满足)，sersync能很完美地替代inotify+rsync，不仅配置简单，且效率高，实时性好。
