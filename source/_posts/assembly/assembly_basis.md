---
title: 汇编基础知识
p: assembly/assembly_basis.md
date: 2020-05-28 10:40:41
tags: Assembly
categories: Assembly
---


# 基础知识

## 机器语言和汇编语言

机器语言是机器指令的集合，是一系列二进制数字组成的，CPU可通过电脉冲驱动直接执行这些二进制的机器指令。因CPU的硬件设计和内部结构的不同，导致不同的CPU具有不同的机器指令集，也即不同的机器语言。

例如8086 CPU完成运算`s=768+12288-1280`时，机器码如下：

```
101110000000000000000011
000001010000000000110000
001011010000000000000101
```

可读性非常差，如果某个二进制位写错，将非常难以找出错误。

所以汇编语言出现了，汇编语言的主题是**汇编指令**。汇编指令和机器指令的区别在于指令的表示方法上，汇编指令是机器指令的更便于记忆的版本。

例如：

```
操作：将寄存器BX的内容送到寄存器AX中
机器指令版：1000100111011000
汇编指令版：mov ax,bx
```

其中寄存器是CPU中可以存放数据的器件，一个CPU可有多个寄存器，AX、BX都是寄存器的代号。

CPU只认识二进制的机器语言，不认识汇编语言的指令，汇编指令只有程序员理解，为了让汇编指令也能让CPU执行，需要使用**汇编编译器**将汇编指令翻译为机器指令。

```
程序员 -> 汇编指令(mov ax,bx) -> 编译器 -> 机器指令(1000100111011000) -> CPU
```

汇编语言有三类指令组成：

- (1).**汇编指令**：是机器指令的助记符，均有对应的二进制机器指令，是汇编语言的核心指令  
- (2).**伪指令**：没有对应的机器指令，是由编译器负责执行的，计算机自身不执行  
- (3).**其它符号**：如`+ - * /`等，也是由编译器识别的，没有对应的机器指令  

## 存储器：内存

CPU是计算器核心部件，负责所有运算。但是运算需要有指令(做什么运算)和数据(对谁做运算)，指令和数据存放在存储器中，即内存中。

CPU不直接从磁盘读取数据，如果指令和数据存放在磁盘上，则先从磁盘读取到内存，然后CPU从内存中读取数据。

所谓的指令和数据都是二进制数字，都存放在内存或磁盘上。但是CPU需要在逻辑上区分某段二进制数字是指令还是数据。比如二进制`10001001 11011000`，计算机既可以将其看作是数据89D8H(其中H用于表示前面的89D8是十六进制)进行处理，也可以将其看作是指令`mov ax,bx`进行执行。

## 存储单元

**存储器(内存)被划分为若干存储单元，从0开始顺序编号，这些编号是每个存储单元在内存中的地址**。例如0-127表示有128个存储单元。

电子计算机的最小信息单位是比特(bit)，即一个二进制的位，1字节(Byte)为8bit，即8个二进制位。

**每个存储单元可存储1字节**，所以128个存储单元可存储128字节的数据。

## CPU对内存的读写

内存被划分成存储单元，CPU要读写内存，需要三步：  

- (1).CPU要找到目标存储单元的地址信息来表示读写哪一个存储单元(**地址信息**)  
- (2).CPU要指明操作哪个部件(因为除了内存还有其它部件)，以及做读操作还是写操作(**控制信息**) 
- (3).从内存读数据，或向内存写数据(**数据信息**)  

CPU要传输信息都是通过电信号(用高低电平来表示1和0)完成，电信号需要使用导线来传送。所以，在计算机中，使用一根根的导线来连接CPU和其它部件的芯片，这一根根的导线称为**总线**。每一个CPU芯片都有许多针脚，这些针脚和总线相连，或者说，这些针脚引出总线。

从物理结构上来看，总线就是导线的集合。从逻辑上来划分，总线分为三类：  

- **地址总线**：在CPU和其它部件的芯片之间传输地址信息  
- **控制总线**：在CPU和其它部件的芯片之间传输控制信息  
- **数据总线**：在CPU和其它部件的芯片之间传输数据  

例如，从内存的3号存储单元读取数据：

- CPU通过地址总线将地址信息3发送出去  
- CPU通过控制总线发出内存读命令，选中内存芯片并通知它，将要读取数据  
- 内存将3号存储单元的数据8通过数据总线传输给CPU  

![](/img/assembly/1590646742506.png)

如果要向3号存储单元写入数据26。则：

