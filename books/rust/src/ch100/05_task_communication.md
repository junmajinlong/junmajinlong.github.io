## tokio task的通信和同步(2): 通信

tokio使用通道在task之间进行通信，有四种类型通道：oneshot、mpsc、broadcast和watch。

### oneshot通道

oneshot通道的特性是：单Sender、单Receiver以及单消息，简单来说就是一次性的通道。

oneshot通道的创建方式是使用`oneshot::channel()`方法：

```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>)
```

它返回该通道的写端sender和读端receiver，其中泛型T表示的是读写两端所传递的消息类型。

例如，创建一个可发送i32数据的一次性通道：
```rust
let (tx, rx) = oneshot::channel::<i32>();
```

返回的结果中，tx是发送者(sender)、rx是接收者(receiver)。

多数时候不需要去声明通道的类型，编译器可以根据发送数据时的类型自动推断出类型。

```rust
let (tx, rx) = oneshot::channel();
```

#### Sender

Sender通过`send()`方法发送数据，因为oneshot通道只能发送一次数据，所以`send()`发送数据的时候，tx直接被消费掉。Sender并不一定总能成功发送消息，比如，Sender发送消息之前，Receiver端就已经关闭了读端。因此`send()`返回Result结果：如果发送成功，则返回`Ok(())`，如果发送失败，则返回`Err(T)`。

因此，发送数据的时候，通常会做如下检测：
```rust
// 或 if tx.send(33).is_err() {}
// 或直接忽略错误 let _ = tx.send();
if let Err(_) = tx.send(33) {
  println!("receiver closed");
}
```

另外需注意，`send()`是非异步但却不阻塞的，它总是立即返回，如果能发送数据，则发送数据，如果不能发送数据，就返回错误，它不会等待Receiver启动读取操作。也因此，`send()`可以应用在同步代码中，也可以应用在异步代码中。

Sender可以通过`is_closed()`方法来判断Receiver端是否已经关闭。

Sender可以通过`close()`方法来等待Receiver端关闭。它可以结合`select!`宏使用：其中一个分支计算要发送的数据，另一个分支为`closed()`等待分支，如果先计算完成，则发送计算结果，而如果是先等到了对端closed的异步任务完成，则无需再计算浪费CPU去计算结果。例如：

```rust
tokio::spawn(async move {
  tokio::select! {
    _ = tx.closed() => {
      // 先等待到了对端关闭，不做任何事，select!会自动取消其它分支的任务
    }
    value = compute() => {
      // 先计算得到结果，则发送给对端
      // 但有可能刚计算完成，尚未发送时，对端刚好关闭，因此可能发送失败
      // 此处丢弃发送失败的错误
      let _ = tx.send(value);
    }
  }
});
```

#### Receiver

Receiver没有`recv()`方法，rx本身实现了Future Trait，它执行时对应的异步任务就是接收数据，因此只需await即可用来接收数据。

但是，接收数据并不一定会接收成功。例如，Sender端尚未发送任何数据就已经关闭了(被drop)，此时Receiver端会接收到`error::RecvError`错误。因此，接收数据的时候通常也会进行判断：

```rust
match rx.await {
  Ok(v) => println!("got = {:?}", v),
  Err(_) => println!("the sender dropped"),
  // Err(e: RecvError) => xxx,
}
```

既然通过`rx.await`来接收数据，那么已经隐含了一个信息，异步任务中接收数据时会进行等待。

Receiver端可以通过`close()`方法关闭自己这一端，当然也可以直接drop来关闭。关闭操作是幂等的，即，如果关闭的是已经关闭的Recv，则不产生任何影响。

关闭Recv端之后，可以保证Sender端无法再发送消息。但需要注意，有可能Recv端关闭完成之前，Sender端正好在这时发送了一个数据过来。因此，在关闭Recv端之后，尽可能地再调用一下`try_recv()`方法尝试接收一次数据。

`try_recv()`方法返回三种可能值：  
- `Ok(T)`: 表示成功接收到通道中的数据  
- `Err(TryRecvError::Empty)`: 表示通道为空
- `Err(TryRecvError::Closed)`: 表示通道为空，且Sender端已关闭，即Sender未发送任何数据就关闭了  

例如：

```rust
let (tx, mut rx) = oneshot::channel::<()>();

drop(tx);

match rx.try_recv() {
    // The channel will never receive a value.
    Err(TryRecvError::Closed) => {}
    _ => unreachable!(),
}
```

