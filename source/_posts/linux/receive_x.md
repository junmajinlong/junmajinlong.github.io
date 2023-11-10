---
title: 使用xmanager接收Linux的图形界面
p: linux/receive_x.md
date: 2019-10-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 使用xmanager接收Linux的图形界面

假设在win(192.168.0.101)上安装了xmanager，想接收来自linux(192.168.100.16)的图形界面。

1.在win端打开`Xmanager - Passive`

![](/img/linux/733013-20170218201257191-1251233857.png)

2.在linux上设置DISPLAY环境变量

```
export DISPLAY=192.168.0.101:0.0
```

其中192.168.0.101是接收图形的客户端地址。`0.0`是一个两边从0开始的序列，一般就这样即可。

3.测试。

在linux端安装一个xclock测试：

```
yum -y install xclock
```

运行xclock命令看win端是否能接收到图形界面

![](/img/linux/733013-20170218201258847-584450209.png)

 
