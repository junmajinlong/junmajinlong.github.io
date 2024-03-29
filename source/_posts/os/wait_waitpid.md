---
title: 操作系统修炼秘籍（29）：Linux进程退出和wait/waitpid
p: os/wait_waitpid.md
date: 2020-04-12 12:13:41
tags: OS
categories: OS
---

  
-----------

**[点我查看操作系统秘籍连载](https://www.junmajinlong.com/os/index/)**

-----------


# 进程退出和wait/waitpid

当进程执行完成后，进程会退出，但是进程退出并不代表着进程真的消失了，内核的进程表中仍然记录着该进程的进程表项。之所以不直接退出子进程，是为了让父进程可以去获取该子进程的退出信息，从而更好地进行多进程的管理和控制。

其实，当某进程执行完所有流程后，它会设置一个2字节的退出状态信息，其中就包含了我们在shell下经常查看的退出状态码`$?`的值。这个退出状态信息非常重要，它只有父进程通过wait()或waitpid()系统调用才能够读取。只有读取了这个退出状态信息后，才意味着父进程已经获取到了子进程的信息，也就是已经完成了子进程的控制，于是内核才会清理掉进程表中的该进程表项，此时子进程才完全退出消失。

![](/img/os/1586671715508.png)

当调度到父进程时，父进程接收这个SIGCHLD信号，虽然这个信号表明了有子进程退出，但是父进程并不一定会根据这个信号做出处理，因为默认情况下这个信号是被忽略的，除非手动定义了SIGCHLD信号的信号处理程序，否则父进程还是不知道子进程退出。那么父进程什么时候才知道要去读取子进程的退出状态信息呢？

其实非常简单。前面说过，wait()和waitpid()是用来读取子进程状态信息的，而这两个系统调用默认是阻塞的（当然，waitpid可以设置为非阻塞方式），当父进程的代码中编写了这两个函数之一时，父进程会在调用到它们的时候一直阻塞等待，直到有子进程退出，wait/waitpid直接就去读取子进程的退出状态信息，相当于wait/waitpid一直在监视着是否有子进程退出。

仍然可以用一段shell的伪代码来描述：

```shell
pid=$(fork)  # 从此开始，有两个进程分支

if [ $pid -eq 0 ];then
    echo "Child Process"   # 子进程
    sleep 3      # 子进程睡眠3秒，3秒后退出
    exit
fi

# 只有父进程会执行下面的代码
echo "Parent Process"
wait    # 父进程调用到wait时将一直阻塞(3秒)，直到子进程退出，然后会读取子进程退出状态信息
echo "Child Terminated"
```

上面的描述虽然稍显复杂，但整个过程其实非常简单，如图：

![](/img/os/1586671459824.png)

另外，既然子进程退出时，内核会发送SIGCHLD信号给父进程，更好的方式自然是在父进程中定义SIGCHLD的信号处理程序，并将wait或waitpid放入这个信号处理程序中去，这样的话，父进程就不需要一开始就阻塞在wait或waitpid上，它可以继续执行其它任务，但只要父进程接收到SIGCHLD信号，就立即触发SIGCHLD处理程序，于是调用该处理程序内部的wait/waitpid去读取子进程的退出状态信息。

