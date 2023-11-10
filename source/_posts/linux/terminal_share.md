---
title: 终端会话共享的两种方法
p: linux/terminal_share.md
date: 2020-08-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 通过kibitz实现终端会话实时共享

kibitz可以将一个会话(你所操作的)实时分享给本机的其它登陆用户(你想让别人看到的)。通过这个工具，你敲什么命令，输出了什么内容对方都能立即看到，用来演示很不错。

它是是expect中的一个工具，所以先安装expect。
```
yum -y install expect
```

使用方式很简单，在kibitz命令后加一个已登录的用户即可(比如你目前登陆的用户名)。例如：
```
[root@xuexi perlapp]# kibitz root
```
它会输出如下信息：
```
asking root to type:  kibitz -11913
write: root is logged in more than once; writing to pts/2
```
只需在想要接收共享会话的终端上输入`kibitz -11913`，就可以接收所有消息了。

结束共享的时候，只需在主终端上输入`exit`命令或者CTRL+D键即可退出。

还可以指定分享给哪个终端，例如当前已登录的终端有pts/0和pts/1，你所操作的是pts/0，想分享给pts/1：
```
kibitz -tty pts/1 root
```
然后将`kibitz -11913`这种握手码复制到pts/1的EOF字符后面按回车即可。

实际上这个tty选项没什么用，就算指定了tty选项，还是可以在任意终端上通过`kibitz -11913`来建立共享终端。


默认情况下，kibitz只支持将会话共享给一个人。如果想要共享给多人，则需要特殊处理。

例如，分享给两个人：
```
kibitz root kibitz root
```

它会在主会话输出：
```
asking root to type:  kibitz -15573
write: root is logged in more than once; writing to pts/3

Message from root@xuexi.longshuai.com on pts/4 at 10:55 ...
Can we talk? Run: kibitz -15587
EOF
```

两个`kibitz -NNNNN`，只需分别复制给不同终端上执行即可。

# 通过script和scriptreplay命令实现终端会话共享

使用script命令录制，使用scriptreplay播放录制的操作。共享终端的操作，则需要使用命名管道来实现。

## 通过script录制终端操作

```
[root@xuexi ~]# cd /tmp

[root@xuexi tmp]# script -t 2> timing.log -a output.session  # 开始录制
Script started, file is output.session
```

```
[root@xuexi tmp]# ls                 # 执行一个操作：命令ls
abc.sh  ab.sh  index.html  lost+found  output.session  scriptfifo  test  test1  timing.log  vmware-root
```
```
[root@xuexi tmp]# cd /tmp/test      # 再执行一个操作：命令cd
[root@xuexi test]# exit  # 结束录制
exit
Script done, file is output.session
```
其中`-t 2> timing.log`是要回放的必须选项，不加`2>`将导致开启录制后的任何输入都是乱码状态，不加`-t timing.log`将不能使用scriptreplay来回放。timing.log记录的是每个时间段输入了多少字符。通过timing.log和output.session配合可以实现回放。

注意点是，录制前保证timing.log和output.session是空文件，否则将导致回放时操作不一致。

## 通过scriptreplay回放终端操作

```
[root@xuexi test]# scriptreplay timing.log output.session
```
如果觉得回放的速度过慢(录制时有些地方停顿，比如输入了一个命令后，隔了一段时间才输入另一个命令，这段时间对于回放来说显得慢很正常)，可以修改timing.log文件。这个文件中分两个字段，第一个字段记录的是从上次输出后到该次输出的时间间隔，第二个字段是从output.session中读取的字符数。要修改回放速度，只需将第一个字段较长的间隔改短一点就可以。但是不应该改的太短，否则回放速度过快。我觉得将间隔较长的改成0.3-0.7秒，效果还不错。
```
[root@xuexi ~]# cat timing.log
0.117244 16
0.007955 1
0.298074 1       # 此处将原来的2.298074改为0.3秒
0.216628 1
0.092781 1
0.081659 1
0.083258 1
0.419445 1
0.314128 1
0.100810 1
0.083998 30
0.491283 1
0.266129 1   # 此处原来也是2.266129秒，显然经过一次命令输出之后停顿了2秒多
0.099767 1
0.127625 1
0.078809 1
0.181493 1
0.147795 1
0.115808 1
0.077416 1
0.274658 1
0.257042 1
0.524460 4
0.297133 38
0.458018 1
0.416350 1
0.187270 1
0.125467 1
0.100756 8
```


## 结合管道实时共享终端屏幕

通过管道来传输信息实现。需要一个pipe文件，并在需要展示的终端打开这个管道文件。

在终端1（作为主终端，即演示操作的终端）上使用mkfifo创建管道文件。
```
[root@xuexi tmp]# mkfifo scriptfifo

[root@xuexi tmp]# ll scriptfifo
prw-r--r-- 1 root root 0 Sep 26 13:04 scriptfifo   # 权限位前面的第一个p代表的就是pipe文件。
```

![](/img/linux/733013-20170825200421605-2073081875.png)

在终端2上打开pipe文件。
```
[root@xuexi ~]# cat /tmp/scriptfifo
```

![](/img/linux/733013-20170825200439636-531572923.png)

在终端1上使用script -f开始记录操作，之后的操作将会分享在终端2上。
```
[root@xuexi tmp]# script -f scriptfifo
```

![](/img/linux/733013-20170825200455511-557663889.png)

![](/img/linux/733013-20170825200500464-2021334641.png)


使用exit即可停止分享并退出记录行为。
```
[root@xuexi tmp]# exit
exit
Script done, file is scriptfifo
```
在被分享终端上参与分享状态后将不能执行任何操作，执行的操作会被记录下来，并在主终端停止分享后自动执行。

需要注意的是，只能给一个会话共享会话终端。如果多个会话`cat scriptfifo`，会导致共享切开显示在多个不同会话上。


