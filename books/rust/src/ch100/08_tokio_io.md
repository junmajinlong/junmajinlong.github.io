# 理解并掌握tokio的异步IO

任何一个异步框架的核心目标都是异步IO，更高效率的IO编程也是多数时候我们使用异步框架的初衷。

tokio的异步IO组件封装了`std::io`中的几乎所有东西的异步版本，同步IO和异步IO的API在使用方法上也类似。例如，下面是异步版本的文件读取操作：

```rust
use tokio::io::AsyncReadExt;
use tokio::fs::File;

#[tokio::main]
async fn main() {
    let mut f = File::open("foo.txt").await.unwrap();
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer).await.unwrap();
    println!("The bytes: {:?}", &buffer[..n]);
}
```

`tokio::io`提供了不少组件，在写代码时可能也会用到一些额外的组件，我自己在学习的时候对这些组件感觉有点懵，因此我觉得非常有必要去搞清楚各个组件是干什么用的，搞清楚之后再去学习组件的相关用法会轻松许多。我会尽量循序渐进地介绍每一个组件。

另外，本文会先从IO读写示例开始引入Rust异步IO编程的方式，然后再尽量从基础开始解释读和写，这部分基础内容可能会较长较枯燥。我觉得对于贯穿多数应用程序的IO来说，解释的篇幅再长都是合理的，它太重要了。此外，即便没有学习过`std::io`，阅读本文也会对`std::io`有系统性的理解。


## 异步IO示例一：文件IO

tokio也支持异步的文件操作，包括文件的IO读写类操作。

例如，按行读取文件：

```rust
let file = tokio::fs::File::open("/tmp/a.log").await.unwrap();
// 将file转换为BufReader
let mut buf_reader = tokio::io::BufReader::new(file).lines();
// 每次读取一行
while let Some(line) = buf_reader.next_line().await.unwrap() {
    // 注意lines()中的行是不带结尾换行符的，因此使用println!()而不是print!()
    println!("{}", line);
}
```

上面将File转换为BufReader将使得读取更为简便，比如上面可以直接按行读取文件。如果不转换为BufReader，而是直接通过File进行读取，将只能按字节来读取。如果文件中的是字符串数据，那么按字节读取时会比较麻烦。

当然，也可以通过`read_line()`的方式来按行读取：

```rust
let file = tokio::fs::File::open("/tmp/a.log").await.unwrap();
let mut buf_reader = tokio::io::BufReader::new(file);
let mut buf = String::new();

loop {
    match buf_reader.read_line(&mut buf).await {
        Err(_e) => panic!("read file error"),
        // 遇到了文件结尾，即EOF
        Ok(0) => break,
        Ok(_n) => {
          	// read_line()总是保留行尾换行符(如果有的话)，因此使用print!()而不是println!()
            print!("{}", buf);
          	// read_line()总是将读取的内容追加到buf，因此每次读取完之后要清空buf
            buf.clear();
        }
    }
}
```

## 异步IO示例二：网络IO

网络IO是最常见的IO方式之一，下面是一个非常简单的Client/Server两端通信中的服务端的示例。该示例中，Client/Server两端协议好以行为单位传输数据。

下面是服务端的代码： 

```rust
use tokio::{
    io::{AsyncBufReadExt, AsyncWriteExt},
    net::{
        tcp::{OwnedReadHalf, OwnedWriteHalf},
        TcpListener, TcpStream,
    },
    sync::mpsc,
};

#[tokio::main]
async fn main() {
    let server = TcpListener::bind("127.0.0.1:8888").await.unwrap();
    while let Ok((client_stream, client_addr)) = server.accept().await {
        println!("accept client: {}", client_addr);
        // 每接入一个客户端的连接请求，都分配一个子任务，
        // 如果客户端的并发数量不大，为每个客户端都分配一个thread，
        // 然后在thread中创建tokio runtime，处理起来会更方便
        tokio::spawn(async move {
            process_client(client_stream).await;
        });
    }
}

async fn process_client(client_stream: TcpStream) {
    let (client_reader, client_writer) = client_stream.into_split();
    let (msg_tx, msg_rx) = mpsc::channel::<String>(100);
  
  	// 从客户端读取的异步子任务
    let mut read_task = tokio::spawn(async move {
        read_from_client(client_reader, msg_tx).await;
    });

    // 向客户端写入的异步子任务
    let mut write_task = tokio::spawn(async move {
        write_to_client(client_writer, msg_rx).await;
    });

    // 无论是读任务还是写任务的终止，另一个任务都将没有继续存在的意义，因此都将另一个任务也终止
    if tokio::try_join!(&mut read_task, &mut write_task).is_err() {
        eprintln!("read_task/write_task terminated");
        read_task.abort();
        write_task.abort();
    };
}

/// 从客户端读取
async fn read_from_client(reader: OwnedReadHalf, msg_tx: mpsc::Sender<String>) {
    let mut buf_reader = tokio::io::BufReader::new(reader);
    let mut buf = String::new();
    loop {
        match buf_reader.read_line(&mut buf).await {
            Err(_e) => {
                eprintln!("read from client error");
                break;
            }
            // 遇到了EOF
            Ok(0) => {
                println!("client closed");
                break;
            }
            Ok(n) => {
                // read_line()读取时会包含换行符，因此去除行尾换行符
                // 将buf.drain(。。)会将buf清空，下一次read_line读取的内容将从头填充而不是追加
                buf.pop();
                let content = buf.drain(..).as_str().to_string();
                println!("read {} bytes from client. content: {}", n, content);
                // 将内容发送给writer，让writer响应给客户端，
                // 如果无法发送给writer，继续从客户端读取内容将没有意义，因此break退出
                if msg_tx.send(content).await.is_err() {
                    eprintln!("receiver closed");
                    break;
                }
            }
        }
    }
}

/// 写给客户端
async fn write_to_client(writer: OwnedWriteHalf, mut msg_rx: mpsc::Receiver<String>) {
    let mut buf_writer = tokio::io::BufWriter::new(writer);
    while let Some(mut str) = msg_rx.recv().await {
        str.push('\n');
        if let Err(e) = buf_writer.write_all(str.as_bytes()).await {
            eprintln!("write to client failed: {}", e);
            break;
        }
    }
}
```