#### 使用示例

一个完整但简单的示例：
```rust
use tokio::{self, runtime::Runtime, sync};

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let (tx, rx) = sync::oneshot::channel();

        tokio::spawn(async move {
            if tx.send(33).is_err() {
                println!("receiver dropped");
            }
        });

        match rx.await {
            Ok(value) => println!("received: {:?}", value),
            Err(_) => println!("sender dropped"),
        };
    });
}
```

另一个比较常见的使用场景是结合`select!`宏，此时应在recv前面加上`&mut`。例如：
```rust
let interval = tokio::interval(tokio::time::Duration::from_millis(100));

// 注意mut
let (tx, mut rx) = oneshot::channel();
loop {
    // 注意，select!中无需await，因为select!会自动轮询推进每一个分支的任务进度
    tokio::select! {
        _ = interval.tick() => println!("Another 100ms"),
        msg = &mut recv => {
            println!("Got message: {}", msg.unwrap());
            break;
        }
    }
}
```

### mpsc通道

mpsc通道的特性是可以有多个发送者发送多个消息，且只有一个接收者。mpsc通道是使用最频繁的通道类型。

mpsc通道分为两种：  
- bounded channel: 有界通道，通道有容量限制，即通道中最多可以存放指定数量(至少为1)的消息，通过`mpsc::channel()`创建  
- unbounded channel: 无界通道，通道中可以无限存放消息，直到内存耗尽，通过`mpsc::unbounded_channel()`创建  

#### 有界通道

通过`mpsc::channel()`创建有界通道，需传递一个大于1的usize值作为其参数。

例如，创建一个最多可以存放100个消息的有界通道。
```rust
// tx是Sender端，rx是Receiver端
// 接收端接收数据时需修改状态，因此声明为mut
let (tx, mut rx) = mpsc::channel(100);
```

mpsc通道只能有一个Receiver端，但可以`tx.clone()`得到多个Sender端，clone得到的Sender都可以使用`send()`方法向该通道发送消息。

发送消息时，如果通道已满，发送消息的任务将等待直到通道中有空闲的位置。

发送消息时，如果Receiver端已经关闭，则发送消息的操作将返回`SendError`。

如果所有的Sender端都已经关闭，则Receiver端接收消息的方法`recv()`将返回None。

一个简单的示例：
```rust
use tokio::{ self, runtime::Runtime, sync };

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let (tx, mut rx) = sync::mpsc::channel::<i32>(10);

        tokio::spawn(async move {
            for i in 1..=10 {
                // if let Err(_) = tx.send(i).await {}
                if tx.send(i).await.is_err() {
                    println!("receiver closed");
                    return;
                }
            }
        });

        while let Some(i) = rx.recv().await {
            println!("received: {}", i);
        }
    });
}
```

输出的结果：
```
received: 1
received: 2
received: 3
received: 4
received: 5
received: 6
received: 7
received: 8
received: 9
received: 10
```

上面的示例中，先生成了一个异步任务，该异步任务向通道中发送10个数据，Receiver端则在while循环中不断从通道中取数据。

将上面的示例改一下，生成10个异步任务分别发送数据：
```rust
use tokio::{ self, runtime::Runtime, sync };

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let (tx, mut rx) = sync::mpsc::channel::<i32>(10);

        for i in 1..=10 {
            let tx = tx.clone();
            tokio::spawn(async move {
                if tx.send(i).await.is_err() {
                    println!("receiver closed");
                }
            });
        }
        drop(tx);

        while let Some(i) = rx.recv().await {
            println!("received: {}", i);
        }
    });
}
```

输出的结果：
```
received: 2
received: 3
received: 1
received: 4
received: 6
received: 5
received: 10
received: 7
received: 8
received: 9
```

10个异步任务发送消息的顺序是未知的，因此接收到的消息无法保证顺序。

另外注意上面示例中的`drop(tx)`，因为生成的10个异步任务中都拥有clone后的Sender，clone出的Sender在每个异步任务完成时自动被drop，但原始任务中还有一个Sender，如果不关闭这个Sender，`rx.recv()`将不会返回None，而是一直等待。

