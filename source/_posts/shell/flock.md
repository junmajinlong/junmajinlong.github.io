---
title: 理解和使用文件锁flock命令
p: shell/flock.md
date: 2024-02-23 07:37:29
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# 理解和使用文件锁flock命令

在写Shell脚本的时候，经常遇到需要加锁的场景。比如定时任务执行的Shell脚本在上一次触发后仍在执行中又被触发了，如何避免重复执行？对这个问题可以判断Shell脚本进程是否存在，仍然存在则退出不执行本次任务，这个问题也可以通过加锁方案来解决。

## 读写锁的共存性

读锁(或共享锁)与读锁之间可以同时存在，互斥锁(写锁、排他锁)只能独立存在。

## Shell中实现锁的几种方式

在Shell脚本中，加锁和锁判断的方式有几种实现方式：

- 判断某个文件或目录是否存在。
  - 如果文件或目录不存在，则认为当前不存在锁，可以立即创建文件或目录表示已经加锁，并在完成任务时删除立即锁文件表示释放锁。
  - 如果文件或目录已经存在，则认为当前有其它任务正在执行并且还未释放锁，可以返回、退出、或循环sleep等待锁被释放。
  - 这种方式只能实现简单的互斥锁。
- 判断文件中的内容或者目录下的文件名。
  - 比如向某个文件中写入1表示加互斥锁，写入0表示加共享锁，文件不存在则认为锁不存在。
  - 比如在某个目录中创建文件名为write的文件表示加互斥锁，创建文件名为read的文件表示加共享锁，目录下这两个文件都不存在则认为锁不存在。
  - 这种方式除了实现读写锁，还可以实现更多复杂状态的判断，很灵活。
- 使用flock命令。

但以上几种方案都是很弱的加锁方式，是劝告式的锁，劝告参与者都遵守锁规则。即，参与者必须总是遵守加锁、判断锁、释放锁的规则才能产生锁的效果。

## 判断文件存在性作为锁

以判断文件存在性作为锁规则为例：

```shell
#!/bin/bash 
# 本脚本以/tmp/this.lock作为锁文件

LOCK_FILE="/tmp/this.lock"

# 判断锁是否存在，
# - 存在则退出脚本(表示有其他同类任务或者本脚本上次调用后还未执行完任务)，
# - 不存在则创建文件表示加锁，任务完成时删除文件表示释放锁。

# 为了确保在各种情况下锁文件都能被删除，使用trap捕获EXIT信号，只要脚本退出，就删除锁文件
trap 'rm -f $LOCK_FILE' EXIT

# 锁文件如果已经存在，则退出脚本
# 如果要等待锁释放，则参考如下方式循环等待，还可以实现等待锁释放的超时时间
# while [ -e "$LOCK_FILE" ];do sleep 0.5; done
if [ -e "$LOCK_FILE" ]; then
  echo "task is doing"
  exit 1
fi

# 立即加锁
touch "$LOCK_FILE"

# 加锁之后，做其他任务
sleep 10
```

## 使用flock命令加锁

flock命令可以让我们更方便地使用文件(或目录)锁。

```
❯ flock --help

Usage:
 flock [options] <file>|<directory> <command> [<argument>...]
 flock [options] <file>|<directory> -c <command>
 flock [options] <file descriptor number>

Manage file locks from shell scripts.

Options:
 -s, --shared             尝试获取共享锁，默认会等待直到能够获取到锁
 -x, --exclusive          尝试获取独占锁(默认锁)，默认会等待直到能够获取到锁
 -u, --unlock             释放锁
 -n, --nonblock           非阻塞获取锁，如果无法立即获取锁，则立即退出而不是一直等待
 -w, --timeout <secs>     等待锁时，最多等待指定的秒数，等待超时后退出
 -E, --conflict-exit-code <number>  当等待锁超时或者非阻塞获取锁失败时的退出状态码，默认退出状态码为1
 -o, --close              获取锁之后，执行命令之前，关闭锁文件的文件描述符
 -c, --command <command>  执行不带参数的命令
 -F, --no-fork            不fork新的进程来调用要执行的命令，而是调用要被执行的命令并直接替换flock进程
```

例如，以/tmp/a.lock为锁文件，加锁或等待已存在的锁释放，加锁成功后带锁sleep 5秒，sleep进程退出时自动释放锁，然后第二个sleep命令才开始执行。

```shell
$ flock /tmp/a.lock sleep 5

# 在另一个终端执行：将阻塞5秒
$ flock /tmp/a.lock sleep 3
```

使用`lslocks`命令可以查看当前存在的flock锁。

```shell
$ flock /tmp/a.lock sleep 20 &
[1] 992

$ lslocks
COMMAND           PID  TYPE SIZE MODE  M START END PATH
cron              397 FLOCK      WRITE 0     0   0 /run...
dockerd           577 FLOCK      WRITE 0     0   0 /...
dockerd           577 FLOCK      WRITE 0     0   0 /...
dockerd           577 FLOCK      WRITE 0     0   0 /...
dockerd           577 FLOCK      WRITE 0     0   0 /...
dockerd           577 FLOCK      WRITE 0     0   0 /...
dockerd           577 FLOCK      WRITE 0     0   0 /...
# 存在/tmp/a.lock上的锁，且是写锁
flock             992 FLOCK      WRITE 0     0   0 /tmp/a.lock
containerd        414 FLOCK      WRITE 0     0   0 /...
```