上面的Server代码示例中，通过`into_split()`将TcpStream分离得到OwnedWriteHalf和OwnedReadHalf，并且将这它们分别放进了负责写和读的异步子任务中。

需要注意的是，在`read_from_client`函数中，将reader转换成了BufReader，在`write_to_client`函数中将writer转换成了BufWriter。如果不进行转换，直接通过reader也可以进行异步读操作，直接通过writer也能进行异步写操作，但是它们读写的对象都是字节，且没有缓冲空间，操作起来要稍繁琐啰嗦，并且有可能需要自己实现缓冲空间来提高效率。因此，通常会将它们转换为带有Buffer的BufReader和BufWriter，方便读写操作。例如上面示例中可以通过BufReader按行读取(一次读取一行)。

示例中处理网络IO的方式是比较常见的一种方式，还有更多处理方式，比如使用`tokio_util::codec::LinesCodec`按行读写时将更方便简洁安全。

## AsyncRead和AsyncWrite

通过前面的示例可发现，File、TcpStream、OwnedReadHalf/OwnedWriteHalf都具有异步读和写的能力，它们之所以能进行异步读写，是因为它们实现了AsyncRead Trait、AsyncWrite Trait。但它们只能以字节为读写对象。

`AsyncRead` Trait和`AsyncWrite` Trait是`tokio::io`的最基本组件，它们是标准库中Read和Write这两个Trait的异步版本，它们提供了最基本的异步读、写能力：**当需要进行异步读、写的时候，不会像同步的读写一样阻塞线程，而是会等待可读、可写事件的发生，同时切换到调度器使其能在等待事件发生的过程中调度其它异步任务来执行**。

## AsyncReadExt和AsyncWriteExt

`AsyncRead`和`AsyncWrite`只是提供了最基本的异步读写能力，它们并没有提供方便的读写方式。好在，只要实现了`AsyncRead`或`AsyncWrite`(例如`tokio::fs::File`、`tokio::net::TcpStream`都实现了它们)，在开启tokio的`io-util`特性后，就会自动拥有`AsyncReadExt`和`AsyncWriteExt`中定义的一些方便的读写方法，例如read()、read_buf()、write()、write_all()、flush()等。

```toml
tokio = {version = "1.13", features = ["rt", "io-util", "rt-multi-thread"]
```

因此，当需要进行异步读写时，几乎总是会导入这两个扩展包：
```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt}
```

AsyncReadExt和AsyncWriteExt都提供了很多方法，但其中的多数是同类不同名的方法。因此，要掌握的方法其实并不多。

### AsyncReadExt

AsyncReadExt提供了以下几个方法：
- read(): 读取数据并填充到指定的buf中  
- read_buf(): 读取数据并追加到指定的buf中，和read()的区别稍后解释  
- read_exact(): 尽可能地读取数据填充满buf，即按照buf长度来读取  
- read_to_end(): 一直读取并不断填充到指定的Vec Buf中，直到读取时遇到EOF  
- read_to_string(): 一直读取并不断填充到指定的String Buf中，直到读取时遇到EOF，要求所读取的是有效的UTF-8数据  
- take(): 消费掉Reader并返回一个Take，Take限制了最多只能读取指定数量的数据  
- chain(): 将多个Reader链接起来，读完一个目标的数据之后可以接着读取下一个目标  

最基本的是`read()`方法，它从目标中读取一定量的数据并保存到指定的buf中。这里有一些注意事项需要解释清楚，参考下面示例中的注释。

假设在项目根目录下有一个名为`a.log`的文件，其中有10个字节的数据`abcdefghij`(没有换行符)。可以使用`tokio::fs::File`去读取文件数据，这是因为`tokio::fs::File`实现了`AsyncRead`。不过要注意，要先在Cargo.toml中开启tokio的fs特性。

```toml
tokio = {version = "1.12", features = ["rt", "rt-multi-thread", "fs", "macros", "io-util"]}
```

