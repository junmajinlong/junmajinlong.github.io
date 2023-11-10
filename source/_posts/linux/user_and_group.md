---
title: 用户和组管理
p: linux/user_and_group.md
date: 2019-07-06 18:20:42
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 用户和组的基本概念

用户和组是操作系统中一种身份认证资源。

每个用户都有用户名、用户的唯一编号uid(user id)、所属组及其默认的shell，可能还有密码、家目录、附属组、注释信息等。

每个组也有自己的名称、组唯一编号gid(group id)。一般来说，gid和uid是可以不相同的，但绝大多数都会让它们保持一致，大致属于约定俗成类的概念吧。

组分为主组(primary group)和辅助组(secondary group)两种，用户一定会属于某个主组，也可以同时加入多个辅助组。

在Linux中，用户分为3类：

(1). **超级管理员**

超级管理员是最高权限者，它的uid=0，默认超级管理员用户名为root。因为uid默认具有唯一性，所以超级管理员默认只能有一个(如何添加额外的超级管理员，见useradd命令)，但这一个超级管理员的名称并非一定要是root。但是没人会去改root的名称，在后续非常非常多的程序中，都认为超级管理员名称为root，这里要是一改，牵一发而动全身。

(2). **系统用户**

有时候需要一类具有某些特权但又不需要登录操作系统的用户，这类用户称为系统用户。它们的uid范围从201到999(不包括1000)，有些老版本范围是1到499(centos 6)，出于安全考虑，它们一般不用来登录，所以它们的shell一般是/sbin/nologin，而且大多数时候它们是没有家目录的。

(3). **普通用户**

普通用户是权限受到限制的用户，默认只能执行/bin、/usr/bin、/usr/local/bin和自身家目录下的命令。它们的uid从500开始。尽管普通用户权限收到限制，但是它对自身家目录下的文件是有所有权限的。

超级管理员和其他类型的用户，它们的命令提示符是不一样的。uid=0的超级管理员，命令提示符是"#"，其他的为"$"。

![](/img/linux/733013-20170614225856884-1602298766.jpg)

默认root用户的家目录为/root，其他用户的家目录一般在/home下以用户名命名的目录中，如longshuai这个用户的家目录为/home/longshuai。当然，家目录是可以自定义位置和名称的。

![](/img/referer.jpg)

# 用户和组管理相关的文件

## 用户文件/etc/passwd