如果通道已满，Sender通过`send()`发送消息时将等待。例如下面的示例中，通道容量为5，但要发送7个数据，前5个数据会立即发送，发送第6个消息的时候将等待，直到1秒后Receiver开始从通道中消费数据。
```rust
use chrono::Local;
use tokio::{self, sync, runtime::Runtime, time::{self, Duration}};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let (tx, mut rx) = sync::mpsc::channel::<i32>(5);

        tokio::spawn(async move {
            for i in 1..=7 {
              if tx.send(i).await.is_err() {
                println!("receiver closed");
                return;
              }
              println!("sended: {}, {}", i, now());
            }
        });

        time::sleep(Duration::from_secs(1)).await;
        while let Some(i) = rx.recv().await {
            println!("received: {}", i);
        }
    });
}
```

输出结果：
```
sended: 1, 2021-11-12 18:25:28
sended: 2, 2021-11-12 18:25:28
sended: 3, 2021-11-12 18:25:28
sended: 4, 2021-11-12 18:25:28
sended: 5, 2021-11-12 18:25:28
received: 1
received: 2
received: 3
received: 4
received: 5
sended: 6, 2021-11-12 18:25:29
sended: 7, 2021-11-12 18:25:29
received: 6
sended: 8, 2021-11-12 18:25:29
received: 7
received: 8
received: 9
sended: 9, 2021-11-12 18:25:29
sended: 10, 2021-11-12 18:25:29
received: 10
```

Sender端和Receiver端有一些额外的方法需要解释一下它们的作用。

对于Sender端：  
- capacity(): 获取当前通道的剩余容量(注意，不是初始化容量)  
- closed(): 等待Receiver端关闭，当Receiver端关闭后该等待任务会立即完成  
- is_closed(): 判断Receiver端是否已经关闭  
- send(): 向通道中发送消息，通道已满时会等待通道中的空闲位置，如果对端已关闭，则返回错误  
- send_timeout(): 向通道中发送消息，通道已满时只等待指定的时长  
- try_send(): 向通道中发送消息，但不等待，如果发送不成功，则返回错误  
- reserve(): 等待并申请一个通道中的空闲位置，返回一个Permit，申请的空闲位置被占位，且该位置只留给该Permit实例，之后该Permit可以直接向通道中发送消息，并释放其占位的位置。申请成功时，通道空闲容量减1，释放位置时，通道容量会加1  
- try_reserve(): 尝试申请一个空闲位置且不等待，如果无法申请，则返回错误  
- reserve_owned(): 与reserve()类似，它返回OwnedPermit，但会Move Sender  
- try_reserve_owned(): reserve_owned()的不等待版本，尝试申请空闲位置失败时会立即返回错误  
- blocking_send(): Sender可以在同步代码环境中使用该方法向异步环境发送消息  

对于Receiver端：
- close(): 关闭Receiver端  
- recv(): 接收消息，如果通道已空，则等待，如果对端已全部关闭，则返回None  
- try_recv(): 尝试接收消息，不等待，如果无法接收消息(即通道为空或对端已关闭)，则返回错误  
- blocking_recv(): Receiver可以在同步代码环境中使用该方法接收来自异步环境的消息  

注意，在这些方法中，`try_xxx()`方法都是立即返回不等待的(可以认为是同步代码)，因此调用它们后无需await，只有调用那些可能需要等待的方法，调用后才需要await。例如`rx.recv().await`和`rx.try_recv()`。

下面是一些稍详细的用法说明和示例。

Sender端可通过`send_timeout()`来设置一个等待通道空闲位置的超时时间，它和`send()`返回值一样，此外还添加一种超时错误：超时后仍然没有发送成功时将返回错误。至于返回的是什么错误，对于发送端来说不重要，重要的是发送的消息是否成功。因此，对于Sender端的条件判断，通常也仅仅只是检测`is_err()`：

```rust
if tx.send_timeout(33, Duration::from_secs(1)).await.is_err() {
  println!("receiver closed or timeout");
}
```

**需要特别注意的是，Receiver端调用close()方法关闭通道后，只是半关闭状态，Receiver端仍然可以继续读取可能已经缓冲在通道中的消息，close()只能保证Sender端无法再发送普通的消息，但Permit或OwnedPermit仍然可以向通道发送消息。只有通道已空且所有Sender端(包括Permit和OwnedPermit)都已经关闭的情况下，recv()才会返回None，此时代表通道完全关闭**。

Receiver的`try_recv()`方法在无法立即接收消息时会立即返回错误。返回的错误分为两种:  
- TryRecvError::Empty错误: 表示通道已空，但Sender端尚未全部关闭  
- TryRecvError::Disconnected错误: 表示通道已空，且Sender端(包括Permit和OwnedPermit)已经全部关闭  