读取文件数据的示例代码如下：
```rust
use tokio::{self, runtime, fs::File, io::{self, AsyncReadExt}};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // 打开文件，用于读取，因读取数据时需要更新文件指针位置，因此要加上mut
        let mut f = File::open("a.log").await.unwrap();

        // 提供buf，用来存放每次读取的数据，因要写入buf，因此加上mut
        // 读取的数据都是字节数据，即u8类型，因此为u8类型的数组或vec(或其它类型)
        let mut buf = [0u8; 5];

        // 读取数据，可能会等待，也可能立即返回，对于文件读取来说，会立即返回，
        // 每次读取数据都从buf的index=0处开始覆盖式写入到buf，
        // buf容量为5，因此一次读取最多5个字节。
        // 返回值为本次成功读取的字节数。
        let n = f.read(&mut buf).await.unwrap();
        // 由于只读取了n个字节保存在buf中，如果buf容量大于n，
        // 那么index=n和后面的数据不是本次读取的数据
        // 因此，截取buf到index=n处，这部分才是本次读取的数据
        let str = std::str::from_utf8(&buf[..n]);
        println!("first read {} bytes: {:?}", n, str);

        // 第二次读取5个字节，本次读取之后，a.log中的10个字节都已经被读取，
        let n = f.read(&mut buf).await.unwrap();
        // 因每次读取都将所读数据从buf的index=0处开始覆盖保存，
        // 因此，仍然是通过`&buf[0..n]`来获取本次读取的数据
        println!("second read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]));

        // a.log中的数据在第二次read时已被读完，再次读取，将遇到EOF，
        // 遇到EOF时，read将返回Ok(0)表示读取的数据长度为0，
        // 但返回的长度为0不一定代表遇到了EOF，也可能是buf的容量为0
        let n = f.read(&mut buf).await.unwrap();
        // 因遇到EOF，没有读取到任何数据保存到buf，
        // 因此`&buf[..n]`为空slice，转换为字符串则为空字符串
        println!("third read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]));
    });
}
```

输出结果：
```
first read 5 bytes: Ok("abcde")
second read 5 bytes: Ok("fghij")
third read 0 bytes: Ok("")
```

上面的示例中使用数组作为buf，使用Vec也是可以的。
```rust
let mut buf = vec![0u8; 5];
```

`read()`每次将读取的数据从buf的index=0处开始覆盖时保存到buf。另一个方法`read_buf()`则是追加式保存，这要求每次读取数据时都会自动维护buf的指针位置，因此直接使用数组和Vec作为buf是不允许的。事实上，`read_buf()`参数要求了应使用实现了`bytes::buf::BufMut` Trait的类型，它会在维护内部的位移指针，且在需要时会像vec一样自动扩容。

例如`bytes::BytesMut`实现了`bytes::buf::BufMut`。在Cargo.toml中添加bytes：
```toml
bytes = "1.1"
```

假设a.log文件的前4字节为abcd，它后面还有几十个字节的数据。示例代码如下：
```rust
use tokio::{self, fs::File, io::{self, AsyncReadExt}, runtime};
use bytes::BytesMut;

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let mut f = File::open("a.log").await.unwrap();
        // 初始容量为4
        let mut buf = BytesMut::with_capacity(4);

        // 第一次读取，读取容量大小的数据，即4字节数据，
        // 此时BytesMut内部的位移指针在offset = 3处
        let n = f.read_buf(&mut buf).await.unwrap();
        println!("first read {} bytes: {:?}", n, std::str::from_utf8(&buf));

        // 第二次读取，因buf已满，这次将一次性读取剩余所有数据(只请求一次读系统调用)，
        // BytesMut也将自动扩容以便存放更多数据，且可能会根据所读数据的多少进行多次扩容，
        // 所读数据都将从index=4处开始追加保存
        let n = f.read_buf(&mut buf).await.unwrap();
        println!("second read {} bytes: {:?}", n, std::str::from_utf8(&buf));
    });
}
```

输出结果：
```
first read 4 bytes: Ok("abcd")
second read 36 bytes: Ok("abcdefghijABCDEFGHIJabcdefghij")
```

`read_exact()`方法是根据buf的容量来决定读取多少字节的数据，和`read()`一样的是，每次读取都会将所读数据从buf的index=0处开始覆盖(到了这里应该可以发现一点，除非是内部自动维护buf位置的，都会从index=0处开始覆盖式存储)，和`read()`不一样的是，read_exact()明确了要读取多少字节的数据后(即buf的容量)，如果没有读取到这么多数据，就会报错，比如提前遇到了EOF，将报`ErrorKind::UnexpectedEof`错误。

例如，a.log文件中只有10个字节的数据，但buf的容量为40字节，在`read_exact()`时会报错。
```rust
use tokio::{ self, fs::File, io::{self, AsyncReadExt}, runtime };

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let mut f = File::open("a.log").await.unwrap();
        let mut buf = [0u8; 40];

        let n = f.read_exact(&mut buf).await.unwrap();
        println!("first read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]));
    });
}
```

`read_to_end()`方法提供了一次性读取所有数据的功能，它会不断读取，直到遇到EOF。在不断读取的过程中，buf可能会进行多次扩容，因此buf不是固定大小的数组，而是Vec。这是非常好用的功能，不过对于文件的读取来说，`tokio::fs::read()`提供了更简单更高效的一次性读取文件所有内容的方式。

另外需要注意的是，`read_to_end()`所读取的数据是在现有Vec数据的基础上进行追加的，因此，Vec一定会有至少一次的扩容。

