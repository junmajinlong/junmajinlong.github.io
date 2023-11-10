---
title: Shell脚本深入教程：Bash进程替换
p: shell/script_course/shell_process_substitution.md
date: 2020-05-19 14:19:07
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# 进程替换

Bash还支持进程替换(注：有些Shell不支持进程替换)。

进程替换的语法：
```shell
<(cmd)
>(cmd)
```

进程替换和命令替换类似，都是让cmd命令先执行，因为它们都是在Shell解析命令行的阶段执行的。

进程替换先让cmd放入后台异步执行，并且不会等待cmd执行完。

其实，每个进程替换都是一个虚拟文件，只不过这个文件的内容是由cmd命令产生的(`<(cmd)`)或被cmd命令读取的(`>(cmd)`)。
```shell
$ echo <(echo www.junmajinlong.com)
/dev/fd/63
```

既然进程替换是文件，那么它就可以像文件一样被操作。比如被读取、被当作标准输入重定向的数据源等等：
```shell
# cmd做数据产生者
$ cat <(echo www.junmajinlong.com)   # 等价于cat /dev/fd/63
$ cat < <(echo www.junmajinlong.com) # 等价于cat </dev/fd/63

# cmd做数据接收者
$ echo hello world > >(grep 'llo')
$ echo hello world | tee >(grep 'llo') >(grep 'rld') >/dev/null
```
