## tokio task的通信和同步(3): 同步

`tokio::sync`模块提供了几种状态同步的机制：
- Mutex: 互斥锁  
- RwLock: 读写锁  
- Notify: 通知唤醒机制  
- Barrier: 屏障  
- Semaphore: 信号量  

因为tokio是跨线程执行任务的，因此通常会使用`Arc`来封装这些同步原语，以使其能够跨线程。例如：
```rust
let mutex = Arc::new(Mutex::new());
let rwlock = Arc::new(RwLock::new());
```

### Mutex互斥锁

当多个并发任务(tokio task或线程)可能会修改同一个数据时，就会出现数据竞争现象(竞态)，具体表现为：某个任务对该数据的修改不生效或被覆盖。

互斥锁的作用，就是保护并发情况下可能会出现竞态的代码，这部分代码称为临界区。当某个任务要执行临界区中的代码时，必须先申请锁，申请成功，则可以执行这部分代码，执行完成这部分代码后释放锁。释放锁之前，其它任务无法再申请锁，它们必须等待锁被释放。

假如某个任务一直持有锁，其它任务将一直等待。因此，互斥锁应当尽量快地释放，这样可以提高并发量。

简单介绍完互斥锁之后，再看tokio提供的互斥锁。

`tokio::sync::Mutex`使用new()来创建互斥锁，使用lock()来申请锁，申请锁成功时将返回MutexGuard，并通过drop的方式来释放锁。

例如：
```rust
use std::sync::Arc;
use tokio::{self, sync, runtime::Runtime, time::{self, Duration}};

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let mutex = Arc::new(sync::Mutex::new(0));

        for i in 0..10 {
            let lock = Arc::clone(&mutex);
            tokio::spawn(async move {
                let mut data = lock.lock().await;
                *data += 1;
                println!("task: {}, data: {}", i, data);
            });
        }

        time::sleep(Duration::from_secs(1)).await;
    });
}
```

输出结果：
```
task: 0, data: 1
task: 2, data: 2
task: 3, data: 3
task: 4, data: 4
task: 1, data: 5
task: 7, data: 6
task: 9, data: 7
task: 6, data: 8
task: 5, data: 9
task: 8, data: 10
```

可以看到，任务的调度顺序是随机的，但是数据加1的操作是依次完成的。

需特别说明的是，`tokio::sync::Mutex`其内部使用了标准库的互斥锁，即`std::sync::Mutex`，而标准库的互斥锁是针对线程的，因此，使用tokio的互斥锁时也会锁住整个线程。此外，`tokio::sync::Mutex`是对标准库的Mutex的封装，性能相对要更差一些。也因此，官方文档中建议，如非必须，应使用标准库的Mutex或性能更高的`parking_lot`提供的互斥锁，而不是tokio的Mutex。

例如，将上面的示例该成标准库的Mutex锁。
```rust
fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let mutex = Arc::new(std::sync::Mutex::new(0));

        for i in 0..10 {
            let lock = mutex.clone();
            tokio::spawn(async move {
                let mut data = lock.lock().unwrap();
                *data += 1;
                println!("task: {}, data: {}", i, data);
            });
        }

        time::sleep(Duration::from_secs(1)).await;
    });
}
```

什么情况下可以选择使用tokio的Mutex？当跨await的时候，可以考虑使用tokio Mutex，因为这时使用标准库的Mutex将编译错误。当然，也有相应的解决方案。

什么是跨await？**每个await都代表一个异步任务，跨await即表示该异步任务中出现了至少一个子任务。而每个异步任务都可能会被tokio内部偷到不同的线程上执行，因此跨await时要求其父任务实现Send Trait，这是因为子任务中可能会引用父任务中的数据**。

例如，下面定义的async函数中使用了标准库的Mutex，且有子任务，这会编译错误：