/etc/passwd文件里记录的是**操作系统中用户**的信息，这里面记录了几行就表示系统中有几个系统用户。它的格式大致如下：

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
mysql:x:27:27:MySQL Server:/var/lib/mysql:/bin/bash
nginx:x:498:499:Nginx web server:/var/lib/nginx:/sbin/nologin
longshuai:x:1000:1000::/home/longshuai:/bin/bash
```

每一行表示一个用户，每一行的格式都是6个冒号共7列属性，其中有很多用户的某些列属性是留空的。

```
用户名:x:uid:gid:用户注释信息:家目录:使用的shell类型
```

- 第一列：用户名。注意两个个特殊的用户名，root、nobody
- 第二列：x。在以前老版本的系统上，第二列是存放用户密码的，但是密码和用户信息放在一起不便于管理(密钥要保证其特殊属性)，所以后来将密码单独放在另一个文件/etc/shadow中，这里就都写成x了
- 第三列：uid
- 第四列：gid
- 第五列：用户注释信息。
- 第六列：用户家目录。注意root用户的家目录为/root
- 第七列：用户的默认shell，虽然叫shell，但其实可以是任意一个可执行程序或脚本。例如上面的/bin/bash、/sbin/nologin、/sbin/shutdown

用户的默认shell表示的是用户登录(如果允许登录)时的环境或执行的命令。例如shell为/bin/bash时，表示登录时就执行/bin/bash命令进入bash环境；shell为/sbin/nologin表示该用户不能登录，之所以不能登录不是因为指定了这个特殊的程序，而是由/sbin/nologin这个程序的功能实现的，假如修改Linux的源代码，将/sbin/nologin这个程序变成可登录，那么shell为/sbin/nologin时也是可以登录的。

## 密码文件/etc/shadow

/etc/shadow文件中存放的是用户的密码信息。该文件具有特殊属性，除了超级管理员，任何人都不能直接读取和修改该文件，而用户自身之所以能修改密码，则是因为passwd程序的suid属性，使得修改密码时临时提升为root权限。

该文件的格式大致如下：

```
root:$6$hS4yqJu7WQfGlk0M$Xj/SCS5z4BWSZKN0raNncu6VMuWdUVbDScMYxOgB7mXUj./dXJN0zADAXQUMg0CuWVRyZUu6npPLWoyv8eXPA.::0:99999:7::: ftp:*:16659:0:99999:7::: nobody:*:16659:0:99999:7::: 
longshuai:$6$8LGe6Eh6$vox9.OF3J9nD0KtOYj2hE9DjfU3iRN.v3up4PbKKGWLOy3k1Up50bbo7Xii/Uti05hlqhktAf/dZFy2RrGp5W/:17323:0:99999:7:::
```

每一行表示一个用户密码的属性，有8个冒号共9列属性。该文件更详细的信息看wiki：<https://en.wikipedia.org/wiki/Passwd#Shadow_file>。

- 第一列：用户名。
- 第二列：加密后的密码。但是这一列是有玄机的，有些特殊的字符表示特殊的意义。
  - 1.该列留空，即`::`，表示该用户没有密码。
  - 2.该列为`!`，即`:!:`，表示该用户被锁，被锁将无法登陆，但是可能其他的登录方式是不受限制的，如ssh key的方式，su的方式。
  - 3.该列为`*`，即`:*:`，也表示该用户被锁，和`!`效果是一样的。
  - 4.该列以`!`或`!!`开头，则也表示该用户被锁。
  - 5.该列为`!!`，即`:!!:`，表示该用户从来没设置过密码。
  - 6.如果格式为`$id$salt$hashed`，则表示该用户密码正常。其中`$id$`的id表示密码的加密算法，`$1$`表示使用MD5算法，`$2a$`表示使用Blowfish算法，`$2y$`是另一算法长度的Blowfish,`$5$`表示SHA-256算法，而`$6$`表示SHA-512算法，可见上面的结果中都是使用sha-512算法的。`$5$`和`$6$`这两种算法的破解难度远高于MD5。`$salt$`是加密时使用的salt，`$hashed`才是真正的密码部分。
- 第三列：从1970年1月1日到上次密码修改经过的时间(天数)。通过计算现在离1970年1月1日的天数减去这个值，结果就是上次修改密码到现在已经经过了多少天，即现在的密码已经使用了多少天。
- 第四列：密码最少使用期限(天数)。省略或者0表示不设置期限。例如，刚修改完密码又想修改，可以限制多久才能再次修改
- 第五列：密码最大使用期限(天数)。超过了它不一定密码就失效，可能下一个字段设置了过期后的宽限天数。设置为空时将永不过期，后面设置的提醒和警告将失效。root等一些用户的已经默认设置为了99999，表示永不过期。如果值设置小于最短使用期限，用户将不能修改密码。
- 第六列：密码过期前多少天就开始提醒用户密码将要过期。空或0将不提醒。
- 第七列：密码过期后宽限的天数，在宽限时间内用户无法使用原密码登录，必须改密码或者联系管理员。设置为空表示没有强制的宽限时间，可以过期后的任意时间内修改密码。
- 第八列：帐号过期时间。从1970年1月1日开始计算天数。设置为空帐号将永不过期，不能设置为0。不同于密码过期，密码过期后账户还有效，改密码后还能登录；帐号过期后帐号失效，修改密码重设密码都无法使用该帐号。
- 第九列：保留字段。

## 组文件/etc/group和/etc/gshadow

大致知道有这么两个文件即可，至于文件中的内容无需关注。

/etc/group包含了组信息。每行一个组，每一行3个冒号共4列属性。
```
root:x:0: longshuai:x:500: xiaofang:x:501:zhangsan,lisi
```
- 第一列：组名。
- 第二列：占位符。
- 第三列：gid。
- 第四列：该组下的user列表，这些user成员以该组做为辅助组，多个成员使用逗号隔开。

/etc/gshadow包含了组密码信息

## 骨架目录/etc/skel

骨架目录中的文件是每次新建用户时，都会复制到新用户家目录里的文件。默认只有3个环境配置文件，可以修改这里面的内容，或者添加几个文件在骨架目录中，以后新建用户时就会自动获取到这些环境和文件。

```shell
$ ls –l -A /etc/skel
total 12
-rw-r--r--. 1 root root 18 Oct 16 2014 .bash_logout
-rw-r--r--. 1 root root 176 Oct 16 2014 .bash_profile
-rw-r--r--. 1 root root 124 Oct 16 2014 .bashrc
```

删除家目录下这些文件，会导致某些设置出现问题。例如删除".bashrc"这个文件，会导致提示符变异的问题，如下右图。

![](/img/linux/733013-20170614225857884-877337404.jpg)

要解决这个问题，只需拷贝一个正常的.bashrc文件到其家目录中即可。一般还会修改该文件的所有者和权限。

![](/img/referer.jpg)

## /etc/login.defs

设置用户帐号限制的文件。该文件里的配置对root用户无效。

如果/etc/shadow文件里有相同的选项，则以/etc/shadow里的设置为准，也就是说/etc/shadow的配置优先级高于/etc/login.defs。

该文件有很多配置项，文件的默认内容只给出了一小部分，若想知道全部的配置项以及配个配置项的详细说明，可以"man 5 login.defs"查看。

```
[root@xuexi ~]# less /etc/login.defs
#QMAIL_DIR      Maildir          # QMAIL_DIR是Qmail邮件的目录，所以可以不设置它
MAIL_DIR        /var/spool/mail  # 默认邮件根目录，即信箱
#MAIL_FILE      .mail            # mail文件的格式是.mail

