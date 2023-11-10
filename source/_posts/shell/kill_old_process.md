---
title: Shell脚本杀掉除自己外的旧进程
p: shell/kill_old_process.md
date: 2020-04-04 10:20:43
tags: Shell
categories: Shell
---


--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# Shell脚本杀掉除自己外的旧进程

在写后台Shell脚本的时候，这是比较常见的一个需求。比如之前运行了一个叫做a.sh的脚本在后台运行，后来更新了a.sh脚本想重新运行，但却不想手动杀掉已经存在的后台a.sh进程。

命令其实非常简单：

```shell
# from: www.junmajinlong.com
kill $(pgrep -f "${0//./\\.}" | grep -v $BASHPID) &>/dev/null
```

其中`pgrep -f $0 | grep -v $BASHPID`是筛选出除脚本自己之外的旧进程的PID。

这里的`$0`做了些处理，防止脚本路径名称中包含了点`.`而导致模式匹配出现误匹配。比如`b.sh`脚本，在`pgrep -f $0`时会匹配出b.sh和bash，`${0//./\\.}`处理完后就得到了`b\.sh`，这样便不会匹配到bash，且没有包含`.`的路径名称也不会受到影响。

