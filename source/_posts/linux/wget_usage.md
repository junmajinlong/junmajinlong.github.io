---
title: wget命令常用选项和示例
p: linux/wget_usage.md
date: 2020-07-06 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# wget命令常用选项和示例

wget命令用来从指定的URL下载文件。wget非常稳定，它在带宽很窄的情况下和不稳定网络中有很强的适应性，如果是由于网络的原因下载失败，wget会不断的尝试，直到整个文件下载完毕。如果是服务器打断下载过程，它会再次联到服务器上从停止的地方继续下载。这对从那些限定了链接时间的服务器上下载大文件非常有用。

```
语法
wget(选项)(参数)

选项
-b：进行后台的方式运行wget；
-c：继续执行上次终端的任务；
-r：递归下载方式；
-O：指定文件名
-nc：文件存在时，下载文件不覆盖原有文件；
-nv：下载时只显示更新和出错信息，不显示指令的详细执行过程；
-P：指定下载目录；
-e：可指定网络代理参数
--show-progress：显示进度条
--no-check-certificate：下载https网站资源时可能需要使用该选项跳过证书检测的过程。

参数
URL：下载指定的URL地址。
```

## 示例1: 使用wget下载单个文件

```
wget http://www.linuxde.net/testfile.zip  
```

从网络下载一个文件并保存在当前目录，在下载的过程中会显示进度条，包含（下载完成百分比，已经下载的字节，当前下载速度，剩余下载时间）。

## 示例2: 使用wget下载文件到指定目录

```
wget -P /tmp  http://www.linuxde.net/testfile.zip 
```

下载testfile.zip到/tmp目录下。

## 示例3: 下载并以不同的文件名保存

使用"-O"选项重命名保存。

**正确：**

```
wget -O wordpress.zip  http://www.linuxde.net/download.aspx?id=1080
```

wget默认会以最后一个符号/的后面的字符来命令，对于动态链接的下载通常文件名会不正确。

**错误**：

下面的例子会下载一个文件并以名称`download.aspx?id=1080`保存:

```
wget http://www.linuxde.net/download?id=1
```

即使下载的文件是zip格式，它仍然以`download.php?id=1080`命名。

## 示例4: 使用wget断点续传

```
wget -c http://www.linuxde.net/testfile.zip
```

使用`wget -c`重新启动下载中断的文件，对于我们下载大文件时突然由于网络等原因中断非常有帮助，我们可以继续接着下载而不是重新下载一个文件。

## 示例5: 使用wget后台下载

对于下载非常大的文件的时候，我们可以使用参数-b进行后台下载

```
wget -b http://www.linuxde.net/testfile.zip
Continuing in background, pid 1840.
Output will be written to `wget-log'.
```

## 示例6: 指定下载时使用的代理

```
wget -e http_proxy=http://127.0.0.1:8118 -e https_proxy=http://127.0.0.1:8118 <URL>
```