例如，a.log文件有10个字节数据，初始Vec buf容量为5，那么这10个数据将从Vec的index=5处开始追加存储。
```rust
use tokio::{ self, fs::File, io::{self, AsyncReadExt}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let mut f = File::open("a.log").await.unwrap();
        // buf初始容量为5
        let mut buf = vec![0u8; 5];

        // read_to_end读取的数据，从buf的index=5处开始追加保存
        // 返回成功读取的字节数
        let n = f.read_to_end(&mut buf).await.unwrap();
        println!("first read {} bytes: {:?}", n, buf);
        println!("first read {} bytes: {:?}", n, std::str::from_utf8(&buf[(buf.len() - n)..]));
    });
}
```

输出结果：
```
first read 10 bytes: [0, 0, 0, 0, 0, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106]
first read 10 bytes: Ok("abcdefghij")
```

`read_to_string()`方法类似于`read_to_end()`，不同的是它将读取的字节数据直接解析为UTF-8字符串。因此，该方法需指定String作为buf。同样的，`read_to_string()`所读取的数据会追加在当前String buf的尾部。
```rust
use tokio::{ self, fs::File, io::{self, AsyncReadExt}, runtime };

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let mut f = File::open("a.log").await.unwrap();
        let mut buf = "xyz".to_string();

        let n = f.read_to_string(&mut buf).await.unwrap();
        println!("first read {} bytes: {:?}", n, buf);
    });
}
```
输出结果：
```
first read 10 bytes: "xyzabcdefghij"
```

`take()`方法可限制最多只读取几个字节的数据，该方法会消费掉Reader，并返回一个Take类型的实例。Take实例内部会保留原来的Reader，并添加了一个限制接下来最多只能读取多少字节的字段`limit_`。
```rust
pub struct Take<R> {
    #[pin]
    inner: R,  // Move进来的原始的Reader
    // Add '_' to avoid conflicts with `limit` method.
    limit_: u64,  // 最多只允许从Reader中读取多少个字节
}
```

当已读数据量达到了限制的数量后，下次再读，将强制遇到EOF，尽管这时候可能还没有遇到内部Reader的EOF。不过，可以通过Take的`set_limit()`重新修改接下来可最多读取的字节数，set_limit()会重置已读数量，相当于重新返回了一个新的Take实例。当然，需要的时候可以通过`remaining()`来确定还可允许读取多少数量的数据。

例如，a.log文件有20字节的数据，先通过`take()`得到限制最多读取5字节Take，通过Take读取2字节，再读取3字节，将遇到EOF，再通过Take的`set_limit()`修改限制再最多读取10字节。

```rust
use tokio::{ self, fs::File, io::{self, AsyncReadExt}, runtime };

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f = File::open("a.log").await.unwrap();
        let mut t = f.take(5);

        let mut buf = [0u8; 2];
        let n = t.read(&mut buf).await.unwrap();
        println!( "first read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]) );

        let mut buf = [0u8; 3];
        let n = t.read(&mut buf).await.unwrap();
        println!( "second read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]) );

        let mut buf = [0u8; 15];
        t.set_limit(10);
        let n = t.read(&mut buf).await.unwrap();
        println!( "third read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]) );

        let n = t.read(&mut buf).await.unwrap();
        println!( "fourth read {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]) );
    });
}
```

输出结果：
```
first read 2 bytes: Ok("ab")
second read 3 bytes: Ok("cde")
third read 10 bytes: Ok("fghij01234")
fourth read 0 bytes: Ok("")
```

另外一个方法`chain()`，可将两个Reader串联起来(可多次串联)，当第一个Reader遇到EOF时，继续读取将自动读取第二个Reader的数据。实际上，当第一个Reader遇到EOF时，串联后得到的Reader不会因此而遇到EOF，只是简单地将内部的`done_first`字段设置为true，表示第一个Reader已经处理完。只有第二个Reader遇到EOF时，串联后的Reader才遇到EOF。

多数时候用来读取多个文件的数据，当然，也可以将同一个文件串联多次。

```rust
use tokio::{ self, fs::File, io::{self, AsyncReadExt}, runtime };

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f1 = File::open("a.log").await.unwrap();
        let f2 = File::open("b.log").await.unwrap();
        let mut f = f1.chain(f2);

        let mut buf = [0u8; 20];
        let n = f.read(&mut buf).await.unwrap();
        println!("data {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]));

        let n = f.read(&mut buf).await.unwrap();
        println!("data {} bytes: {:?}", n, std::str::from_utf8(&buf[..n]));
    });
}
```

输出结果：
```
data 10 bytes: Ok("abcdefghij")
data 10 bytes: Ok("0123456789")
```

从上面示例的结果可知，虽然读完第一个Reader后chain Reader不会EOF，但是读取却会在此停止，下次读取才会继续读取第二个Reader。

但如果使用`read_to_end()`或`read_to_string()`则会一次性读完所有数据，因为这两个方法内部会多次读取直到遇到EOF。例如：
```rust
use tokio::{self, fs::File, io::{self, AsyncReadExt}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f1 = File::open("a.log").await.unwrap();
        let f2 = File::open("b.log").await.unwrap();
        let mut f = f1.chain(f2);

        let mut data = String::new();
        let n = f.read_to_string(&mut data).await.unwrap();
        println!("data {} bytes: {:?}", n, data);
    });
}
```