# Password aging controls:
PASS_MAX_DAYS   99999         # 密码最大有效期(天)
PASS_MIN_DAYS   0             # 两次密码修改之间最小时间间隔
PASS_MIN_LEN    5             # 密码最短长度
PASS_WARN_AGE   7             # 密码过期前给警告信息的时间

# 控制useradd创建用户时自动选择的uid范围
# Min/max values for automatic uid selection in useradd
UID_MIN                  1000
UID_MAX                 60000
# System accounts
SYS_UID_MIN               201
SYS_UID_MAX               999

# 控制groupadd创建组时自动选择的gid范围
# Min/max values for automatic gid selection in groupadd
GID_MIN                  1000
GID_MAX                 60000
# System accounts
SYS_GID_MIN               201
SYS_GID_MAX               999

# 设置此项后，在删除用户时，将自动删除用户拥有的at/cron/print等job
#USERDEL_CMD    /usr/sbin/userdel_local

# 控制useradd添加用户时是否默认创建家目录，useradd -m选项会覆盖此处设置
CREATE_HOME     yes

# 设置创建家目录时的umask值，若不指定则默认为022
UMASK           077

# 设置此项表示当组中没有成员时自动删除该组
# 且useradd是否同时创建同用户名的主组。
# (该文件中并没有此项说明，来自于man useradd中-g选项的说明)
USERGROUPS_ENAB yes

