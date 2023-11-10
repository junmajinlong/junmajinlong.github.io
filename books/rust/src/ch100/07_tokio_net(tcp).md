# 使用tokio::net进行网络编程

tokio提供了类似`std::net`所提供的基本设施以便进行异步网络编程，主要包括tcp、udp和unix domain三方面。

网络编程需要大量的网络编程知识，且和IO编程息息相关，因暂时还未介绍`tokio::io`，所以本文暂且仅介绍`tokio::net`的tcp编程相关的基础设施，不涉及具体的网络编程逻辑。(所以本文会比较枯燥，基本上是对官方文档的总结和引用)

要使用`tokio::net`，需在Cargo.toml文件中开启net特性：

```toml
tokio = {version = "1.13", features = ["rt", "net", "rt-multi-thread"]}
```

开启该特性之后，将可使用以下三个组件：  
- TcpSocket: 创建和操作套接字的基础组件  
- TcpListener: 对TcpSocket的一些封装，主要提供服务端套接字的相关操作  
- TcpStream: 代表已建立的可直接传递数据的连接，对客户端来说代表已经被服务端接收，对服务端来说代表accept后的套接字  

通常客户端可直接使用TcpStream，服务端可直接使用TcpListener和TcpStream，如果需要自定义修改套接字的选项或属性，则考虑使用TcpSocket。

## IpAddr和SocketAddr

在开始介绍`tokio::net`之前，需先简单介绍一下与之相关的`std::net::IpAddr`和`std::net::SocketAddr`(注意它们来自标准库)。

### IpAddr

IpAddr封装了IP地址，包括IP v4地址和IP v6地址：
```rust
pub enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}
```

IpAddr实现了FromStr，可直接将代表IP地址的字符串解析为IpAddr：
```rust
let localhsot: IpAddr = "127.0.0.1".parse().unwrap();
```

例如：
```rust
use std::net::{IpAddr, Ipv4Addr, Ipv6Addr};

let localhost = IpAddr::V4(Ipv4Addr::new(127, 0, 0, 1));
assert_eq!("127.0.0.1".parse(), Ok(localhost));
```

IpAddr还有一些方法，主要是一些布尔判断方法：  
- is_ipv4()：是否是一个ipv4地址  
- is_ipv6()：是否是一个ipv6地址  
- is_loopack()：是否是一个loopback地址  
- is_multicast()：是否是一个多播地址  
- is_unspecified()：是否是一个0.0.0.0地址  

IpAddr封装了ip v4地址或ip v6地址，以代表ip v4地址的Ipv4Addr为例。可使用`new()`并提供4个u8参数来创建ip v4地址：
```rust
use std::net::Ipv4Addr;

let localhost = Ipv4Addr::new(127, 0, 0, 1);
```

Ipv4Addr实现了FromStr，也可以很方便地直接将字符串解析为ip地址：
```rust
let localhost = "127.0.0.1".parse().unwrap();
```

可使用`octets()`将一个IP地址转换为u8数组，即new()的反向操作：

```rust
use std::net::Ipv4Addr;

let addr = Ipv4Addr::new(127, 0, 0, 1);
assert_eq!(addr.octets(), [127, 0, 0, 1]);
```

Ipv4Addr还有其它一些方法，多数都是布尔判断方法:  
- is_broadcast(): 是否是广播地址(255.255.255.255)  
- is_multicast(): 是否是多播地址(224.0.0.0/4)  
- is_private(): 是否是私有地址(10.0.0.0/8、172.16.0.0/12、192.168.0.0/16)  
- is_link_local(): 是否是链路本地地址(169.254.0.0/16)  
- is_loopback(): 是否是环回地址(127.0.0.0/8)
- is_unspecified(): 是否是0.0.0.0

此外，可直接对地址进行大小比较和等值比较。

### SocketAddr

SocketAddr代表包含了IP地址和端口号的套接字地址，它封装了ipv4套接字地址和ipv6套接字地址：

```rust
pub enum SocketAddr {
    V4(SocketAddrV4),
    V6(SocketAddrV6),
}
```

