---
title: 操作系统系列文章
p: os/index.md
date: 2019-09-06 17:37:29
tags: OS
categories: OS
---

# OS书籍推荐

入门推荐书籍1：**《计算机的心智：操作系统之哲学原理》**(建议看第一版)。要阅读这本书，除了几个概念(比较常见的是"中断")，完全不需要任何基础(没错，不需要C和任何语言的知识)，看故事一样就可以将操作系统的进程、线程、内存、IO、多核全部有个了解。当然，这本书只能浅层次、全面地了解操作系统，适合入门操作系统。

入门推荐书籍2：**《Operating Systems: Three Easy Pieces》(OSTEP)**，总共50章。如果说上面推荐的《计算机的心智》是看故事，那么这本书就是从知识点的角度去系统性地认识操作系统，但偏偏没有任何难度。本书2019年6月出了中文版《操作系统导论》。

入门推荐书籍3：**《Operating.System.Concepts.10th》，中文版《操作系统概念》**，OSTEP描述的多是原理和概念，操作系统概念是细节加原理加概念，写作方式是比较大众化的方式，本书结合OSTEP看，基本上能将操作系统相关的基础都了解清楚。

入门推荐书籍4：**《Linux-UNIX系统编程手册（上、下册）》或《UNIX环境高级编程》(APUE)**，系统编程的体系中，有关进程、内存等方面的内容，对于了解操作系统也是非常有帮助的，这可能需要一点C基础，至少，要能看的懂C。

# 操作系统修炼秘籍

本秘籍只专注于介绍操作系统中的一些概念和术语，从前向后循序渐进，所以建议从前向后不要跳过，否则断层而突然出现的概念导致看不懂。

- [操作系统修炼秘籍(1): 秘籍简介](/os/introduce)  
- [操作系统修炼秘籍(2): 并行的假象和分时系统](/os/time_sharing)  
- [操作系统修炼秘籍(3): 了解一点重要的操作系统发展历史](/os/os_history)  
- [操作系统修炼秘籍(4): 内核态和用户态](/os/kernel_user_space)  
- [操作系统修炼秘籍(5): 中断](/os/interrupt)  
- [操作系统修炼秘籍(6): 系统调用](/os/syscalls)  
- [操作系统修炼秘籍(7): Idle进程](/os/idle)  
- [操作系统修炼秘籍(8): 虚拟内存简介](/os/mem_introduce)  
- [操作系统修炼秘籍(9): 虚拟内存分段](/os/mem_segment)  
- [操作系统修炼秘籍(10): 栈空间之用户栈和内核栈](/os/mem_stack)  
- [操作系统修炼秘籍(11): 分页和页表](/os/mem_paging)  
- [操作系统修炼秘籍(12): 页翻译——快速地址转换](/os/mem_TLB)  
- [操作系统修炼秘籍(13): OOM和Swap分区](/os/mem_oom_swap)  
- [操作系统修炼秘籍(14): 两个缓冲空间：kernel buffer和io buffer](/os/kernelBuffer_ioBuffer)  
- [操作系统修炼秘籍(15): IO操作和DMA、RDMA](/os/io_dma_rdma)  
- [操作系统修炼秘籍(16): 进程间通信(1)：简介](/os/IPC_introduce)  
- [操作系统修炼秘籍(17): 进程间通信(2)：管道](/os/IPC_pipe)  
- [操作系统修炼秘籍(18): 进程间通信(3)：套接字](/os/IPC_socket)  
- [操作系统修炼秘籍(19): 进程间通信(4)：文件映射和内存共享](/os/IPC_filemap_memshare)  
- [操作系统修炼秘籍(20): 进程间通信(5)：消息队列和信号](/os/IPC_msq_signal)  
- [操作系统修炼秘籍(21): 进程间通信(6)：信号量](/os/IPC_semaphore)  
- [操作系统修炼秘籍(22): 进程间通信(7)：锁](/os/IPC_lock)  
- [操作系统修炼秘籍(23): 程序如何变成进程](/os/program_process)  
- [操作系统修炼秘籍(24): 进程表和进程数据结构以及上下文切换](/os/pcb_context_switch)  
- [操作系统修炼秘籍(25): 进程状态以及状态转换](/os/process_state_switch)  
- [操作系统修炼秘籍(26): 进程调度算法图解说明](/os/process_schedule)  
- [操作系统修炼秘籍(27): Linux进程调度和调整优先级](/os/nice_renice_ionice)  
- [操作系统修炼秘籍(28): Linux进程的创建](/os/fork_process)  
- [操作系统修炼秘籍(29): Linux进程退出和wait/waitpid](/os/wait_waitpid)  
- [操作系统修炼秘籍(30): Linux僵尸进程和孤儿进程](/os/zombie_orphan)  
- [操作系统修炼秘籍(31): 并行和并发](/os/parallelism_concurrency)  

# 番外篇

- [1.关于汇编和计算机的一些基本知识](/assembly/assembly_basis/)  
- [2.关于寄存器和CPU执行指令的一些基本知识](/assembly/assembly_register/)  
- [3.关于CPU的一些基本知识总结](/os/cpu)  
- [4.关于总线的一些基本知识总结](/os/bus)  
- [5.关于CPU上的高速缓存](/os/cpu_cache)  
- [6.关于多CPU、多核和多线程](/os/multi_cpu)  
- [7.操作系统中随处可见的抽象](/coding/abstraction)  