# 设置用户和组密码的加密算法
ENCRYPT_METHOD SHA512
```

注意，/etc/login.defs中的设置控制的是shadow-utils包中的组件，也就是说，该组件中的工具执行操作时会读取该文件中的配置。该组件中包含下面的程序：

```
/usr/bin/gpasswd      ：administer /etc/group and /etc/gshadow
/usr/bin/newgrp       ：log in to a new group，可用来修改gid，哪怕是正在登陆的会话也可以修改
/usr/bin/sg           ：execute command as different group ID
/usr/sbin/groupadd    ：添加组
/usr/sbin/groupdel    ：删除组
/usr/sbin/groupmems   ：管理当前用户的主组中的成员，root用户则可以指定要管理的组
/usr/sbin/groupmod    ：modify a group definition on the system
/usr/sbin/grpck       ：verify integrity of group files
/usr/sbin/grpconv     ：无视它
/usr/sbin/grpunconv   ：无视它
/usr/sbin/pwconv      ：无视它
/usr/sbin/pwunconv    ：无视它
/usr/sbin/adduser     ：是useradd的一个软链接，添加用户
/usr/sbin/chpasswd    ：update passwords in batch mode
/usr/sbin/newusers    ：update and create new users in batch
/usr/sbin/pwck        ：verify integrity of passsword files
/usr/sbin/useradd     ：添加用户
/usr/sbin/userdel     ：删除用户
/usr/sbin/usermod     ：重定义用户信息
/usr/sbin/vigr        ：edit the group and shadow-group file
/usr/sbin/vipw        ：edit the password and shadow-password file
/usr/bin/lastlog      ：输出所有用户或给定用户最近登录信息
```

## /etc/default/useradd

创建用户时的默认配置。useradd -D修改的就是此文件。

```
[root@xuexi ~]# cat /etc/default/useradd  
# useradd defaults file
GROUP=100       # 在useradd使用-N或/etc/login.defs中USERGROUPS_ENAB=no时表示创建
                # 用户时不创建同用户名的主组(primary group)，此时新建的用户将默认以
                # 此组为主组，网上关于该设置的很多说明都是错的，具体可看man useradd
                # 的-g选项或useradd -D的-g选项
HOME=/home      # 把用户的家目录建在/home中
INACTIVE=-1     # 是否启用帐号过期设置(是帐号过期不是密码过期)，-1表示不启用
EXPIRE=         # 帐号过期时间，不设置表示不启用
SHELL=/bin/bash # 新建用户默认的shell类型
SKEL=/etc/skel  # 指定骨架目录，前文的/etc/skel就在这里
CREATE_MAIL_SPOOL=yes  # 是否创建用户mail缓冲
```

man useradd的useradd -D选项介绍部分说明了这些项的意义。



# 用户和组管理命令

## useradd和adduser

adduser是useradd的一个软链接。

```
useradd [options] login_name
选项说明：
-b：指定家目录的basedir，默认为/home目录
-d：指定用户家目录，不写时默认为/home/user_name
-m：要创建家目录时，若家目录不存在则自动创建，若不指定该项且/etc/login.defs中的CREATE_HOME未启用时将不会创建家目录
-M：显式指明不要创建家目录，会覆盖/etc/login.defs中的CREATE_HOME设置
 
-g：指定用户主组，要求组已存在
-G：指定用户的辅助组，多个组以逗号分隔
-N：明确指明不要创建和用户名同名的组名
-U：明确指明要创建一个和用户名同名的组，并将用户加入到此组中

-o：允许创建一个重复UID的用户，只有和-u选项同时使用时才生效
-r：创建一个系统用户。useradd命令不会为此选项的系统用户创建家目录，除非明确使用-m选项
-s：指定用户登录的shell，默认留空。此时将选择/etc/default/useradd中的SHELL变量设置
-u：指定用户uid，默认uid必须唯一，除非使用了-o选项
-c：用户的注释信息 

-k：指定骨架目录(skeleton)
-K：修改/etc/login.defs文件中有关于用户的配置项，不能修改组相关的配置。设置方式为KEY=VALUE，如-K UID_MIN=100
-D：修改useradd创建用户时的默认选项，就修改/etc/default/useradd文件
-e：帐户过期时间，格式为"YYYY-MM-DD"
-f：密码过期后，该账号还能存活多久才被禁用，设置为0表示密码过期立即禁用帐户，设置为-1表示禁用此功能
-l：不要将用户的信息写入到lastlog和faillog文件中。默认情况下，用户信息会写入到这两个文件中

