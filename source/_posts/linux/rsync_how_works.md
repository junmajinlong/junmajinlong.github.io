---
title: rsync系列(5)：翻译rsync工作机制(How Rsync Works)
p: linux/rsync_how_works.md
date: 2023-07-16 18:20:34
tags: Linux
categories: Linux
---

--------

**[回到Linux基础(rsync)系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# rsync系列(5)：翻译rsync工作机制(How Rsync Works)

只翻译了重要的部分，没有翻译的保留英文。

```
# How Rsync Works：A Practical Overview

## Foreword

The original [Rsync technical report](http://rsync.samba.org/tech_report/) and Andrew Tridgell's [Phd thesis (pdf)](http://samba.org/~tridge/phd_thesis.pdf) Are both excellent documents for understanding the theoretical mathematics and some of the mechanics of the rsync algorithm. Unfortunately they are more about the theory than the implementation of the rsync utility (hereafter referred to as Rsync).

In this document I hope to describe...

- A      non-mathematical overview of the rsync algorithm.
- How that      algorithm is implemented in the rsync utility.
- The      protocol, in general terms, used by the rsync utility.
- The      identifiable roles the rsync processes play.

This document be able to serve as a guide for programmers needing something of an entré into the source code but the primary purpose is to give the reader a foundation from which he may understand

- Why rsync      behaves as it does.
- The      limitations of rsync.
- Why a      requested feature is unsuited to the code-base.

This document describes in general terms the construction and behaviour of Rsync. In some cases details and exceptions that would contribute to specific accuracy have been sacrificed for the sake meeting the broader goals.
```


## Processes and Roles

当我们讨论rsync时，我们使用了一些特殊的术语来代表不同的进程以及它们在任务执行过程中所扮演的角色。人类为了更方便、更准确地交流，使用同一种语言是非常重要的；同样地，在特定的上下文环境中，使用固定的术语来描述相同的事情也是非常重要的。在Rsync邮件列表中，经常会有一些人对role和processes产生疑惑。出于这些原因，我将定义一些在未来会使用的关于role和process的术语。

| client       | role                      | client(客户端)会启动同步进程。                               |
| ------------ | ------------------------- | ------------------------------------------------------------ |
| server       | role                      | client本地传输时，或通过远程shell、网络套接字连接的对象，它可以是远程rsync进程，也可以表示远程的系统。   server只是一个通用术语，请不要与daemon相混淆。   当client和server建立连接之后，将使用sender和receiver这两个role来代替区分它们。 |
| daemon       | role and process          | 一个等待从client连接的rsync进程。在某些特定平台下，常称之为service。 |
| remote shell | role and set of processes | 为Rsync client和远程rsync server之间提供连接的一个或多个进程。 |
| sender       | role and process          | 一个会访问将被同步的源文件的进程。                           |
| receiver     | role and process          | 当receiver是一个目标系统时将作为一个role，当receiver是一个更新数据并写入磁盘的进程时将作为一个process。 |
| generator    | process                   | generator进程识别出文件变化的部分并管理文件级的逻辑。        |

## Process Startup

当Rsync client启动时，将首先和server端建立一个连接，这个连接的两端可以通过管道，也可以通过网络套接字进行通信。

当Rsync和远程非daemon模式的server通过远程shell通信时，进程的启动方法是fork远程shell，它会通过此方法在远程系统上启动一个Rsync server端进程。Rsync客户端和服务端都通过远程shell间的管道进行通信。此过程中，rsync进程未涉及到网络。在这种模式下，服务端的rsync进程的选项是由远程shell传递的。

当rsync与rsync daemon通信时，它直接使用网络套接字进行通信。这是唯一一种可以称为网络感知的rsync通信方式。这种模式下，rsync的选项必须通过套接字发送，具体内容下文描述。

在客户端和服务端通信最初，双方都会发送最大的协议版本号给对方，双方都会使用较小版本的协议来进行传输。如果是daemon模式的连接，rsync的选项将从客户端发送到服务端，然后再传输exclude列表，从这一刻开始，客户端和服务端的关系仅与错误和日志消息传递有关。(译者注：即从此时开始，将采用sender和receiver这两个角色来描述rsync连接的两端)

本地Rsync任务(源和目标都在本地文件系统)的处理方式类似于push。客户端(译者注：此时即源文件端)变为sender，并fork一个server进程以履行receiver角色的职责，然后client/sender与server/receiver之间通过管道进行通信。

## The File List

file list不仅包含了路径名，还包含了拷贝模式、所有者、权限、文件大小、mtime等属性。如果使用了"--checksum"选项，则还包括文件级的校验码。

rsync连接建立完成的第一件事是sender创建它的file list，当file list创建完成后，其内的每一项都会传递(共享)到receiver端。

当这件事完成后，两端都会按照相对于基目录(base directory)的路径对file list排序(排序算法依赖于传输的协议版本号)，当排序完成后，以后对所有文件的引用都通过file list中的索引来查找。

当receiver接收到file list后，会fork出generator进程，它和receiver进程一起完成pipeline。

## The Pipeline

rsync是高度流水线化的(pipelined)。这意味着进程之间以单方向的方式进行通信。当file list已经传输完毕，pipeline的行为如下：

```
generator --> sender --> receiver
```

generator的输出结果是sender的输入，sender的输出结果是receiver的输入。它们每个进程独立运行，且只有在pipeline被阻塞或等待磁盘IO、CPU资源时才被延迟。

(译者注：虽然它们是单方向的，但每个进程在处理完相关工作的那一刻都会立即将数据传输给它的接收进程，并开始处理下一个工作，接收进程接收到数据后也开始处理这段数据，所以它们虽然是流水线式的工作方式，但它们是独立、并行工作的，基本上不会出现延迟和阻塞)

## The Generator

generator进程将file list与本地目录树进行比较。如果指定了"--delete"选项，则在generator主功能开始前，它将首先识别出不在sender端的本地的文件(译者注：因为此generator为receiver端的进程)，并在recevier端删除这些文件。

然后generator将开始它的主要工作，它会从file list中一个文件一个文件地向前处理。每个文件都会被检测以确定它是否需要跳过。如果文件的mtime或大小不同，最常见的文件操作模式不会忽略它。如果指定了"--checksum"选项，则会生成文件级别的checksum并做比较。目录、块设备和符号链接都不会被忽略。缺失的目录在目标上也会被创建。

如果文件不被忽略，所有目标路径下已存在的文件版本将作为基准文件(basis file)(译者注：请记住这个词，它贯穿整个rsync工作机制)，这些基准文件将作为数据匹配源，使得sender端可以不用发送能匹配上这些数据源的部分(译者注：从而实现增量传输)。为了实现这种远程数据匹配，将会为basis file创建块校验码(block checksum)，并放在文件索引号(文件id)之后立即发送给sender端。如果指定了"--whole-file"选项，则对于新文件(即sender有而receiver没有的文件)，将会发送空的块校验码。(译者注：也就是说，generator每计算出一个文件的块校验码集合，就立即发送给sender，而不是将所有文件的块校验码都计算完成后才一次性发送)

每个文件被分割成的块的大小以及块校验和的大小是根据文件大小计算出来的(译者注：rsync命令支持手动指定block size)。

## The Sender

Sender进程读取来自generator的数据，每次读取一个文件的id号以及该文件的块校验码集合(译者注：或称为校验码列表)。

对于generator发送的每个文件，sender会存储块校验码并生成它们的hash索引以加快查找速度。

然后读取本地文件，并为从第一个字节开始的数据块生成checksum。然后查找generator发送的校验码集合，看该checksum是否能匹配集合中的某项，如果没有匹配项，则无匹配的字节将作为附加属性附加在无匹配数据块上(译者注：此处，无匹配字节即表示第一个字节，它表示无匹配数据块的偏移量，标识无匹配数据块是从哪里开始的)，然后从下一个字节(即第二个字节)开始继续生成校验码并进行比较匹配，直到所有的数据块都匹配完成。这种实现方式就是所谓的滚动校验"rolling checksum"。

如果源文件的块校验码能匹配上校验码集合中的某项，则认为该数据块是匹配块，然后所有累积下来的非文件数据(译者注：如数据块重组指令、文件id等)将随同receiver端对应文件的匹配数据块的偏移量和长度一同发送给receiver端(译者注：例如匹配块对应的是receiver端此文件的第8个数据块，则发送偏移量、匹配的块号和数据块的长度值，虽然数据块的大小都是固定的，但是由于文件分割为固定大小的数据块的时候，最后一个数据块的大小可能小于固定大小的值，因此为了保证长度完全匹配，还需要发送数据块的长度值)，然后generator进程将滚动到匹配块的下一个字节继续计算校验码并比较匹配(译者注：此处能匹配数据块，滚动的大小是一个数据块，对于匹配不上的数据块，滚动的大小是一个字节)。

通过这种方式，即使两端文件的数据块顺序或偏移量不同，也可以识别出所有能匹配的数据块。在rsync算法中，这个处理过程是非常核心的。

使用这种方式，sender将发送一些指令给receiver端，这些指令告诉receiver如何将源文件重组为一个新的目标文件。并且这些指令详细说明了在重组新目标文件时，所有可从basis file中直接复制的匹配数据块(当然，前提是它们在receiver端已存在)，还包含了所有receiver端不存在的裸数据(注：即纯数据)。在每个文件处理的最后阶段，还会发送一个whole-file的校验码(译者注：此为文件级的校验码)，之后sender将开始处理下一个文件。

生成滚动校验码(rolling checksum)以及从校验码集合中搜索是否能匹配的阶段需要一个不错的CPU。在rsync所有的进程中，sender是最消耗CPU的。

## The Receiver

receiver将读取从sender发送过来的数据，并通过其中的文件索引号来识别每个文件，然后它会打开本地文件(即被称为basis file的文件)并创建一个临时文件。

之后receiver将从sender发送过来的数据中读取无匹配数据块(即纯数据)以及匹配上的数据块的附加信息。如果读取的是无匹配数据块，这些纯数据将写入到临时文件中，如果收到的是一个匹配记录，receiver将查找basis file中该数据块的偏移量，然后拷贝这些匹配的数据块到临时文件中。通过这种方式，临时文件将从头开始组建直到组建完成。

当临时文件组建完成，将生成此临时文件的校验码。最后，会将此校验码与sender发送过来的校验码比较，如果比较发现不能匹配，则删除临时文件，并将在第二阶段重新组建该文件，如果失败了两次，则报告失败。

在临时文件最终完全组建成功后，将设置它的所有者、权限、mtime，然后重命名并替换掉basis file。

在rsync所有的进程中，由于receiver会从basis file中拷贝数据到临时文件，所以它是磁盘消耗最高的进程。由于小文件可能一直处于缓存中，所以可以减轻磁盘IO，但是对于大文件，缓存可能会随着generator已经转移到其他文件而被冲刷掉，并且sender会引起进一步的延迟。由于可能从一个文件中随机读取数据并写入到另一个文件中，如果工作集(working set)比磁盘的缓存大，将可能会发生所谓的seek storm，这会再一次降低性能。

## The Daemon

和很多其他的daemon类似，会为每个连接都fork一个daemon子进程。在启动时，它将解析rsyncd.conf文件，以确定存在哪些模块，并且设置全局选项。

当已定义好的模块接收到一个连接时，daemon将会fork一个子进程来处理该连接。然后该子进程将读取rsyncd.conf文件并设置被请求模块的选项，这可能会chroot到模块路径，还可能会删除进程的setuid/setgid。完成上述过程之后，daemon子进程将和普通的rsync server一样，扮演的角色可能是sender也可能是receiver。

## The Rsync Protocol

一个设计良好的通信协议会有一系列的特点。

(1). 所有要发送的东西明确定义在数据包中，包括首部，可选的body或数据负载量。

(2). 每个数据包的首部指定了协议类型或指定了命令行。

(3). 每个数据包的长度都是明确的。

除了这些特点之外，协议还应该具有不同程度的状态、数据包之间的独立性、人类可读性以及重建断开连接的会话的能力。

rsync的协议不包括上述任何特性。数据通过不间断的字节流进行传输。除了非匹配的数据外，既没有指定长度说明符，也没有长度计数器。相反，每个字节的含义取决于由协议层次定义的上下文环境。

例如，当sender正在发送file list，它仅只是简单地发送每个file list中的条目，并且使用一个空字节表示终止整个列表。generator以相同的方式发送文件号以及块校验码集合。

在可靠连接中，这种通信方式能非常好地工作，它比正式的协议工作方式具有更少的数据开销。但很不幸，这也同样使得帮助文档、调试过程变得非常晦涩难懂。每个版本的协议可能都会有细微的差异，因此只能通过了解确切的协议版本来预知发生了什么改变。

## notes

```
This document is a work in progress. The author expects that it has some glaring oversights and some portions that may be more confusing than enlightening for some readers. It is hoped that this could evolve into a useful reference.

Specific suggestions for improvement are welcome, as would be a complete rewrite.
```