输出结果：
```
data 20 bytes: "abcdefghij0123456789"
```

上面介绍了AsyncReadExt提供的各种方法，下面再介绍AsyncWriteExt提供的各种方法。

### AsyncWriteExt

AsyncWriteExt提供了以下几个方法：
- write(): 将给定的字节数组中的数据写入到Writer中  
- write_all(): 将给定的字节数组中的所有数据写入到Writer中  
- write_buf(): 将给定buf的数据写入到Writer，每次写入时，buf会自动维护内部的位移指针  
- write_all_buf(): 将给定buf的数据全部写入到Writer  
- write_vectored(): 将一个或多个buf的所有数据写入到Writer  
- flush(): 将缓冲中的数据刷入目标Writer。适用于BufWriter  
- shutdown(): 关闭Writer，关闭时如果(BufWriter的)缓冲中还有数据，则会触发flush保证数据刷入Writer  

最基础的是write()方法，它尝试将给定的字节数组(即`[u8; N]`)中的所有字节写入到Writer中，但不一定会全部写入成功。
```rust
use tokio::{self, fs::File, io::AsyncWriteExt, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // 以write-only模式打开文件
        // 如果文件不存在，则创建，如果已存在，则截断文件
        let mut f = File::create("a.log").await.unwrap();

        let n = f.write(b"hello world").await.unwrap();
        println!("write {} bytes", n);
    });
}
```

和`write()`类似的是`write_all()`方法，它要求给定的字节数组的所有数据全部写入成功后才返回，除非遇到错误。

`flush()`方法适用于使用了`BufWriter`的场景。当使用了`BufWriter`，写入的数据首先写入到一个缓冲空间，在适当的时候(比如缓冲空间已满时)才会将缓冲空间中的数据真正写入到目标，使用flush()可强制将缓冲空间的数据写入到目标。
```rust
use tokio::io::{BufWriter, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() {
    let f = File::create("foo.txt").await.unwrap();
    let mut buffer = BufWriter::new(f);

    // 这次写入只是写入到缓冲空间
    buffer.write_all(b"some bytes").await.unwrap();

    // 将缓冲空间的数据刷入writer
    buffer.flush().await.unwrap();
}
```

`shutdown()`用于关闭Writer，shutdown之后，无法再通过writer写入新数据。但如果在关闭时，`BufWriter`的缓冲空间中还有数据，则会自动将数据刷入到writer。

## 带缓冲的读、写

虽然`AsyncReadExt`和`AsyncWriteExt`提供了方便的读写方法，但是每次调用其中的读、写方法都会立即向操作系统请求发起一次读、写系统调用，如果需要读、写大量数据，且每次只读、写少量字节(比如对于读来说，给定的buf小，但有大量数据要读时)，那么会请求非常多次数的系统调用。而每次请求系统调用都意味着要从用户空间陷入操作系统的内核空间，频繁切换上下文会出现大量CPU时间的浪费，IO效率也会随之降低。

并且，浏览一下`AsyncReadExt` Trait所提供的方法就会发现，它只提供了按字节数读取或读取所有数据的方法，这种读取方式比较原始，有时候也不太方便。很多时候，特别是对文件或终端的读取来说，需要的是更简便的按行读取的方式，即每次读取一行，而想要按行读取，前提是能够在读取时去识别所读数据中的换行符并返回换行符前面的数据。显然，按字节数量来读取时，是不具备这种能力的。

标准库和`tokio::io`都提供了带缓冲功能的读写组件。当调用读、写方法时，不一定会立即就执行操作系统上的读写操作，而是先尝试从缓冲中读或先写向缓冲：
- 对于读操作，如果缓冲中已经有数据，则直接返回缓冲中的数据，如果缓冲中没有，则请求操作系统发起读系统调用。向操作系统请求读时，有可能会请求比实际所需更多的数据，多出来的数据将先缓冲在缓冲空间中等待可能的下次读取  
   - **除了提供缓冲空间，还更进一步地提供了按行读取的方式**，一直读取到换行符并返回，缓冲中换行符后面剩余的数据则继续保留在缓冲空间中  
- 对于写操作，将先写入缓冲，然后按照缓冲模式决定何时执行真正的写操作(即发起写系统调用)，此时会将缓冲中的数据写入操作系统(可能是写入操作系统所维护的缓冲空间)。例如，只有当缓冲中的数据达到了8K时才开始真正写入操作系统，如果没有达到8K，则数据一直保存在缓冲中  

`tokio::io`提供了`AsyncBufRead` Trait，实现该Trait的结构将能够在读写时使用缓冲空间。当然，所需的缓冲空间由实现者自身提供。

注意，只有`AsyncBufRead`，没有`AsyncBufWrite` Triat。实际上并不需要`AsyncBufWrite`，因为写入数据时只是需要一个写缓冲空间来缓冲写操作，实现`AsyncBufRead`就可以提供和维护一个缓冲空间。也就是说，在必要时，让某个Writer实现`AsyncBufRead`就可以提供带缓冲的写能力。

当实现了`AsyncBufRead`时，将自动实现`AsyncBufReadExt`并获取其中定义的一些方便的方法，这些方法稍后介绍。