```rust
use std::sync::{Arc, Mutex, MutexGuard};
use tokio::{self, runtime::Runtime, time::{self, Duration}};

async fn add_1(mutex: &Mutex<u64>) {
    let mut lock = mutex.lock().unwrap();
    *lock += 1;
  
    // 子任务，跨await，且引用了父任务中的数据
    time::sleep(Duration::from_millis(*lock)).await;
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let mutex = Arc::new(Mutex::new(0));

        for i in 0..10 {
            let lock = mutex.clone();
            tokio::spawn(async move {
                add_1(&lock).await;
            });
        }

        time::sleep(Duration::from_secs(1)).await;
    });
}
```

`std::sync::MutexGuard`未实现Send，因此父任务`async move{}`语句块是非Send的，于是编译报错。但如果上面的示例中没有子任务`sleep().await`子任务，则编译无错，因为已经可以明确知道该Mutex所在的任务是在当前线程执行的。

对于上面的错误，可简单地使用`tokio::sync::Mutex`来修复。

```rust
use std::sync::Arc;
use tokio::{ self, runtime::Runtime, sync::{Mutex, MutexGuard}, time::{self, Duration} };

async fn add_1(mutex: &Mutex<u64>) {
    let mut lock = mutex.lock().await;
    *lock += 1;
    time::sleep(Duration::from_millis(*lock)).await;
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let mutex = Arc::new(Mutex::new(0));
        for i in 0..10 {
            let lock = mutex.clone();
            tokio::spawn(async move {
                add_1(&lock).await;
            });
        }

        time::sleep(Duration::from_secs(1)).await;
    });
}
```

前面已经说过，tokio的Mutex性能相对较差一些，因此可以不使用tokio Mutex的情况下，尽量不使用它。对于上面的需求，仍然可以继续使用标准库的Mutex，但需要做一些调整。

例如，可以在子任务await之前，把所有未实现Send的数据都drop掉，保证子任务无法引用父任务中的任何非Send数据。

```rust
use std::sync::{Arc, Mutex, MutexGuard};

async fn add_1(mutex: &Mutex<u64>) {
    {
      let mut lock = mutex.lock().unwrap();
      *lock += 1;
    }
    // 子任务，跨await，不引用父任务中的数据
    time::sleep(Duration::from_millis(10)).await;
}
```

这种方案的主要思想是让子任务和父任务不要出现不安全的数据交叉。如果可以的话，应尽量隔离子任务和非Send数据所在的任务。上面的例子已经实现了这一点，但更好的方式是将子任务`sleep().await`从这个函数中移走。

```rust
use std::sync::{Arc, Mutex};
#[allow(unused_imports)]
use tokio::{ self, runtime::Runtime, sync, time::{self, Duration}};

async fn add_1(mutex: &Mutex<u64>) -> u64 {
    let mut lock = mutex.lock().unwrap();
    *lock += 1;
    *lock
}  // 申请的互斥锁在此被释放

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let mutex = Arc::new(Mutex::new(0));

        for i in 0..100 {
            let lock = mutex.clone();
            tokio::spawn(async move {
                let n = add_1(&lock).await;
                time::sleep(Duration::from_millis(n)).await;
            });
        }

        time::sleep(Duration::from_secs(1)).await;
        println!("data: {}", mutex.lock().unwrap());
    });
}
```

另外注意，标准库的Mutex存在毒锁问题。所谓毒锁，即某个持有互斥锁的线程panic了，那么这个锁有可能永远得不到释放(除非线程panic之前已经释放)，也称为被污染的锁。毒锁问题可能很严重，因为出现毒锁有可能意味着数据将从此开始不再准确，所以多数时候是直接让毒锁的panic向上传播或单独处理。但出现毒锁并不总是危险的，所以标准库也提供了对应的方案。

但tokio Mutex不存在毒锁问题，在持有Mutex的线程panic时，tokio的做法是直接释放锁。

### RwLock读写锁

相比Mutex互斥锁，读写锁区分读操作和写操作，读写锁允许多个读锁共存，但写锁独占。因此，在并发能力上它比Mutex要更好一些。

下面是官方文档中的一个示例：
```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    // 多个读锁共存
    {
        // read()返回RwLockReadGuard
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);  // 对Guard解引用，即可得到其内部的值
        assert_eq!(*r2, 5);
    } // 读锁(r1, r2)在此释放

    // 只允许一个写锁存在
    {
        // write()返回RwLockWriteGuard
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    } // 写锁(w)被释放
}
```