SocketAddr实现了FromStr，因此可直接将代表套接字地址的字符串解析为SocketAddr:
```rust
use std::net::{IpAddr, Ipv4Addr, SocketAddr};

let socket: SocketAddr = "127.0.0.1:8080".parse().unwrap();
```

SocketAddr自身也提供了new()方法，需提供IpAddr和端口号(u16)作为参数：
```rust
use std::net::{IpAddr, Ipv4Addr, SocketAddr};

let ip = IpAddr::V4(Ipv4Addr::new(127, 0, 0, 1));
let socket = SocketAddr::new(ip, 8080);
```

此外，还有以下几个方法：  
- is_ipv4(): 是否是ip v4套接字地址  
- is_ipv6(): 是否是ip v6套接字地址  
- ip(): 返回IP地址  
- port(): 返回端口号  
- set_ip(): 修改IP地址  
- set_port(): 修改端口号  

SocketAddr封装的代表ipv4套接字的SocketAddrV4也很简单直接，可由代表ipv4套接字的字符串解析得到，也可由new()方法创建，其也具有ip()、port()、set_ip()以及set_port()这几个方法。

```rust
use std::net::{Ipv4Addr, SocketAddrV4};

let socket = SocketAddrV4::new(Ipv4Addr::new(127, 0, 0, 1), 8080);

assert_eq!("127.0.0.1:8080".parse(), Ok(socket));
assert_eq!(socket.ip(), &Ipv4Addr::new(127, 0, 0, 1));
assert_eq!(socket.port(), 8080);
```

## tokio::net::TcpListener

TcpListener代表服务端套接字，可使用bind()方法指定要绑定的地址，bind()之后再await，即可开始监听。
```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main(){
  let listener = TcpListener::bind("127.0.0.1:8888").await.unwrap();
}
```

这里的listener代表的是服务端负责监听的套接字。

注意，`TcpListener::bind()`默认会开启TCP的地址重用选项(SO_REUSEADDR)。如果想要修改该选项或设置其它TCP选项，应使用TcpSocket来创建套接字并设置选项，然后再调用bind()方法得到监听套接字。

得到监听套接字之后，可使用accept()去接收来自客户端的连接请求。accept()会阻塞(等待)，直到有新的客户端发起连接请求。

accept()成功，表示和客户端之间成功建立TCP连接(连接进入Established状态)，同时它会返回一个新的套接字(TcpStream)和代表客户端的套接字地址(SocketAddr)。可通过该TcpStream和客户端传输数据，可通过该SocketAddr获取客户端的地址和端口信息。如果要获取本地套接字地址相关的信息，可使用listener的`local_addr()`方法。

通常来说，会在一个无限循环中去accept()，这样可以保证多次接收客户端的连接请求。此外，一般也会为每一个accept()成功后返回的TcpStream去分配一个独立的线程或异步任务，这样可以异步地和每个客户端进行通信，且不影响监听套接字继续监听更多的客户端连接请求。

因此，tcp编程的服务端最基本的处理模式大致如下：
```rust
async fn main(){
    let listener = TcpListener::bind("127.0.0.1:8888").await.unwrap();

    loop {
        let (client, client_sock_addr) = listener.accept().await.unwrap();
        tokio::spawn(async move {
          // 该任务负责处理client
        });
    }
}
```

此外，tokio的监听套接字可和标准库的监听套接字(`std::TcpListener`)来回转换。由于tokio只提供了成品套接字，无法设置很多的套接字选项，因此如果需要修改或设置某些套接字选项，需要先构建标准库的套接字并设置选项，然后使用`from_std()`将标准库套接字转换为tokio的套接字。与`from_std()`对应的是`into_std()`。

## tokio::net::TcpSocket

TcpSocket用于创建和设置套接字选项，它是未进行连接的套接字，可通过bind()和listen()操作得到服务端的监听套接字，可通过connect()得到客户端的套接字。