useradd -D [options]
修改/etc/default/useradd文件
选项说明：不加任何选项时会列出默认属性
-b, --base-dir BASE_DIR
-e, --expiredate EXPIRE_DATE
-f, --inactive INACTIVE
-g, --gid GROUP
-s, --shell SHELL
```

示例：

```
[root@xuexi ~]# useradd -D -e "2016-08-20"    # 设置用户2016-08-20过期

[root@xuexi ~]# useradd -D
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=2016-08-20
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes

[root@xuexi ~]# cat /etc/default/useradd
# useradd defaults file
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=2016-08-20
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

useradd创建用户时，默认会自动创建一个和用户名相同的用户组，这是/etc/login.defs中的USERGROUP_ENAB变量控制的。

useradd创建普通用户时，不加任何和家目录相关的选项时，是否创建家目录是由/etc/login.defs中的CREATE_HOME变量控制的。

## 批量创建用户newusers

newusers用于批量创建或修改已有用户信息。在创建用户时，它会读取/etc/login.defs文件中的配置项。

```
newusers [options] [file]
```

newusers命令从file中或标准输入中读取要创建或修改用户的信息，文件中每行格式都一样，一行代表一个用户。格式如下：

```
pw_name:pw_passwd:pw_uid:pw_gid:pw_gecos:pw_dir:pw_shell
```

各列的意义如下：

- pw_name：用户名，若不存在则新创建，否则修改已存在用户的信息
- pw_passwd：用户密码，该项使用明文密码，在修改或创建用户时会按照指定的算法自动对其进行加密转换
- pw_uid：指定uid，留空则自动选择uid。如果该项为已存在的用户名，则使用该用户的uid，但不建议这么做，uid应尽量保证唯一性
- pw_gid：用户主组的gid或组名。若给定组不存在，则自动创建组。若留空，则创建同用户名的组，gid将自动选择
- pw_gecos：用户注释信息
- pw_dir：指定用户家目录，若不存在则自动创建。留空则不创建。注意，newusers命令不会递归创建父目录，父目录不存在时将会给出信息，但newusers命令仍会继续执行以完成创建剩下的用户，所以这些错误的用户家目录需要手动去创建
- pw_shell：指定用户的默认shell


```
newusers [options] [file]
选项说明：
-c：指定加密方法，可选DES,MD5,NONE,SHA256和SHA512
-r：创建一个系统用户
```
newusers首先尝试创建或修改所有指定的用户，然后将信息写入到user和group的文件中。如果尝试创建或修改用户过程中发生错误，则所有动作都将回滚，但如果在写入过程中发生错误，则写入成功的不会回滚，这将可能导致文件的不一致性。要检查用户、组文件的一致性，可以使用showdow-utils包提供的grpck和pwck命令。

示例：

```
$ cat /tmp/userfile
zhangsan:123456:2000:2000::/home/zhangsan:/bin/bash
lisi:123456:::::/bin/bash

$ newusers -c SHA512 /tmp/userfile   

$ tail -2 /etc/passwd
zhangsan:x:2000:2000::/home/zhangsan:/bin/bash
lisi:x:2001:2001:::/bin/bash

$ tail -2 /etc/shadow
zhangsan:$6$aI1Mk/krF$xN0TFOIRibrb/mYngJ/sV3M7g4zOxqOh8CWyDlI0uwmr5qNTzsmwauRFvCpfLtvtiJYZ/5bil.XfJMNB.sqDY1:17323:0:99999:7:::
lisi:$6$bngXo/V6wWW$.TlQCJtEm9krBX0Oiep/iahS59a/BwVYcSc8F9lAnMGF55K6W5YoUZ2nK6WkMta3p7sihkxHm/AuNrrJ6hqNn1:17323:0:99999:7:::
```