关于`reserve()`和`reserve_owned()`，看官方示例即可轻松理解：
```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // 创建容量为1的通道
    let (tx, mut rx) = mpsc::channel(1);
    // 申请并占有唯一的空闲位置
    let permit = tx.reserve().await.unwrap();
    // 唯一的位置已被permit占有，tx.send()无法发送消息
    assert!(tx.try_send(123).is_err());
    // Permit可以通过send()方法向它占有的那个位置发送消息
    permit.send(456);
    // Receiver端接收到消息
    assert_eq!(rx.recv().await.unwrap(), 456);


    // 创建容量为1的通道
    let (tx, mut rx) = mpsc::channel(1);
    // tx.reserve_owned()会消费掉tx
    let permit = tx.reserve_owned().await.unwrap();
    // 通过permit.send()发送消息，它又返回一个Sender
    let tx = permit.send(456);
    assert_eq!(rx.recv().await.unwrap(), 456);
    //可以继续使用返回的Sender发送消息
    tx.send(789).await.unwrap();
}
```

#### 无界通道

理解了mpsc的有界通道之后，再理解无界通道会非常轻松。

```rust
let (tx, mut rx) = mpsc::unbounded_channel();
```

对于无界通道，它的通道中可以缓冲无限数量的消息，直到内存耗尽。这意味着，Sender端可以无需等待地不断向通道中发送消息，这也意味着无界通道的Sender既可以在同步环境中也可以在异步环境中向通道中发送消息。只有当Receiver端已经关闭，Sender端的发送才会返回错误。

使用无界通道的关键，在于必须要保证不会无限度地缓冲消息而导致内存耗尽。例如，让Receiver端消费消息的速度尽量快，或者采用一些复杂的限速机制让严重超前的Sender端等一等。

### broadcast通道

broadcast通道是一种广播通道，可以有多个Sender端以及多个Receiver端，可以发送多个数据，且任何一个Sender发送的每一个数据都能被所有的Receiver端看到。

使用`mpsc::broadcast()`创建广播通道，要求指定一个通道容量作为参数。它返回Sender和Receiver。Sender可以克隆得到多个Sender，可以调用Sender的`subscribe()`方法来创建新的Receiver。

例如，下面是官方文档提供的一个示例：
```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    // 最多存放16个消息
    // tx是Sender，rx1是Receiver
    let (tx, mut rx1) = broadcast::channel(16);

    // Sender的subscribe()方法可生成新的Receiver
    let mut rx2 = tx.subscribe();

    tokio::spawn(async move {
        assert_eq!(rx1.recv().await.unwrap(), 10);
        assert_eq!(rx1.recv().await.unwrap(), 20);
    });

    tokio::spawn(async move {
        assert_eq!(rx2.recv().await.unwrap(), 10);
        assert_eq!(rx2.recv().await.unwrap(), 20);
    });

    tx.send(10).unwrap();
    tx.send(20).unwrap();
}
```

Sender端通过`send()`发送消息的时候，如果所有的Receiver端都已关闭，则`send()`方法返回错误。

Receiver端可通过`recv()`去接收消息，如果所有的Sender端都已经关闭，则该方法返回`RecvError::Closed`错误。该方法还可能返回`RecvError::Lagged`错误，该错误表示接收端已经落后于发送端。

虽然broadcast通道也指定容量，但是通道已满的情况下还可以继续写入新数据而不会等待(因此上面示例中的`send()`无需await)，此时通道中最旧的(头部的)数据将被剔除，并且新数据添加在尾部。就像是FIFO队列一样。出现这种情况时，就意味着接收端已经落后于发送端。