如果某Reader或Writer没有实现`AsyncBufRead`，那么可以使用`tokio::io::BufReader`和`tokio::io::BufWriter`来将其转换为带有缓冲空间的Reader或Writer。`tokio::io::BufReader`和`tokio::io::BufWriter`内部带有并维护缓冲空间。

```rust
pub struct BufReader<R> {
  #[pin]
  pub(super) inner: R,
  pub(super) buf: Box<[u8]>,
  pub(super) pos: usize,
  pub(super) cap: usize,
  pub(super) seek_state: SeekState,
}

pub struct BufWriter<W> {
  #[pin]
  pub(super) inner: W,
  pub(super) buf: Vec<u8>,
  pub(super) written: usize,
  pub(super) seek_state: SeekState,
}
```

例如，`tokio::fs::File`没有实现`AsyncBufRead`，但是可以转换为`BufReader`和`BufWriter`：
```rust
let f1 = File::open("foo.txt").await.unwrap();
let mut reader = tokio::io::BufReader::new(f);

let f2 = File::create("foo.txt").await.unwrap();
let mut writer = tokio::io::BufWriter::new(f);
```

此外，`BufReader`和`BufWriter`分别为读、写提供缓冲功能，还有一个`tokio::io::BufStream`则同时提供读、写的缓冲功能，它相当于`BufReader`和`BufWriter`的结合体。也就是说，`BufStream`的实例即可进行带缓冲的读，也可以进行带缓冲的写。
```rust
let f1 = File::open("foo.txt").await.unwrap();
let mut reader = tokio::io::BufStream::new(f);
```

需注意的是，带缓冲空间的读、写操作不总是比不带缓冲的读、写操作更高效，只有对于多次且少量的读、写操作来说，带缓冲的读写效率会更高。如果是读、写少量次数或一次性读、写大量数据的操作，不带缓冲空间的读、写操作效率会更高一些。

再来介绍`AsyncBufReadExt`提供的方法的用法，有以下几个方法：
- lines(): 返回Lines，Lines有一个`next_line()`方法，可不断地从BufReader中读取下一行(返回内容不包括换行符)，直到遇到EOF  
- read_line(): 从BufReader中读取下一行(返回内容包含换行符)并追加到指定的String buf的尾部  
- read_until(): 一直读取，直到遇到指定的字节(分隔符字节)或EOF(返回内容包含分隔符)，读取的内容将追加到buf的尾部  
- split(): 根据指定的字节将Reader进行划分，返回Split，Split提供了`next_segment()`方法，可不断从BufReader中读取下一个分割片段(返回内容不包含分隔符)直到遇到EOF  

`lines()`可按行进行异步迭代式的读取：
```rust
use tokio::{self, fs::File, io::{AsyncBufReadExt, BufReader}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f = File::open("a.log").await.unwrap();
        let mut lines = BufReader::new(f).lines();
        while let Some(line) = lines.next_line().await.unwrap() {
            println!("read line: {}", line);
        }
    });
}
```

类似的，`split()`是指定分隔符，而不是默认的换行符作为分隔符。例如，指定换行符作为分隔符。
```rust
use tokio::{self, fs::File, io::{AsyncBufReadExt, BufReader}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f = File::open("a.log").await.unwrap();
        let mut lines = BufReader::new(f).split(b'\n');
        while let Some(line) = lines.next_segment().await.unwrap() {
            println!("read line: {}", String::from_utf8(line).unwrap());
        }
    });
}
```

需注意的是，split()方法只能指定字节作为分隔符，不能指定字符分隔符，另外，Split的`next_segment()`方法读取的数据会保存到`Vec<u8>`中，而不是直接返回String。

`read_line()`方法则是从缓冲空间中读取一行，读取的内容(包含换行符)会追加到指定的String buf中：
```rust
use tokio::{self, fs::File, io::{AsyncBufReadExt, BufReader}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f = File::open("a.log").await.unwrap();
        let mut f = BufReader::new(f);

        let mut data = String::new();
        f.read_line(&mut data).await.unwrap();
        print!("first line: {}", data);
    });
}
```

`read_until()`方法类似于`read_line()`，只是不是读取到换行符停止，而是读取到指定的分隔符停止。同样的，也只能使用字节分隔符，读取的内容追加到Vec buf中。
```rust
use tokio::{self, fs::File, io::{AsyncBufReadExt, BufReader}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f = File::open("a.log").await.unwrap();
        let mut f = BufReader::new(f);

        let mut data = Vec::new();
        f.read_until(b'\n', &mut data).await.unwrap();
        print!("first line: {}", String::from_utf8(data).unwrap());
    });
}
```



## 随机读写Seek

在进行读、写时，会判断从Reader的哪个位置开始读以及从Writer的哪个位置开始写，这个位置称为位置偏移(offset)。

多数情况下，Reader第一次读数据时，是从offset=0处开始读的，即从Reader的第一个字节开始读，读多少字节，偏移指针就向前前进几个字节，下次再读取将从更新后的偏移位置处开始向后读取。

例如，以只读方式打开a.log文件时，第一次读取时，从第一个字节开始读取，如果第一次读取了10个字节，那么偏移指针将更新到文件的index=9处，第二次读取将从index=10处的字节开始读取。一直这样一边读取一边更新位置偏移，直到读完最后一个字节，偏移指针将更新到文件的最尾部，再向后读取将遇到EOF。

