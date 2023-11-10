---
title: Perl和OS交互(二)：fork
p: perl/perl_fork.md
date: 2019-07-07 17:38:40
tags: Perl
categories: Perl
---

# Perl和OS交互(二)：fork

## fork + exec

fork是低层次的系统调用，通过复制父进程来创建子进程。

### fork的行为

fork用来拷贝当前进程，生成一个基本完全一样的子进程。
```
my $pid=fork();
```

如果fork成功：  
- 则表示成功创建子进程，这时会有两条执行路线：继续执行父进程、执行子进程  
- **fork成功时，会返回两个值：对父进程返回子进程的pid，对子进程返回0**  

如果fork失败，将对父进程返回undef，并设置错误信息。fork失败的可能原因有：  
- 内核内存不够，无法申请内存来fork  
- 达到了允许的最大进程数量(进程数上限)  
- 达到了rlimit限制的某种资源上限


例如：
```
#!/usr/bin/perl
#
use 5.010;

my $pid=fork();
say $pid,"======";
```

执行该程序，将返回两行数据：
```
62620======
0======
```

其中第一行输出是父进程输出的，第二行是子进程输出的。

虽然这里父进程先输出，**但fork成功之后，父、子进程并没有执行的先后顺序**，也可能cpu会先调度到上子进程去执行。

![](/img/perl/733013-20180923181459760-1882305440.png)

注意上图中子进程部分只画了"say"那行语句，但实际上子进程是完全复制父进程的，子进程也可以有say前面的那段语句(比如那个fork语句)，但由于父进程的状态已经执行完了fork，所以子进程也是从fork语句之后开始执行的，fork语句之前的语句对于子进程来说是透明的。而且按照写时复制的技术，子进程用不到它所以不会复制fork前面的代码(注：也就是说子进程和父进程是共享代码的)。

更复杂一点的例子：
```
#!/usr/bin/perl
#
use 5.010;
print "id1: ",$pid,"\n";
my $pid=fork;
print "id2: ",$pid,"\n";
if(!$pid){  # ==0 exec
    say "child process: ",$pid;
}
waitpid($pid,0);
say "parent process: ",$pid;
```

首先perl进程输出id1。然后fork一个子进程，这时有两条执行路线。假如fork后先执行父进程，则：  
- 此父进程将输出id2  
- 然后判断pid的值，因为fork返回给父进程的的pid变量值为子进程的进程号，所以不会等于0，于是if判断不通过  
- 继续执行waitpid()，它将等待子进程执行结束，以下是子进程的执行过程：  
    - 子进程首先输出id2  
    - 然后判断`$pid`，由于fork返回给子进程的pid变量值为0，所以if判断通过，于是子进程输出`child process`  
    - 继续执行waitpid()，由于`$pid=0`，waitpid()的等待pid为0时，表示等待以自己为leader进程的进程组中的其它进程，由于没有进程组，所以waitpid失败  
    - 继续执行，输出`parent process`  
    - 子进程执行完毕  
- 父进程的waitpid()等待子进程执行完毕，继续向下执行  
- 父进程输出`parent process`  

假如fork之后，先执行子进程，且还先把子进程执行完了，cpu才调度到父进程，则也没有影响，子进程执行完毕后，迟早会调度到父进程，而父进程的`waitpid($pid)`已经没有子进程了，于是waitpid()失败(返回-1)，继续向下执行。

![](/img/perl/733013-20180923173138323-1675337278.png)

显然，上面fork的代码有一些问题。由于fork创建子进程之后。父、子进程都继续执行，且执行的先后顺序不定。所以：  
1. 在fork之后，应该紧接着判断是否是子进程，避免有些在操作父子中都执行  
2. 在父进程中等待子进程  
3. 在子进程中加入执行完后就退出子进程的动作，免得执行本该父进程执行的动作  

大概代码如下：
```
my $pid=fork;
unless($pid){   # 判断子进程的语句紧跟着fork
    CODE1;
    exit;       # 要让子进程退出
}
waitpid($pid,0);  # 要等待子进程
CODE2;
```

### fork和exec结合

一般fork和exec会一起用，fork用来创建新的子进程，exec启动一个程序替代当前子进程并在子进程结束时退出子进程。

例如`system "date"`命令，替换为低层次的fork+exec+waitpid。

```
defined(my $pid=fork) or die "Cannot fork: $!";
unless($pid){
    # 进入到这里执行的，表示进入子进程
    exec 'date';       # exec正确执行时，执行完后将结束子进程
    die "cannot exec date: $!";     # exec启动失败时，将执行die来结束子进程
}

# 这里的表示是父进程
waitpid($pid,0);
```
