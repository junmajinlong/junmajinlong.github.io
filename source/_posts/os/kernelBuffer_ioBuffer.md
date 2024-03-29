---
title: 操作系统修炼秘籍（14）：两个缓冲空间：kernel buffer和io buffer
p: os/kernelBuffer_ioBuffer.md
date: 2019-11-16 08:20:41
tags: OS
categories: OS
---


-----------

**[点我查看操作系统秘籍连载](https://www.junmajinlong.com/os/index/)**

-----------



# 两个缓冲空间：kernel buffer和io buffer

先看一张图，稍后将围绕这张图展开描述。图中的fd table、open file table以及两个inode table都可以不用理解，只需要知道它们体现出来的文件描述符和磁盘文件之间的对应关系：文件描述符fd（例如图中的fd=3）是对应磁盘上文件的。

![](/img/os/733013-20191116085939253-853333343.jpg)

在Linux下，我们经常会在IO操作时不可避免的涉及到文件描述符，因为Linux下的所有IO操作都是通过文件描述符来完成的。但是，文件描述符是一个非常底层的概念，通过它操作的数据，都是二进制数据，所以通过文件描述符完成IO的模式通常也称为**裸IO**（Raw IO）。而且，直接通过底层的文件描述符进行编程会比较麻烦，因为是二进制数据，它缺少很多功能，比如无法指定编码，无法指定换行符（换行符有多种：\n、\n\r、\r）等等。注意fd是用户空间的，它仅仅是一个数值而已，并不是想象中感觉比较底层就在内核空间。

所以，现代高级语言（比如C、Python、Java、Golang）都提供了比文件描述符更高一层次的**标准IO库**，比如C的标准IO库是stdio，Python的标准IO库是IO模块，等等。使用这些标准IO库中的函数进行IO操作时，都会使用比文件描述符更高一层次的对象，例如C中称为IO流（io stream），其它面向对象的语言中一般称为IO对象，为了方便说明，这里统称为IO对象。上图中的F就是文件对象。

标准IO库可以看作是文件描述符的更高层次的封装，提供了比文件描述符操作IO更多的功能。例如，可以在IO对象上指定编码、指定换行符，此外还在用户空间提供了一个标准IO库的缓冲空间，通常可称为**stdio buffer**或**IO buffer**，而这些功能在文件描述符上都是没有的。另外，标准IO库既然是高层封装，当然也会提供用户不使用这些功能（比如不使用IO Buffer），而是直接使用文件描述符，那么这时候的文件对象就相当于是文件描述符了，这时候的IO操作模式也就是裸IO模式。

![](/img/os/733013-20191116085726512-440991229.jpg)

所有从硬件读取或写入到硬件的数据，默认都会经过操作系统维护的这个Kernel Buffer。正如上图中描述的是读数据过程。

例如，cat进程想要读取a.log文件，cat进程是用户空间进程，它自身没有权限打开文件以及读文件数据，它只能通过系统调用的方式陷入内核，请求操作系统帮助读取数据，操作系统读取数据后会将数据放入到page cache（刚才已说明，对于普通文件维护的Kernel buffer称为page cache或buffer cache）。然后还要将内核空间page cache中的数据拷贝到用户空间的IO Buffer缓冲空间（因为cat程序的源代码中使用了标准IO库stdio），然后cat进程从自己的IO Buffer中读取数据。这就是整个读数据的过程。

需要注意的是，虽然这两段缓冲空间都在内存中，但仍然有拷贝操作，因为内核的内存空间和用户进程的虚拟内存空间是隔离的，用户空间进程没有权限访问到内核空间的内存，但是内核具有最高权限，允许访问任何内存地址。换句话说，在将Kernel Buffer的数据拷贝到IO Buffer空间的过程中，需要陷入到内核，OS需要掌控CPU。

此外，Linux也提供了所谓的**直接IO模式**，只需使用一个称为**O_DIRECT**的标记即可，这时会绕过Kernel Buffer，直接将硬件数据拷贝到用户空间。虽然看上去直接IO少了一个层次的参与感觉性能会更优秀，但实际上并非如此，操作系统为内核缓冲空间做了非常多的优化，使得并不会因此而降低性能。最典型且常见的一个优化是**预读**功能，它表示在读数据时，会比所请求要读取的数据量多读一点放入到Kernel Buffer，这样在下次读取接下来的一段数据时可以直接从Kernel Buffer中取数据，而无需再和硬件IO交互。所以，使用直接IO模式的场景是非常少的，一般只会在自带了完整缓冲模型的大型软件（比如数据库系统）上可能会使用直接IO模式。

上面所描述的都是读操作，关于写操作，这里不再多花篇幅去描述，整体过程和读是类似的，都会经过IO Buffer和Kernel Buffer，只是其中一些细节有所不同，如果感兴趣，可以阅读《Linux/Unix系统编程手册》的第13章。