需注意，读写锁有几种不同的设计方式：
- 读锁优先: 只要有读操作申请锁，优先将锁分配给读操作。这种方式可以提供非常好的并发能力，但是大量的读操作可能会长时间阻挡写操作  
- 写锁优先: 只要有写操作申请锁，优先将锁分配给写操作。这种方式可以保证写操作不会被饿死，但会严重影响并发能力  

tokio RwLock实现的是写锁优先，它的具体规则如下：
1. 每次申请锁时都将等待，申请锁的异步任务被切换，CPU交还给调度器  
2. 如果申请的是读锁，并且此时没有写锁存在，则申请成功，对应的任务被唤醒  
3. 如果申请的是读锁，但此时有写锁(包括写锁申请)的存在，那么将等待所有的写锁释放(因为写锁总是优先)  
4. 如果申请的是写锁，如果此时没有读锁的存在，则申请成功  
5. 如果申请的是写锁，但此时有读锁的存在，那么将等待当前正在持有的读锁释放  

注意，RwLock的写锁优先会很容易产生死锁。例如，下面的代码会产生死锁：

```rust
use std::sync::Arc;
use tokio::{self, runtime::Runtime, sync::RwLock, time::{self, Duration}};

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let lock = Arc::new(RwLock::new(0));

        let lock1 = lock.clone();
        tokio::spawn(async move {
            let n = lock1.read().await;

            time::sleep(Duration::from_secs(2)).await;
            let nn = lock1.read().await;
        });

        time::sleep(Duration::from_secs(1)).await;
        let mut wn = lock.write().await;
        *wn = 2;
    });
}
```

上面示例中，按照时间的流程，首先会在子任务中申请读锁，1秒后在当前任务中申请写锁，再1秒后子任务申请读锁。

申请第一把读锁时，因为此时无锁，所以读锁(即变量n)申请成功。1秒后申请写锁时，由于此时读锁n尚未释放，因此写锁申请失败，将等待。再1秒之后，继续在子任务中申请读锁，但是此时有写锁申请存在，因此第二次申请读锁将等待，于是读锁写锁互相等待，死锁出现了。

当要使用写锁时，**如果要避免死锁，一定要保证同一个任务中的任意两次锁申请之间，前面已经无锁，并且写锁尽早释放**。

对于上面的示例，同一个子任务中申请两次读锁，但是第二次申请读锁时，第一把读锁仍未释放，这就产生了死锁的可能。只需在第二次申请读锁前，将第一把读锁释放即可。更完整一点，在写锁写完数据后也手动释放写锁(上面的示例中写完就退出，写锁会自动释放，因此无需手动释放)。

```rust
use std::sync::Arc;
use tokio::{self, runtime::Runtime, sync::RwLock, time::{self, Duration}};

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let lock = Arc::new(RwLock::new(0));

        let lock1 = lock.clone();
        tokio::spawn(async move {
            let n = lock1.read().await;
            drop(n);  // 在申请第二把读锁前，先释放第一把读锁

            time::sleep(Duration::from_secs(2)).await;
            let nn = lock1.read().await;
            drop(nn);
        });

        time::sleep(Duration::from_secs(1)).await;
        let mut wn = lock.write().await;
        *wn = 2;
        drop(wn);
    });
}
```

RwLock还有一些其它的方法，在理解了RwLock申请锁的规则之后，这些方法都很容易理解，可以自行去查看官方手册。

### Notify通知唤醒

Notify提供了一种简单的通知唤醒功能，它类似于只有一个信号灯的信号量。

下面是官方文档中的示例：
```rust
use tokio::sync::Notify;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    tokio::spawn(async move {
        notify2.notified().await;
        println!("received notification");
    });

    println!("sending notification");
    notify.notify_one();
}
```

`Notify::new()`创建Notify实例，Notify实例初始时没有permit位，permit可认为是执行权。