![](/img/referer.jpg)

## groupadd

创建一个新组。

```
groupadd [options] group
选项说明：
-f：如果要创建的组已经存在，默认会错误退出，使用该选项则强制创建且以正确状态退出，只不过gid可能会不受控制。
-g：指定gid，默认gid必须唯一，除非使用了-o选项。
-K：修改/etc/login.defs中关于组相关的配置项。配置方式为KEY=VALUE，例如-K GID_MIN=100 -K GID_MAX=499
-o：允许创建一个非唯一gid的组
-r：创建系统组
```

## 修改密码passwd

修改密码的工具。默认passwd命令不允许为用户创建空密码。

passwd修改密码前会通过pam认证用户，pam配置文件中与此相关的设置项如下：

```
passwd password requisite pam_cracklib.so retry=3
passwd password required pam_unix.so use_authtok
```

命令的用法如下：

```
passwd options [username]
选项说明：
-l：锁定指定用户的密码，在/etc/shadow的密码列加上前缀"!"或"!!"。这种锁定不是完全锁定，使用ssh公钥还是能登录。要完全锁定，使用chage -E 0来设置帐户过期。
-u：解锁-l锁定的密码，解锁的方式是将/etc/shadow的密码列的前缀"!"或"!!"移除掉。但不能移除只有"!"或"!!"的项。
--stdin：从标准输入中读取密码
-d：删除用户密码，将/etc/shadow的密码列设置为空
-f：指定强制操作
-e：强制密码过期，下次登录将强制要求修改密码
-n：密码最小使用天数
-x：最大密码使用天数
-w：过期前几天开始提示用户密码将要过期
-i：设置密码过期后多少天，用户才过期。用户过期将被禁用，修改密码也无法登陆。
```

## 批量修改密码chpasswd

以批处理模式从标准输入中获取提供的用户和密码来修改用户密码，可以一次修改多个用户密码。也就是说不用交互。适用于一次性创建了多个用户时为他们提供密码。

```
chpasswd [-e -c] "user:passwd"
-c：指定加密算法，可选的算法有DES,MD5,NONE,SHA256和SHA512
user:passwd为用户密码对，其中默认passwd是明文密码，可以指定多对，每行一个用户密码对。前提是用户是已存在的。
-e：passwd默认使用的是明文密码，如果要使用密文，则使用-e选项。参见man chpasswd
```

chpasswd会读取/etc/login.defs中的相关配置，修改成功后会将密码信息写入到密码文件中。

该命令的修改密码的处理方式是先在内存中修改，如果所有用户的密码都能设置成功，然后才写入到磁盘密码文件中。在内存中修改过程中出错，则所有修改都回滚，但若在写入密码文件过程中出错，则成功的不会回滚。

示例：

修改单个用户密码。

```
$ echo "user1:123456" | chpasswd -c SHA512
```

修改多个用户密码，则提供的每个用户对都要分行。

```
$ echo  -e 'usertest:123456\nusertest2:123456' | chpasswd
```

更方便的是写入到文件中，每行一个用户密码对。

```
$ cat /tmp/passwdfile
zhangsan:123456
lisi:123456

$ chapasswd -c SHA512 </tmp/passwdfile
```

## chage

chage命令主要修改或查看和密码时间相关的内容。具体的看man文档，可能用到的两个选项如下：

```
-l：列出指定用户密码相关信息
-E：指定帐户(不是密码)过期时间，所以是强锁定，如果指定为0，则立即过期，即直接锁定该用户
```