例如，创建监听套接字，下面的操作等价于`TcpListener.bind()`操作，它将监听`127.0.0.1:8080`端口：
```rust
use tokio::net::TcpSocket;

#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:8080".parse().unwrap();
    let socket = TcpSocket::new_v4().unwrap();
    socket.set_reuseaddr(true).unwrap();
    socket.bind(addr).unwrap();

    let listener = socket.listen(1024).unwrap();
}
```

下面的操作等价于`TcpStream::connect()`操作，它将连接`127.0.0.1:8080`并返回该连接的TcpStream：
```rust
use tokio::net::TcpSocket;

#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:8080".parse().unwrap();

    let socket = TcpSocket::new_v4().unwrap();
    let stream = socket.connect(addr).await.unwrap();
}
```

## TcpStream

TcpStream代表客户端和服务端之间已经建立的可以进行数据通信的TCP连接。当然，TcpStream也提供了`connect()`方法来方便地建立和TCP服务端的连接。

```rust
let mut stream = TcpStream::connect("127.0.0.1:8080").await.unwrap();
```

TcpStream用于客户端和服务端的通信，因此可对其进行读和写。读操作表示接收来自对端发送过来的数据，写操作表示将数据通过TCP连接发送给对端。但是，通常会使用`tokio::io::AsyncReadExt`和`tokio::io::AsyncWriteExt`提供的读写API来读写TcpStream，因尚未介绍`tokio::io`，因此先跳过相关的读写操作。

TcpStream本身也提供了和读写相关的一些api：
- readable(): 等待TcpStream有数据可读  
- writable(): 等待TcpStream可写入数据  
- ready(): 类似Linux的select系统调用，注册可读、可写、读写关闭等事件后等待这些事件的出现  
- try_read(): 尝试以不等待的方式读取TcpStream  
- try_read_buf(): 尝试以不等待的方式读取TcpStream，并将读取成功的数据追加到给定的buf中  
   - 和try_read()不同的是，try_read()每次读取数据后都会从前向后覆盖buf的字节，而try_read_buf()则是将读取的数据追加到buf的尾部  
- try_read_vectored(): 尝试以不等待的方式读取TcpStream，并将读取成功的数据分别填充到给定的一个或多个buf中  
   - 例如，给定了两个64K大小的buf，读取了100K数据，则前64K填充到第一个buf中，剩余的36K填充到第二个buf中  
- try_write(): 尝试以不等待的方式写入TcpStream  
- try_write_vectored(): 尝试以不等待的方式写入TcpStream，写入的数据源来自于给定的一个或多个buf  
- peek(): 从TcpStream中读取数据，但不消费TcpStream中本次读取的数据。即，peek后还可以再次读取这部分数据  
- split(): 将TcpStream的读和写进行分离，得到的读、写两端不可跨线程(或任务)  
- into_split(): 将TcpStream的读和写进行分离，得到的读、写两端可跨线程(或任务)  

稍后将简单介绍这些和读写相关的API的基本用法。

除了以上和IO相关的API，TcpSteam还提供了几个TCP连接选项设置的API： 
- set_linger(): 修改TCP连接的`SO_LINGER`选项。在关闭连接时如果仍有未发送数据(比如仍然在缓冲等待着更多数据进入)，设置该选项决定是否要等待一段时间(期待后续会将缓冲的数据发送出去)才允许关闭TCP连接。若不设置该选项，则默认不等待  
- linger(): 获取linger设置的值  
- set_nodelay(): 修改TCP连接的`TCP_NODELAY`选项。设置该选项后，写入TcpStream的数据都将立即发送，而不会缓冲并等待凑够数据后才发送  
- nodelay(): 是否设置了nodelay选项  

再来介绍TcpStream提供的和读写相关的API。

通常，读相关的操作(try_read、peek等)会结合readable()来使用，写相关的操作(try_write)会结合writable()来使用。但是注意，即便readable()、writable()的返回分别代表了可读和可写，但这个可读、可写的就绪事件并不能确保真的可读可写，因此读、写时要做好判断。