每当调用`notified().await`时，将判断此时是否有执行权，如果有，则可直接执行，否则将进入等待。因此，初始化之后立即调用`notified().await`将会等待。

每当调用`notify_one()`时，将产生一个执行权，但多次调用也最多只有一个执行权。因此，调用`notify_one()`之后再调用`notified().await`则并无需等待。

如果同时有多个等待执行权的等候者，释放一个执行权，在其它环境中可能会产生惊群现象，即大量等候者被一次性同时唤醒去争抢一个资源，抢到的可以继续执行，而未抢到的等候者又重新被阻塞。好在，tokio Notify没有这种问题，tokio使用队列方式让等候者进行排队，先等待的总是先获取到执行权，因此不会一次性唤醒所有等候者，而是只唤醒队列头部的那个等候者。

Notify还有一个`notify_waiters()`方法，它不会释放执行权，但是它会一次性唤醒所有正在等待的等候者。严格来说，是让当前已经注册的等候者(即已经调用`notified()`，但是还未await)在下次等待的时候，可以直接通过。

官方手册给了一个示例：
```rust
use tokio::sync::Notify;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    // 注册两个等候者
    let notified1 = notify.notified();
    let notified2 = notify.notified();

    let handle = tokio::spawn(async move {
        println!("sending notifications");
        notify2.notify_waiters();
    });

    // 两个等候者的await都会直接通过
    notified1.await;
    notified2.await;
    println!("received notifications");
}
```

### Barrier屏障

Barrier是一种让多个并发任务在某种程度上保持进度同步的手段。

例如，一个任务分两步，有很多个这种任务并发执行，但每个任务中的第二步都要求所有任务的第一步已经完成。这时可以在第二步之前使用屏障，这样可以保证所有任务在开始第二步之前的进度是同步的。

当然，也不一定要等待所有任务的进度都同步，可以设置等待一部分任务的进度同步。也就是说，让并发任务的进度按批次进行同步。第一批的任务进度都同步后，这一批任务将通过屏障，但是该屏障依然会阻挡下一批任务，直到下一批任务的进度都同步之后才放行。

官方文档给了一个示例，不算经典，但有助于理解：
```rust
use tokio::sync::Barrier;
use std::sync::Arc;

let mut handles = Vec::with_capacity(10);

// 参数10表示屏障宽度为10，只等待10个任务达到屏障点就放行这一批任务
// 也就是说，某时刻已经有9个任务在等待，当第10个任务调用wait的时候，屏障将放行这一批
let barrier = Arc::new(Barrier::new(10));

for _ in 0..10 {
    let c = barrier.clone();
    handles.push(tokio::spawn(async move {
        println!("before wait");

        // 在此设置屏障，保证10个任务都已输出before wait才继续向下执行
        let wait_result = c.wait().await;
        println!("after wait");
        wait_result
    }));
}

let mut num_leaders = 0;
for handle in handles {
    let wait_result = handle.await.unwrap();
    if wait_result.is_leader() {
        num_leaders += 1;
    }
}

assert_eq!(num_leaders, 1);
```

Barrier调用`wait()`方法时，返回`BarrierWaitResult`，该结构有一个`is_leader()`方法，可以用来判断某个任务是否是该批次任务中的第一个任务。每一批通过屏障的任务都只有一个leader，其余非leader任务调用`is_leader()`都将返回false。

使用屏障时，一定要保证可以到达屏障点的并发任务数量是屏障宽度的整数倍，否则多出来的任务将一直等待。例如，将屏障的宽度设置为10(即10个任务一批)，但是有15个并发任务，多出来的5个任务无法凑成完整的一批，这5个任务将一直等待。

```rust
use std::sync::Arc;
use tokio::sync::Barrier;
use tokio::{ self, runtime::Runtime, time::{self, Duration} };

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let barrier = Arc::new(Barrier::new(10));

        for i in 1..=15 {
            let b = barrier.clone();
            tokio::spawn(async move {
                println!("data before: {}", i);

                b.wait().await; // 15个任务中，多出5个任务将一直在此等待
                time::sleep(Duration::from_millis(10)).await;
                println!("data after: {}", i);
            });
        }
        time::sleep(Duration::from_secs(5)).await;
    });
}
```

