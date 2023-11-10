---
title: Linux将不断生成的新数据发送到远程节点
p: linux/data_to_remote.md
date: 2020-05-26 13:20:42
tags: Linux
categories: Linux
---

------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

------

# Linux将不断生成的新数据发送到远程节点

**方法1.在远程节点上使用nc或socat监听一个端口，在本地端向该端口发送数据**。

```bash
# 远程节点
nc -l -k 12345 | cat

# 本地节点
while true;do
  echo $RANDOM
  sleep 1
done >/dev/tcp/192.168.200.142/12345
```

**方法2.使用ssh命令的远程命令功能，ssh命令默认会将本地端的标准输入写到目标节点**。

```bash
# 本地节点
while true;do
  echo $RANDOM
  sleep 1
done | ssh 192.168.200.142 'cat /dev/stdin >/tmp/x.log'

# 远程节点验证数据
tail -f /tmp/x.log
```

以下是使用nc的方式发送新生成数据的gif动图：

![](/img/linux/data_to_remote.gif)