- CPU通过地址总线将地址信息3发出  
- CPU通过控制总线发出内存写命令，选中内存芯片并通知它，将要写入数据  
- CPU将数据26通过数据总线传给内存的3号存储单元  

基于如上描述的读、写过程，读操作对应的机器指令和汇编指令分别为：

```
含义：从3号单元读取数据送到寄存器AX
机器指令：1010 0001 0000 0011 0000 0000
汇编指令：MOV AX,[3]
```

## 地址总线

CPU通多地址总线来寻找存储单元，所以地址总线上能传输多少个不同信息，就决定了CPU能对多少个存储单元进行寻址。

每根导线都只能传输高低电平两种稳定状态，分别对应二进制的1和0。所以，10根总线总共能传输10个二进制位，所以10根总线能描述2^10个数据，即0-1023。

所以，如果是10根地址总线，那么总共能寻址1024个存储单元。如果此时10根地址总线的CPU要向11号存储单元发送信息，这10根地址总线上的二进制数据将为`00 0000 1011`。

![](/img/assembly/1590651002927.png)

所以，**CPU地址总线的宽带(即有几根地址总线)决定了CPU能对多少个存储单元进行寻址(2的N次方)，也即决定了能认识多少内存地址，也即决定了最多识别多大内存**。

## 数据总线

CPU和内存或其他部件之间的数据传输是通过数据总线传输的。

**数据总线的宽度决定了CPU和外界部件之间数据传输的速度**。

例如，8根数据总线的CPU(比如8088 CPU)一次可传送8个二进制位即一个字节的数据，而16根数据总线的CPU(比如8086 CPU)可一次性传输2个字节。假设要向内存写入数据89D8H，8088和8086传输数据的方式：

![](/img/assembly/1590650926515.png)

## 控制总线

CPU通过控制总线来控制外界部件，有多种控制总线的类型。比如读信号控制线负责将CPU发出的控制信息向外传送。

有多少控制总线，决定了CPU对外界有多少种控制。所以，**控制总线的宽度决定CPU对外界部件的控制能力**。

## 内存地址空间

主板上有核心部件和一些主要部件，包括CPU、内存、外围芯片组、扩展插槽等，这些部件通过总线(地址总线、数据总线、控制总线)相连。其中扩展插槽一般插有内存条和其它接口卡。

对于不在主板上的硬件(比如显示器、音箱)，它们插在扩展插槽的接口卡上，CPU无法直接控制它们，控制它们的是扩展插槽上的接口卡。因为CPU和扩展插槽是总线相连的，所以**CPU可以控制扩展插槽上的内存条或接口卡，然后接口卡控制这些外部设备，从而让CPU间接控制这些外部设备**。

一台PC机种，装有多个存储器芯片，分为两类：RAM(可读可写)和ROM(只读)

- 主存一般由多个位置上的RAM组成：主板上的RAM、扩展插槽上的RAM，接口卡上可能也有RAM，比如显卡的RAM(即显存)  
- ROM可能也在多个地方提供，一般用于存放BIOS：主板的ROM存放主板的BIOS、显卡的ROM存放显卡的BIOS、网卡的ROM存放网卡的BIOS  

![](/img/assembly/1590652234930.png)

虽然有这么些独立的物理存储器，但是这些存储器都通过总线和CPU相连，且CPU都通过控制总线向它们发送读写控制命令。所以CPU把它们都当作内存来对待：将它们组成一个大的逻辑存储器，对其划分存储单元后，就是**内存地址空间**。

![](/img/assembly/1590652270685.png)

虽然各物理存储器组成了大的逻辑存储器，但**每个物理存储器都对应这个大的逻辑存储器中的一个地址段**。CPU在某段地址空间读写数据，实际上就是在对应的物理存储器上读写数据。注意，如果CPU是对ROM存储器对应的地址空间段写数据，输入操作将无效，因为ROM是只读不可写的。

CPU的寻址能力由地址总线的宽度决定，所以CPU能寻址的内存地址空间的大小也是由总线宽度决定的。

汇编语言，面对的就是内存地址空间。在进行编程的时候，必须要知道内存地址空间的分配情况。比如想要在某类存储器中读写数据，必须要知道它的第一个存储单元的地址和最后一个存储单元的地址，才能保证读写操作是在预期的存储器中进行的。

不同计算机的内存地址空间分配情况不一样，8086 PC机的内存地址空间分配情况如下：

![](/img/assembly/1590653151178.png)

所以，CPU向A0000-BFFFF的内存单元写数据，表示向显存中写数据，向C0000-FFFFF的内存单元写数据是无效的，因为它们对应的是只读的ROM。