### Semaphore信号量

信号量可以保证在某一时刻最多运行指定数量的并发任务。

使用信号量时，需在初始化时指定信号灯(tokio中的SemaphorePermit)的数量，每当任务要执行时，将从中取走一个信号灯，当任务完成时(信号灯被drop)会归还信号灯。当某个任务要执行时，如果此时信号灯数量为0，则该任务将等待，直到有信号灯被归还。因此，信号量通常用来提供类似于限量的功能。

例如，信号灯数量为1，表示所有并发任务必须串行运行，这种模式和互斥锁是类似的。再例如，信号灯数量设置为2，表示最多只有两个任务可以并发执行，如果有第三个任务，则必须等前两个任务中的某一个先完成。

例如：

```rust
use chrono::Local;
use std::sync::Arc;
use tokio::{ self, runtime::Runtime, sync::Semaphore, time::{self, Duration}};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        // 只有3个信号灯的信号量
        let semaphore = Arc::new(Semaphore::new(3));

        // 5个并发任务，每个任务执行前都先获取信号灯
        // 因此，同一时刻最多只有3个任务进行并发
        for i in 1..=5 {
            let semaphore = semaphore.clone();
            tokio::spawn(async move {
                let _permit = semaphore.acquire().await.unwrap();
                println!("{}, {}", i, now());
                time::sleep(Duration::from_secs(1)).await;
            });
        }

        time::sleep(Duration::from_secs(3)).await;
    });
}
```

输出结果：
```
3, 2021-11-17 17:06:38
1, 2021-11-17 17:06:38
2, 2021-11-17 17:06:38
4, 2021-11-17 17:06:39
5, 2021-11-17 17:06:39
```

`tokio::sync::Semaphore`提供了以下一些方法:  
- close(): 关闭信号量，关闭信号量时，将唤醒所有的信号灯等待者  
- is_closed(): 检查信号量是否已经被关闭  
- acquire(): 获取一个信号灯，如果信号量已经被关闭，则返回错误AcquireError  
- acquire_many(): 获取指定数量的信号灯，如果信号灯数量不够则等待，如果信号量已经被关闭，则返回AcquireError  
- add_permits(): 向信号量中额外添加N个信号灯  
- available_permits(): 当前信号量中剩余的信号灯数量  
- try_acquire(): 不等待地尝试获取一个信号灯，如果信号量已经关闭，则返回TryAcquireError::Closed，如果目前信号灯数量为0，则返回TryAcquireError::NoPermits  
- try_acquire_many(): 尝试获取指定数量的信号灯  
- acquire_owned(): 获取一个信号灯并消费掉信号量  
- acquire_many_owned(): 获取指定数量的信号灯并消费掉信号量  
- try_acquire_owned(): 尝试获取信号灯并消费掉信号量  
- try_acquire_many_owned(): 尝试获取指定数量的信号灯并消费掉信号量  

对于获取到的信号灯SemaphorePermit，有一个`forget()`方法，该方法可以将信号灯不归还给信号量，因此信号量中的信号灯将永久性地减少(当然，可使用`add_permits()`添加额外的信号灯)。

信号量的限量功能，也可以通过`sync::mpsc`通道来实现。大致逻辑为：设置通道宽度为允许的最大并发任务数量，并先填满通道，当执行一个任务时，先从通道取走一个消息，再执行任务，每次执行完任务后都重新向通道中回补一个消息。

## parking_lot

虽然tokio自身提供了Mutex、RwLock等同步原语，但它们的性能并不优秀，如果需要更高效的同步原语，可考虑使用`parking_lot`提供的同步原语。虽然`parking_lot`的同步原语在等待时是阻塞整个线程的，但不得不提出的是，如果持有锁的时间非常短，那么阻塞线程的时间非常短，这种情况下对异步调度的影响并不大。实际上前文也说了，tokio所提供的Mutex本身就是阻塞线程的。