```
[root@server2 ~]#  chage -l zhangsan
Last password change                                    : Jun 06, 2017
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7

[root@server2 ~]# chage -E 0 zhangsan

[root@server2 ~]# chage -l zhangsan 
Last password change                                    : Jun 06, 2017
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : Jan 01, 1970
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

## 删除用户和组

userdel命令用于删除用户。

```
userdel [options] login_name
-r：递归删除家目录，默认不删除家目录。
-f：强制删除用户，即使这个用户正处于登录状态。同时也会强制删除家目录。
```

一般不直接删除家目录，即不用-r，可以vim /etc/passwd，将不需要的用户直接注释掉。

groupdel命令删除组。如果要删除的组是某用户的主组，需要先删除主组中的用户。

## usermod

修改帐户属性信息。必须要确保在执行该命令的时候，待修改的用户没有在执行进程。

```
usermod [options] login
选项说明：
-l：修改用户名，仅仅只是改用户名，其他的一切都不会改动(uid、家目录等)
-u：新的uid，新的uid必须唯一，除非同时使用了-o选项
-g：修改用户主组，可以是以gid或组名。对于那些以旧组为所属组的文件(除原家目录)，需要重新手动修改其所属组
-m：移动家目录内容到新的位置，该选项只在和-d选项一起使用时才生效
-d：修改用户的家目录位置，若不存在则自动创建。默认旧的家目录不会删除
    如果同时指定了-m选项，则旧的家目录中的内容会移到新家目录
    如果当前用户家目录不存在或没有家目录，则也不会创建新的家目录
-o：允许用户使用非唯一的UID
-s：修改用的shell，留空则选择默认shell
-c：修改用户注释信息

-a：将用户以追加的方式加入到辅助组中，只能和-G选项一起使用
-G：将用户加入指定的辅助组中，若此处未列出某组，而此前该用户又是该组成员，则会删除该组中此成员

-L：锁定用户的密码，将在/etc/shadow的密码列加上前缀"!"或"!!"
-U：解锁用户的密码，解锁的方式是移除shadow文件密码列的前缀"!"或"!!"
-e：帐户过期时间，时间格式为"YYYY-MM-DD"，如果给一个空的参数，则立即禁用该帐户
-f：密码过期后多少天，帐户才过期被禁用，0表示密码过期帐户立即禁用，-1表示禁用该功能
```

同样，还有groupmod修改组信息，用法非常简单，几乎也用不上，不多说了。

## vipw和vigr

vipw和vigr是编辑用户和组文件的工具，vipw可以修改/etc/passwd和/etc/shadow，vigr可以修改/etc/group和/etc/gshadow，用这两个工具比较安全，在修改的时候会检查文件的一致性。

删除用户出错时，提示用户正在被进程占用。可以使用vi编辑/etc/paswd和/etc/shadow文件将该用户对应的行删除掉。也可以使用vipw和vipw -s来分别编辑/etc/paswd和/etc/shadow文件。它们的作用是一样的。

## 手动创建用户

手动创建用户的全过程：需要管理员权限。

- 在/etc/group中添加用户所属组的相关信息。如果用户还有辅助组则在对应组中加入该用户作为成员。
- 在/etc/passwd和/etc/shadow中添加用户相关信息。此时指定的家目录还不存在，密码不存在，所以/etc/shadow的密码位使用"!!"代替。
- 创建家目录，并复制骨架目录中的文件到家目录中。

    ```
    $ mkdir /home/user_name
    $ cp -r /etc/skel /home/user_name。
    ```

- 修改家目录及子目录的所有者和属组。

    ```
    $ chown -R user_name:user_name /home/user_name
    ```

- 修改家目录及子目录的权限。例如设置组和其他用户无任何权限但所有者有。

    ```
    $ chmod -R 700 /home/user_name
    ```

到此为止，用户已经创建完成了，只是没有密码，所以只能su，不能登录。

- 生成密码。可直接使用passwd命令创建密码，也可使用openssl passwd生成密码。但openssl passwd生成的密码只能是MD5算法的，很容易被破解

    ```
    # 生成使用md5算法的密码，然后将其复制到/etc/shadow对应的密码位
    # 其中-1是指md5，-salt '12345678'是使用8位字符创建密码的杂项
    $ openssl passwd -1 -salt '12345678' '123456'
    ```

- 测试手动创建的用户是否可以正确登录。

以下是全过程。

```
$ mkdir /tmp/12;cp /etc/group /etc/passwd /etc/shadow /tmp/12/ # 备份
$ echo "userX:x:666" >> /etc/group
$ echo "userX:x:666:666::/home/userX:/bin/bash" >> /etc/passwd
$ echo 'userX:!!:17121:0:99999::::' >> /etc/shadow
$ cp -r /etc/skel /home/userX
$ chown -R userX:userX /home/userX
$ chmod -R go= /home/userX
$ passwd --stdin userX <<< '123456'
```

测试使用userX是否可以登录。

如果是使用openssl passwd创建的密码。那么使用下面的方法将这部分密码替换到/etc/shadow中。

```
$ field=$(tail -1 /etc/shadow | cut -d":" -f2)
$ password=$(openssl passwd -1 -salt 'abcdefg' 123456)
$ sed -i '$s%'$field'%'$password'%' /etc/shadow
```



# 其他用户相关命令

## finger查看用户信息

从CentOS 6版本开始就没有该命令了，要先安装。

```
shell> yum -y install finger