同理，写数据时，也会不断递进更新偏移指针。例如从当前位置处写了10个字节，那么偏移指针将向后递增10个字节。

需要注意的是，如果偏移指针的后面还有数据，那么在写数据时，将会逐字节覆盖原本的数据。例如，a.log文件有10个字节的数据，分别是从0到9的数字，当偏移指针位于offset=1处时，写入`"ab"`两个字节，这两个字节将分别覆盖原来的数字1和2，但3到9保持不变。

此外，可以在代码中轻松修改位置偏移。`std::io`和`tokio::io`都提供了操作偏移指针相关的组件：`std::io::Seek`和`tokio::io::AsyncSeek`。因为通过它们可以随时修改偏移指针，因此可以从任意位置开始进行读、写操作，这种方式的IO，也常称为随机读写，与通常情况下的顺序读写相对应。

本文介绍tokio相关的随机读写(即如何操作偏移指针)，对标准库的随机读写方式完全可以照葫芦画瓢。

`tokio::io::AsyncSeek`是一个Trait，实现了该Trait的类型可异步操作偏移指针。当实现了该Trait时，将自动实现`tokio::io::AsyncSeekExt`并从中获取以下几个操作偏移指针的方法：
- seek(): 设置偏移指针的位置并返回新的偏移位置  
- rewind(): 将偏移指针设置到offset=0处  
- stream_position(): 返回当前偏移指针的位置  

例如，`tokio::fs::File`已经实现了AsyncSeek，当打开a.log文件时，可设置它的偏移指针。假设a.log文件中存放了`abcdefghij`共10个字节的数据。
```rust
use std::io::SeekFrom;
use tokio::{self, fs::File, io::{AsyncReadExt, AsyncSeekExt}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
      // 只读方式打开文件时，偏移位置offset = 0
      let mut f = File::open("a.log").await.unwrap();

      // seek()设置offset = 4，从offset = 4开始读取，即从第5个字节开始读取
      // seek()返回设置后的偏移位置
      let n = f.seek(SeekFrom::Start(5)).await.unwrap();
      println!("set, offset = {}", n);
      
      let mut str = String::new();
      f.read_to_string(&mut str).await.unwrap();
      // 返回当前的偏移位置
      let n = f.stream_position().await.unwrap();
      println!("after read, offset = {}, data = {}", n, str);

      // 将偏移指针重置于offset = 0处
      f.rewind().await.unwrap();
      let n = f.stream_position().await.unwrap();
      println!("rewind, offset = {}", n);
    });
}
```

输出结果：
```
set, offset = 5
after read, offset = 10, data = fghij
rewind, offset = 0
```

上面的示例代码中，使用了`std::io::SeekFrom`，它是一个Enum，用来描述偏移位置。
```rust
pub enum SeekFrom {
    Start(u64),
    End(i64),
    Current(i64),
}
```

`SeekFrom::Start(u64)`描述从最开头(即offset = 0)开始计算的字节数，只能是u64，即可以是0，但不能是负数。例如，`SeekFrom::Start(10)`表示第10个字节位置处。

`SeekFrom::End(i64)`描述从最尾部开始计算的字节数，可以是正数、负数或0。例如：
- `SeekFrom::End(0)`表示最尾部的位置，即最后一个字节的后面  
- `SeekFrom::End(10)`表示最尾部向后的10个字节位置，即最后一个字节的后面10个字节，显然将偏移指针设置到此位置时已经向后超越了边界，但这是允许的。中间的10个字节将成为孔洞。对于文件来说，如果有很多空洞，这样的文件称为稀疏文件  
- `SeekFrom::End(-10)`表示最尾部向前的10个字节位置，即倒数第10个字节前面、倒数第11个字节的后面。不允许向前超出边界，否则将报错  

`SeekFfrom::Current(i64)`描述从当前偏移指针的位置开始计算的字节数，可以是正数、负数或0。例如：
- `SeekFrom::Current(0)`表示当前偏移指针的位置  
- `SeekFrom::Current(10)`表示当前偏移指针向后的10个字节位置，允许向后超越边界  
- `SeekFrom::Current(-10)`表示当前偏移指针向前的10个字节位置，不允许向前超出边界，否则将报错  

另外需要了解的是，有以下几个类型实现了AsyncSeek，也就是说它们都能进行随机读写：
- `tokio::fs::File`
- `tokio::io::BufReader`
- `tokio::io::BufWriter`
- `tokio::io::BufStream`

对于带缓冲空间的Reader和Writer需要额外注意，tokio提供的AsyncSeek，只允许在缓冲的底层IO目标中进行seek。也即是说，对`BufReader::new(fs::File::open(FILE))`进行seek，是对打开的文件进行seek，而不是在缓冲层进行seek。

之所以特地提醒这一点，是因为在某些语言中，允许在缓冲空间中进行seek(也即是说，缓冲空间也维护了一套偏移指针)，同时也会提供更底层的方法以便在缓冲的底层IO目标中进行seek。

## 标准输入、标准输出、标准错误

tokio在开启`io-std`特性之后，将提供三个函数：
- `tokio::io::stdin()`: 得到`tokio::io::Stdin`，即标准输入Reader，可从标准输入读取数据  
- `tokio::io::stdout()`: 得到`tokio::io::Stdout`，标准输出Writer，可写向标准输出  
- `tokio::io::stderr()`: 得到`tokio::io::Stderr`，标准错误Writer，可写向标准错误  