例如，readable()结合try_read(): 
```rust
use tokio::net::TcpStream;
use std::io;

#[tokio::main]
async fn main() {
    let stream = TcpStream::connect("127.0.0.1:8080").await.unwrap();
    let mut msg = vec![0; 1024];

    loop {
        // 等待可读事件的发生
        stream.readable().await.unwrap();

        // 即便readable()返回代表可读，但读取时仍然可能返回WouldBlock
        match stream.try_read(&mut msg) {
            Ok(n) => {    // 成功读取了n个字节的数据
                msg.truncate(n);
                break;
            }
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                continue;
            }
            Err(e) => {
                return;
            }
        }
    }

    println!("GOT = {:?}", msg);
}
```

当然，读写操作也可以结合ready()来使用，调用ready()时可注册感兴趣的事件，当注册的事件之一发生之后，ready()将返回Ready结构体，Ready结构体有一些布尔判断方法，用来判断某个事件是否发生。

例如：
```rust
use tokio::io::Interest;
use tokio::net::TcpStream;
use std::io;

#[tokio::main]
async fn main() {
    let stream = TcpStream::connect("127.0.0.1:8080").await.unwrap();

    loop {
        // 注册可读和可写事件，并等待事件的发生
        let ready = stream.ready(Interest::READABLE | Interest::WRITABLE).await.unwrap();

        // 如果注册的事件中，发生了可读事件，则执行如下代码
        if ready.is_readable() {
            let mut data = vec![0; 1024];
            match stream.try_read(&mut data) {
                Ok(n) => {
                    println!("read {} bytes", n);
                }
                Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                    continue;
                }
                Err(e) => {
                    return;
                }
            }
        }

        // 如果注册的事件中，发生了可写事件，则执行如下代码
        if ready.is_writable() {
            match stream.try_write(b"hello world") {
                Ok(n) => {
                    println!("write {} bytes", n);
                }
                Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                    continue
                }
                Err(e) => {
                    return;
                }
            }
        }
    }
}
```

`peek()`可读取TcpStream中的数据，但是和其它读取操作不同，peek()读取之后不会消费TcpStream中的数据。
```rust
use tokio::net::TcpStream;
use tokio::io::AsyncReadExt;

#[tokio::main]
async fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await.unwrap();
    let mut b1 = [0; 10];
    let mut b2 = [0; 10];

    let n = stream.peek(&mut b1).await.unwrap();
    let n1 = stream.read(&mut b2[..n]).await.unwrap();
}
```

比较关键的是`split()`方法。TCP连接是全双工通信的，无论是TCP连接的客户端还是服务端，每一端都可以进行读操作和写操作。为了方便描述，此处将其称为读端和写端。即，客户端有读端和写端，服务端也有读端和写端。

通过TcpStream，可进行读操作，也可以进行写操作，正如前面几个示例代码所示。但是，通过TcpStream同时进行读写有时候会很麻烦，甚至无解。很多时候，需要将TcpStream的读端和写端进行分离，然后将分离的读、写两端放进独立的异步任务中去执行读或写操作(此时需跨线程)，即一个线程(或异步任务)负责读，另一个线程(或异步任务)负责写。

split()和into_split()正是用来分离TcpStream的读写两端的。

split()可将TcpStream分离为ReadHalf和WriteHalf，ReadHalf用于读，WriteHalf用于写。
```rust
let mut conn = TcpStream::connect("127.0.0.1:8888").await.unwrap();
let (mut read_half, mut write_half) = conn.split();
```

split()并没有真正将TcpStream的读写两端进行分离，仅仅只是引用TcpStream中的读端和写端。因此，split()得到的读写两端只能在当前任务中进行读写操作，不允许跨线程跨任务。

into_split()是split()的owned版，分离后可得到OwnedReadHalf和OwnedWriteHalf。它是真正地分离TcpStream的读写两端，它会消费掉TcpStream。OwnedReadHalf和OwnedWriteHalf可跨任务进行读写操作。

```rust
let conn = TcpStream::connect("127.0.0.1:8888").await.unwrap();
let (mut read_half, mut write_half) = conn.into_split();
```

请记住TcpStream的`split()`和`into_split()`方法，这两个方法在tokio网络编程时非常常用。


