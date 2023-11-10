---
title: 操作系统修炼秘籍（24）：进程表和进程数据结构以及上下文切换
p: os/pcb_context_switch.md
date: 2019-11-21 22:20:44
tags: OS
categories: OS
---

-----------

**[点我查看操作系统秘籍连载](https://www.junmajinlong.com/os/index/)**

-----------

# 进程表和进程数据结构

内核负责管理维护所有进程，为了管理进程，内核在内核空间维护了一个称为**进程表**（Process Table）的数据结构，这个数据结构中记录了所有进程，每个进程在数据结构中都称为一个**进程表项**（Process Table Entry），如图。

![](/img/os/733013-20191121224234787-622882346.jpg)


从图中可知，进程表中除了记录了所有进程的PID，还使用一个字段记录了所有进程的指针，指向每个进程的**进程控制块**（Process Control Block，PCB），请记住PCB这个词，它太重要了。

![](/img/os/733013-20191121224332142-507157457.jpg)


既然PCB代表的是进程，在这个数据结构（task_struct）中自然保留了和进程相关的很多信息，至少在进行上下文切换时，能够保存下在CPU中关于当前运行进程的一些重要寄存器信息（所以，PCB代表的是未执行的进程，CPU中某些寄存器中的数据代表当前正在运行的进程）。以下是PCB中包含的内容，有兴趣的话可以了解下各项是什么东西：

```
1.Process management：
Registers
Program counter
Program status word
Stack pointer
Process state
Priority
Scheduling parameters Process ID
Parent process
Process group
Signals
Time when process started CPU time used
Children’s CPU time
Time of next alarm
 
2.Memory management：
Pointer to text segment info 
Pointer to data segment info 
Pointer to stack segment info
 
3.File management：
Root directory Working directory File descriptors User ID
Group ID
```

PCB包含了进程非常重要的信息，是上下文切换的关键，它保存在每个进程的**内核栈**中（是否还记得前面提到过的每个进程都有两个栈空间：用户栈和内核栈）。


# 上下文切换

上下文切换是操作系统同时运行多个程序的基本保证，它表现为：运行一会进程A，再运行一会进程B。

在上下文切换时，有一些必要的事情要做。例如，CPU要从进程A切换到进程B时，必然要保存好进程A上下文，以便下次切换回进程A的时候能够原样恢复继续运行。另一方面，既然要切换到进程B，必然还要恢复之前已保存好的进程B的上下文，才能原样恢复进程B并继续执行进程B。很多时候，保存上下文被称为"保护现场"，恢复上下文被称为"恢复现场"。

具体的过程是，当进程A要切换到进程B时，**首先要陷入内核，然后内核将CPU中关于进程A的进程信息（即某些寄存器中的值）保存在进程A内核栈中的PCB结构中，然后从进程B内核栈的PCB结构中恢复进程B的信息到CPU的某些寄存器中，再退出内核模式回到进程B，这样CPU就开始执行进程B了**。

下图描述了进程P0切换到进程P1的基本流程。

![](/img/os/733013-20200229094017637-56957918.png)

另外需要说明的，上下文切换时保护现场和恢复现场都是有代价的，而且代价并不止于这里所描述的PCB保存和恢复。在一个进程正常运行过程中，CPU的各个寄存器和高速缓存中还保存了当前进程的很多数据和状态，在上下文切换时会丢失很多信息，在重新恢复该进程时，将刷新这些状态，不得不重新查询或计算。

所以，上下文切换频繁，必须考虑其代价问题，甚至有时候会将太多频繁的切换当作影响性能的关键指标。