例如：
```rust
use tokio::{io::{AsyncWriteExt,AsyncReadExt}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let mut stdin = tokio::io::stdin();
        let mut stdout = tokio::io::stdout();

        // 循环从标准输入中读取数据
        loop {
            stdout.write(b"entry somethin: ").await.unwrap();
            stdout.flush().await.unwrap();

            let mut buf = vec![0; 1024];
            let n = match stdin.read(&mut buf).await {
                Err(_) | Ok(0) => break,
                Ok(n) => n,
            };

            buf.truncate(n);
            stdout.write(b"data from stdin: ").await.unwrap();
            stdout.write(&buf).await.unwrap();
            stdout.flush().await.unwrap();
        }
    });
}
```

## 全双工管道DuplexStream

`tokio::io::duplex()`提供了类似套接字的全双工读写管道：
```rust
// 参数指定管道的容量
fn duplex(max_buf_size: usize) -> (DuplexStream, DuplexStream)
```

DuplexStream可读也可写，当管道为空时，读操作会进入等待，当管道空间已满时，写操作会进入等待。

```rust
let (mut client, mut server) = tokio::io::duplex(64);

client.write_all(b"ping").await?;

let mut buf = [0u8; 4];
server.read_exact(&mut buf).await?;
assert_eq!(&buf, b"ping");

server.write_all(b"pong").await?;

client.read_exact(&mut buf).await?;
assert_eq!(&buf, b"pong");
```

在两端通信过程中，任意一端的关闭，都会导致写操作报错`Err(BrokenPipe)`，但读操作会继续读取直到管道的内容被读完遇到EOF。

DuplexStream实现了Send和Sync，因此可以跨线程、跨任务进行通信。

下面是模拟一个客户端和服务端，服务端向客户端循环不断地写入当前时间点，客户端不断读取来自服务端的数据并输出。

```rust
use chrono::Local;
use tokio::{self, runtime, time};
use tokio::io::{self, AsyncReadExt, AsyncWriteExt, DuplexStream};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

async fn write_duplex(r: &mut DuplexStream) -> io::Result<usize> {
    r.write(now().as_bytes()).await
}

async fn read_duplex(mut r: DuplexStream) {
    let mut buf = [0u8; 1024];
    loop {
        match r.read(&mut buf).await {
            Ok(0) | Err(_) => break,
            Ok(n) => {
                if let Ok(data) = std::str::from_utf8(&buf[..n]) {
                    println!("read from duplex: {}", data);
                }
            }
        };
    }
}

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let (client, mut server) = tokio::io::duplex(64);

        // client read data from server
        tokio::spawn(async move {
            read_duplex(client).await;
        });

        // server write now() to client 
        loop {
            match write_duplex(&mut server).await {
                Err(_) | Ok(0) => break,
                _ => (),
            }
            time::sleep(time::Duration::from_secs(1)).await;
        }
    });
}
```

## 分离Reader和Writer

`tokio::io::split()`方法可将可读也可写的目标(Stream)分离为Reader和Writer，Reader专门用于读操作，Writer专门用于写操作。分离得到的Reader和Writer分别称为ReadHalf和WriteHalf。

例如，TcpStream、BufStream、DuplexStream等都是可读也可写的Stream，有时候将它们分离更方便。

例如，上一小节通过`DuplexStream`模拟客户端和服务端的示例中，服务端只负责写，客户端只负责读，因此，可以将服务端分离为Reader和Writer，并将其Reader关闭，客户端也分离为Reader和Writer，并将其Writer关闭。

> 注：当然，也可以选择不关闭不用的Reader或Writer。在C语言和其它语言中，几乎总是建议关闭不使用的Reader或Writer，但在Rust中即便不关闭也没有这种担忧。

以修改客户端的读DuplexStream为例，代码如下：
```rust
async fn read_duplex(r: DuplexStream) {
    // 将DuplexStream分离为Reader和Writer，
    // 不使用Writer，因此关闭Writer
    let (mut reader, writer) = tokio::io::split(r);
    drop(writer);

    let mut buf = [0u8; 1024];
    loop {
        match reader.read(&mut buf).await {
            Ok(0) | Err(_) => break,
            Ok(n) => {
                if let Ok(data) = std::str::from_utf8(&buf[..n]) {
                    println!("read from duplex: {}", data);
                }
            }
        };
    }
}
```

## 拷贝Reader的数据到Writer

`tokio::io::copy()`方法可将Reader的所有数据(直到遇到EOF)直接拷贝给Writer。

例如：
```rust
use tokio::io;

let mut reader: &[u8] = b"hello";
let mut writer: Vec<u8> = vec![];

io::copy(&mut reader, &mut writer).await?;

assert_eq!(&b"hello"[..], &writer[..]);
```

## 接下来

本文介绍了tokio提供的IO编程相关的常用组件，对于其他语言来说，通过这些组件，就可以开始上手IO编程了。

但对于Rust语言的编程风格来说还不够，Rust语言充满了抽象，tokio的异步IO也需要一些抽象让某些IO变得更方便。下一篇将介绍Async Stream Sink以及`tokio_util::codec`按帧(Frame)进行IO的方式。
