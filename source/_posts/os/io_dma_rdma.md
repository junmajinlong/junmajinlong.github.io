---
title: 操作系统修炼秘籍（15）：IO操作和DMA、RDMA
p: os/io_dma_rdma.md
date: 2019-11-17 08:20:41
tags: OS
categories: OS
---


-----------

**[点我查看操作系统秘籍连载](https://www.junmajinlong.com/os/index/)**

-----------



# I/O操作和DMA、RDMA

用户进程想要执行IO操作时（例如想要读磁盘数据、向磁盘写数据、读键盘的输入等等），由于用户进程工作在用户模式下，它没有执行这些操作的权限，只能通过发起对应的系统调用请求操作系统帮忙完成这些操作。这里因为系统调用产生中断将陷入到内核，进行一次上下文切换操作。

内核进程帮忙执行IO操作时，由于IO操作相比于CPU来说是极慢的操作，CPU不应该等待在这个过程中，而是切换到其它进程上去执行其它任务。这里再次涉及到一次上下文切换：从内核态回到用户态的其它进程。

![](/img/os/733013-20191117101508460-1262671394.jpg)

DMA要求硬件的支持，需要在硬件中集成一个小型的『CPU』，比如现在的机械硬盘、固态硬盘、网卡等硬件都带有DMA功能，这样操作系统要执行IO操作时，直接将相关指令发送给这些DMA硬件，DMA处理器负责IO操作，而操作系统这时可以放弃CPU，让CPU去执行其它进程。例如对于读磁盘文件时，操作系统将相关指令以及数据应写在哪个内存地址发送给DMA硬件后，由DMA硬件去读写数据到指定内存地址，当IO操作完成后，DMA硬件通过总线发送一个硬件中断给CPU，于是陷入到内核态（这里涉及了一次上下文切换），内核就知道了IO已经完成，于是将Kernel Buffer数据拷贝到用户进程的IO Buffer，并准备调度用户进程（再次上下文切换）。

假如不使用DMA硬件的话，那么IO操作过程中，操作系统将多次参与，负责将硬件数据读入或读出内存，操作系统参与意味着要陷入到内核态，并且获取CPU控制权，这也意味着要进行大量的上下文切换以及占用大量CPU资源。

而使用DMA后，只有4次必要的上下文切换，且IO操作的过程中完全不需要消耗CPU资源。

除了DMA，还有更高级的**RDMA**（Remote Direct Memory Access）机制，它需要操作系统和硬件的支持，还需要编写RDMA方式的代码。

前面介绍缓冲空间时提到过，一般情况下，每个用户进程要读、写数据，都会经过两个必要的缓冲层：内核空间的Kernel Buffer、用户空间的IO Buffer。例如读文件数据时，先将数据拷贝到内核的缓冲空间（page cache），然后陷入内核，内核将该缓冲空间数据拷贝到用户空间的缓冲空间（IO Buffer），当调度到用户进程时，用户进程从自己的缓冲空间读取数据。

DMA机制并没有绕过这两个缓冲层，但使用RDMA机制，程序可以直接绕过Kernel Buffer，内核发现是RDMA操作后，直接告诉RDMA硬件将读取的数据（写操作也一样）写入到用户空间的IO Buffer，而不需要先拷贝到Kernel Buffer，再拷贝到IO Buffer。虽然RDMA机制相比DMA不会减少上下文切换次数，但是它减少了内存数据拷贝的过程，相当于是使用了O_DIRECT标记的直接IO技术。

DMA和RDMA两种技术对比如图：RDMA一般实现在网卡上，但出于方便理解，下图直接使用磁盘来描述

![](/img/os/733013-20191117101615168-796363325.jpg)

像这种绕过内核功能的技术，通常称为**内核旁路**（Kernel Bypass），RDMA技术内核旁路的是一种，还有像TOE也是内核旁路的一种。

虽然RDMA比较优秀，但是它需要硬件、操作系统和代码的同时支持，对编程而言是一个比较大的冲击，所以目前使用的非常少。