默认情况下，获取的是互斥锁，如果要获取共享锁，使用`-s`选项：

```shell
# 两个sleep命令可以同时运行，但echo命令将在等待5秒后才被执行
flock -s /tmp/a.lock sleep 3 &
flock -s /tmp/a.lock sleep 5 &
flock -x /tmp/a.lock echo hello world
```

可以通过`-w`选项指定等待锁的超时时间，如果等待超时，即获取锁失败，其默认退出状态码为1，通过`-E`选项可以指定获取锁失败时的退出状态码。

```shell
# 第二个flock命令将阻塞5秒，然后退出，echo命令不会被执行，且退出状态码为10
flock /tmp/a.lock sleep 10 &
flock -w 5 -E 10 /tmp/a.lock echo hello
echo $?    # 10
```

通过`-n`选项可以非阻塞尝试获取锁，如果无法立即获取锁，则立即退出，退出状态码默认为1，也可通过`-E`选项指定退出状态码。

```shell
flock /tmp/a.lock sleep 10 &
flock -n -E 10 /tmp/a.lock echo hello
echo $?    # 10
```

因为可以非阻塞尝试获取锁，因此可以在while循环中循环判断是否能获取锁：

```shell
while ! flock -n /tmp/a.lock; do 
  sleep 0.5
done
```

上面的示例都是直接在flock命令中指定锁文件，也可以给flock命令指定一个打开的文件描述符，多条命令需要带锁执行时这种方式更方便。

```shell
# 打开文件并分配文件描述符9
exec 9<> /tmp/a.lock

# 指定文件描述符加锁，直到/tmp/a.lock的所有文件描述符都关闭，或者手动释放锁才会释放锁
flock 9
sleep 5   # 带锁执行
echo over # 带锁执行
exec 9>&- # 关闭文件描述符9，将释放锁
# 或者手动释放锁: flock -u 9
```

更友好的多条命令带锁执行且自动释放锁的方式：

```shell
(
  echo '这里无锁执行'
  # 下面开始加锁，所有命令都带锁执行
  flock 9 || exit 1
  sleep 1
  sleep 2
) 9>/tmp/a.lock  # 执行完所有命令后，自动释放锁
```

因为flock底层加锁的原因(后文会解释)，如果有带锁执行的后台命令，但这个命令又不需要持有锁的时候，建议加上手动释放锁的操作：

![](/img/Shell/1708658832005.png)

上例中之所以要`flock -u 9`手动释放锁，是因为后台任务是一个新的子进程，fock子进程的时候会复制文件描述符9，使得flock锁也被复制到新的后台子进程中。使用`flock -u 9`将强制释放文件描述符9对应的所有flock锁。


## 理解flock锁的基本原理

flock命令在内核的open file table级别上对文件进行加锁。

![](/img/Shell/1708652452449.png)

简单描述一下fd table和open file table和g-inode table：

- g-inode table：该表中的每条记录唯一对应文件系统上的每一个文件，一个inode对应一个文件  
- open-file table: 每次打开一个文件，都会在该表中添加一条记录  
- fd table：进程级别，每个进程都有一个fd table。每次打开文件，都会同时在该进程的fd table中添加一条记录指向open-file table中对应文件的记录  

简单的总结：

- fd table中可以有多条记录指向open file table中的同一条记录。比如文件描述符复制时，会在fd table中产生多条记录，它们都指向open file table中的同一条记录
- 多个进程的fd table中不同记录也可以同时指向open file table中的同一条记录。比如fork进程的时候，会复制打开的文件描述符，使得不同fd table中的记录指向open file table中的同一条记录
- open file table中的多条记录可以同时指向g-inode table中的同一条记录。比如同时多次打开同一个文件，会在open file table中有多条记录，它们都指向g-inode table中的同一条记录

由于flock锁存在于open file table级别，因此文件描述符的复制操作或者fork子进程的时候也会同时复制flock锁，flock锁和文件描述符类似，也可以认为是引用计数的，只有带flock锁属性的文件描述符以及所复制的所有文件描述符都被关闭，该flock锁才会释放。

![](/img/Shell/1708658943397.png)

下面是一个使用flock锁的案例。

```shell
function kill_process() {
    (
        flock -x 9 || exit 1
        pkill PROCESS_NAME
    ) 9>"$LOCK_PATH"
}

function run_process() {
    (
        flock -x 9 || exit 1

        # 这是一个启动后台服务的函数，后台服务的运行不需要持有锁
        start_process

        flock -u 9  # 这里需要手动释放锁，否则后台服务也会长期持有锁
    ) 9>"$LOCK_PATH"
}

function restart_process() {
    (
        # kill_process函数是带锁执行的，但执行后会自动释放锁
        kill_process "$@"
        
        # kill后，这里重新加锁
        flock -x 9 || exit 1

        # 这是一个启动后台服务的函数，后台服务的运行不需要持有锁
        start_process

        # 这里需要手动释放锁，否则后台服务也会长期持有锁
        flock -u 9
    ) 9>"$LOCK_PATH"
}
```