shell> useradd zhangsan

shell> finger zhangsan
Login: zhangsan                           Name:
Directory: /home/zhangsan                 Shell: /bin/bash
Never logged in.
No mail.
No Plan.
```

## id

```
id username
-u：得到uid
-n：得到用户名而不是uid
-z：无任何空白字符输出模式，不能在默认的格式下使用。
```

示例：

```
shell> id root
uid=0(root) gid=0(root) groups=0(root)

shell> id wangwu
uid=500(wangwu) gid=500(wangwu) groups=500(wangwu)

shell> id -u wangwu
500

shell> id -u -z wangwu
2002[root@server2 ~]#
```

## users

查看当前正在登陆的用户名。

## last

查看最近登录的用户列表，其实last查看的是/var/log/wtmp文件。

-n 显示行数：列出最近几次登录的用户

```
[root@xuexi ~]# last -4
root     pts/0        192.168.100.1    Wed Mar 30 15:16   still logged in  
root     pts/1        192.168.100.1    Wed Mar 30 14:21 - 14:21  (00:00)   
root     pts/1        192.168.100.1    Wed Mar 30 14:04 - 14:10  (00:06)   
root     pts/0        192.168.100.1    Wed Mar 30 13:12 - 15:16  (02:04)   
 
wtmp begins Thu Feb 18 20:59:39 2016
```

## lastb

查看谁尝试登陆过但没有登录成功的。即能够审核和查看谁曾经不断的登录，可能那就是黑客。

-n:只列出最近的n个尝试对象。

![](/img/referer.jpg)

## who和w

都是查看谁登录过，并干了什么事

w查看的信息比who多。

```
shell> who
root     tty1         2017-06-07 00:49
root     pts/0        2017-06-07 02:06 (192.168.100.1)

shell> w
08:26:38 up 18:48,  2 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     tty1                      00:49    7:36m  0.24s  0.24s -bash
root     pts/0    192.168.100.1    02:06    6.00s  0.97s  0.02s w
```

其中w的第一行，分别表示当前时间，已开机时长，当前在线用户，过去1、5、15分钟的平均负载率。这一行和uptime命令获取的信息是完全一致的。

## lastlog

可以查看登录的来源IP

-u 指定查看用户

```
shell> lastlog|head -n 10
Username         Port     From             Latest
root             pts/0    192.168.100.1    Wed Mar 30 15:16:25 +0800 2016
bin                                        **Never logged in**
daemon                                     **Never logged in**
adm                                        **Never logged in**
lp                                         **Never logged in**
sync                                       **Never logged in**
shutdown                                   **Never logged in**
halt                                       **Never logged in**
mail                                       **Never logged in**
```

# su和sudo

见[su和sudo介绍](/linux/su_and_sudo)。