当接收端已经开始落后于发送端时，下一次的`recv()`操作将直接返回`RecvError::Lagged`错误。如果紧跟着再执行`recv()`且落后现象未再次发生，那么这次的`recv()`将取得队列头部的消息。

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    // 通道容量2
    let (tx, mut rx) = broadcast::channel(2);

    // 写入3个数据，将出现接收端落后于发送端的情况，
    // 此时，第一个数据(10)将被剔除，剔除后，20将位于队列的头部
    tx.send(10).unwrap();
    tx.send(20).unwrap();
    tx.send(30).unwrap();

    // 落后于发送端之后的第一次recv()操作，返回RecvError::Lagged错误
    assert!(rx.recv().await.is_err());

    // 之后可正常获取通道中的数据
    assert_eq!(20, rx.recv().await.unwrap());
    assert_eq!(30, rx.recv().await.unwrap());
}
```

Receiver也可以使用`try_recv()`方法去无等待地接收消息，如果Sender都已关闭，则返回`TryRecvError::Closed`错误，如果接收端已落后，则返回`TryRecvError::Lagged`错误，如果通道为空，则返回`TryRecvError::Empty`错误。

另外，`tokio::broadcast`的任何一个Receiver都可以看到每一次发送的消息，且它们都可以去`recv()`同一个消息，`tokio::broadcast`对此的处理方式是消息克隆：每一个Receiver调用`recv()`去接收一个消息的时候，都会克隆通道中的该消息一次，直到所有存活的Receiver都克隆了该消息，该消息才会从通道中被移除，进而释放一个通道空闲位置。

这可能会导致一种现象：某个ReceiverA已经接收了通道中的第10个消息，但另一个ReceiverB可能尚未接收第一个消息，由于第一个消息还未被全部接收者所克隆，它仍会保留在通道中并占用通道的位置，假如该通道的最大容量为10，此时Sender再发送一个消息，那么第一个数据将被踢掉，ReceiverB接收到消息的时候将收到`RecvError::Lagged`错误并永远地错过第一个消息。

### watch通道

watch通道的特性是：只能有单个Sender，可以有多个Receiver，且通道永远只保存一个数据。Sender每次向通道中发送数据时，都会修改通道中的那个数据。

通道中的这个数据可以被Receiver进行引用读取。

一个简单的官方示例：
```rust
use tokio::sync::watch;
#[tokio::main]
async fn main() {
    // 创建watch通道时，需指定一个初始值存放在通道中
    let (tx, mut rx) = watch::channel("hello");

    // Recevier端，通过changed()来等待通道的数据发生变化
    // 通过borrow()引用通道中的数据
    tokio::spawn(async move {
        while rx.changed().await.is_ok() {
            println!("received = {:?}", *rx.borrow());
        }
    });

    // 向通道中发送数据，实际上是修改通道中的那个数据
    tx.send("world")?;
}
```

watch通道的用法很简单，但是有些细节需要理解。

Sender端可通过`subscribe()`创建新的Receiver端。

当所有Receiver端均已关闭时，`send()`方法将返回错误。也就是说，`send()`必须要在有Receiver存活的情况下才能发送数据。

但是Sender端还有一个`send_replace()`方法，它可以在没有Receiver的情况下将数据写入通道，并且该方法会返回通道中原来保存的值。

无论是Sender端还是Receiver端，都可以通过`borrow()`方法取得通道中当前的值。由于可以有多个Receiver，为了避免读写时的数据不一致，watch内部使用了读写锁：**Sender端要发送数据修改通道中的数据时，需要申请写锁，论是Sender还是Receiver端，在调用`borrow()`或其它一些方式访问通道数据时，都需要申请读锁**。因此，访问通道数据时要尽快释放读锁，否则可能会长时间阻塞Sender端的发送操作。

如果Sender端未发送数据，或者隔较长时间才发送一次数据，那么通道中的数据在一段时间内将一直保持不变。如果Receiver在这段时间内去多次读取通道，得到的结果将完全相同。但有时候，可能更需要的是等待通道中的数据已经发生变化，然后再根据新的数据做进一步操作，而不是循环不断地去读取并判断当前读取到的值是否和之前读取的旧值相同。

watch通道已经提供了这种功能：Receiver端可以标记通道中的数据，记录该数据是否已经被读取过。Receiver端的`changed()`方法用于等待通道中的数据发生变化，其内部判断过程是：如果通道中的数据已经被标记为已读取过，那么`changed()`将等待数据更新，如果数据未标记过已读取，那么`changed()`认为当前数据就是新数据，`changed()`会立即返回。

Receiver端的`borrow()`方法不会标记数据已经读取，所以`borrow()`之后调用的`changed()`会立即返回。但是`changed()`等待到新值之后，会立即将该值标记为已读取，使得下次调用`changed()`时会进行等待。

此外，Receiver端还有一个`borrow_and_update()`方法，它会读取数据并标记数据已经被读取，因此随后调用`chagned()`将进入等待。

最后再强调一次，无论是Sender端还是Receiver端，访问数据的时候都会申请读锁，要尽量快地释放读锁，以免Sender长时间无法发送数据。



