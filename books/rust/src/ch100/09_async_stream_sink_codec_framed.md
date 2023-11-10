# 使用Async Stream和Sink以及codec Framed

在前面介绍异步IO的时候，相关的读写操作都非常底层，要么直接操作字节，要么直接操作更高一层的Buffer，这两种方式都没有明确实际想要读和写的内容是什么，只是最原始的没有意义的字节或字符串。这种原始的操作方式比较底层，也容易出错，更不方便后期扩展和维护。

`tokio_util::codec`的作用是按照提前制定好的数据格式设计出对应的数据结构，之后直接以该数据结构为读写的操作单位。换句话说，codec其实就是编码和解码的作用，就像serde的角色一样。

而Async Stream和Sink则是相比于直接读写更底层字节字符串而言要更高一层的用来读写有具体意义的数据的工具。codec将AsyncRead和AsyncWrite转换为Stream和Sink，并使得Stream和Sink可以以Frame为读写单位进行读写操作。

## tokio、futures、futures_util、futures_core之间的关系

在开始介绍async Stream和Sink之前，有必要先理解清楚这几个库的关系。

在很多地方都看到有人疑惑tokio与futures的关系，大概是因为大家学的第一个Rust异步库是tokio，但却在不少示例和代码中发现引入了futures中的东西，于是产生这种疑惑。看这几个库的文档首页即可找到答案。

这是tokio_stream中的pub use：

```rust
pub use futures_core::Stream;	
```

这是futures中的pub use：

```rust
pub use futures_core::future::Future;	
pub use futures_core::future::TryFuture;	
pub use futures_util::future::FutureExt;	
pub use futures_util::future::TryFutureExt;	
pub use futures_core::stream::Stream;	
pub use futures_core::stream::TryStream;	
pub use futures_util::stream::StreamExt;	
pub use futures_util::stream::TryStreamExt;	
pub use futures_sink::Sink;	
pub use futures_util::sink::SinkExt;	
pub use futures_io::AsyncBufRead;	
pub use futures_io::AsyncRead;	
pub use futures_io::AsyncSeek;	
pub use futures_io::AsyncWrite;	
pub use futures_util::AsyncBufReadExt;	
pub use futures_util::AsyncReadExt;	
pub use futures_util::AsyncSeekExt;	
pub use futures_util::AsyncWriteExt;
```

这是futures_util中的pub use：

```rust
// 查看futures_util的源码，不难发现Future、Stream、Sink等
// 都是futures_core中对应类型的重新导出
pub use crate::future::Future;	
pub use crate::future::FutureExt;	
pub use crate::future::TryFuture;	
pub use crate::future::TryFutureExt;	
pub use crate::stream::Stream;	
pub use crate::stream::StreamExt;	
pub use crate::stream::TryStream;	
pub use crate::stream::TryStreamExt;	
pub use crate::sink::Sink;	
pub use crate::sink::SinkExt;	
pub use crate::io::AsyncBufRead;	
pub use crate::io::AsyncBufReadExt;	
pub use crate::io::AsyncRead;	
pub use crate::io::AsyncReadExt;	
pub use crate::io::AsyncSeek;	
pub use crate::io::AsyncSeekExt;	
pub use crate::io::AsyncWrite;	
pub use crate::io::AsyncWriteExt;
```

显然，Stream相关的类型和Sink相关的类型，其实都来自于futures_core。因此，在需要使用到Stream相关类型或Sink相关类型的代码文件中，引入以上任意一个库都行。


## Async Stream Trait

Stream Trait用于读操作，它模拟Rust标准库的Iterator，可进行迭代式读取和迭代式操作，非常具有Rust的风味。

例如：

```rust
use tokio_stream::{self as stream, StreamExt};

#[tokio::main]
async fn main() {
    let mut stream = stream::iter(vec![0, 1, 2]);

    while let Some(value) = stream.next().await {
        println!("Got {}", value);
    }
}
```

上面示例中通过`tokio_stream::iter()`创建了一个Stream，然后通过不断调用Stream的next()方法来读取Stream中的下一个数据。需要注意的是，目前不能对Stream执行`for value in stream{}`的迭代操作，只能不断显式地调用next()方法来读取。比如可以使用下面两种循环读取的方式。

```rust
while let Some(value) = s.next().await {}

loop {
  match s.next().await {
    Some(value) => {}
    None => break;
  }
}
```

有很多场景下需要手动提供Stream，上面使用的`tokio_stream::iter()`是一个很常用的创建Stream的方法，它返回`tokio_stream::Iter`，该类型实现了Stream。

在`tokio_stream::wrappers`中还提供了一些对tokio中的类型封装，并将封装后的类型实现了Stream，因此它们都可以直接作为Stream使用。例如，`wrappers::ReceiverStream`是对`tokio::sync::mpsc::Receiver`的封装，并实现了Stream。参考<https://docs.rs/tokio-stream/latest/tokio_stream/wrappers/index.html>。

最后，还有一个比较常用的生成Stream的方式的是使用`async-stream` crate提供的宏，它可以返回`impl Stream`类型。

## Async Sink Trait

Sink Trait用于写操作。Sink的意思是下沉、沉入，表示可直接通过Sink方便简单地写入有意义的数据，Sink会自动将该数据转换为更底层的字节传输出去(事物放在水面即可，它会自动下沉到底层)。

这就是使用Sink过程中所需要理解的全部啦。

## StreamExt和SinkExt

Async Stream和Async Sink都只提供了非常原始的方法，更多时候我们会使用StreamExt和SinkExt中提供的扩展方法来简化读写操作。其中tokio_stream提供了StreamExt，`futures`和`futures_util`中提供了StreamExt和SinkExt，因此需要引入相关库才能使用相关的扩展方法。

关于StreamExt提供的用法，可参考官方手册(<https://docs.rs/tokio-stream/latest/tokio_stream/trait.StreamExt.html>)，用法和Iterator没有太大区别，因此不多赘述。

有关SinkExt提供的方法，有必要介绍主要的几个写入方法：

- send()：写入Sink并flush  
- feed()：写入Sink但不flush  
- flush()：将已经写入Sink的数据flush  
- send_all()：将给定的Stream中的零或多个数据全部写入Sink并一次或多次flush(自动决定何时flush)  

用法示例如下：下面代码中的sink是一个Sink，其传输的有意义的数据格式是字符串，msg类型是字符串。

```rust
// 方式一: feed() + flush()
sink.feed(msg).await.unwrap();
sink.flush().await.unwrap();

// 方式二: send() == feed() + flush()
sink.send(msg).await.unwrap();

// 方式三：send_all()，一次发送一条或多条，但只允许futures::TryStream作为参数，
// 所以要用到futures crates来构建Stream。例如：
// let msgs = vec![Ok("hello world".to_string()), Ok("HELLO WORLD".to_string())];
let msgs = vec!["hello world".to_string(), "HELLO WORLD".to_string()];
let mut ss = futures_util::stream::iter(msgs).map(Ok);
sink.send_all(&mut ss).await.unwrap();
```

## codec和Framed

`tokio_util::codec`可以将实现了AsyncRead/AsyncWrite的结果转换为Stream/Sink，得到的Stream和Sink可以以帧Frame为读写单位。

其中：

- 实现了`codec::Decoder`的类型可以转换AsyncRead为FramedRead，FramedRead实现了Stream，因此FramedRead可进行按帧读取的操作  
- 实现了`codec::Encoder`的类型可以转换AsyncWrite为FramedWrite，FramedWrite实现了Sink，因此FramedWrite可进行按帧写入的操作  
- 同时实现了Decoder和Encoder的类型可转换为Framed(当然，也可以转换为FramedRead或FramedWrite)，Framed既是Stream，也是Sink，可同时进行以帧为单位的读写操作。Framed可通过`split()`方法分离出独立的Stream和Sink分别进行读写    

codec还提供了几个常用的已经同时实现了Decoder、Encoder的类型：

- LinesCodec：以行为单位的帧  
- AnyDelimiterCodec：以指定分隔符为单位的帧  
- BytesCodec：以字节为单位的帧  

因为它们同时实现了Decoder和Encoder，因此它们可转换为FramedRead、FramedWrite或Framed。

将Decoder、Encoder转换为对应Framed的方式参考如下：

```rust
// T是实现了AsyncRead的类型，例如TcpStream、File等  
// D是实现了Decoder的类型，例如LinesCodec、BytesCodec等
let framed_reader = FramedRead::new(T, D);

// T是实现了AsyncWrite的类型，例如TcpStream、File等  
// E是实现了Encoder的类型，例如LinesCodec、BytesCodec等
let framed_writer = FramedWrite::new(T, E);

// T是实现了AsyncRead + AsyncWrite的类型，例如TcpStream、File等  
// U是实现了Encoder + Decoder的类型，例如LinesCodec、BytesCodec等
let framed = Framed::new(T, U);

// T是实现了AsyncRead + AsyncWrite的类型，例如TcpStream、File等  
// 只要实现了Decoder，例如LinesCodec、BytesCodec等，就可以通过它的framed()方法生成Framed
let framed = LinesCodec::new().framed(T);
```

例如，通过LinesCodec使TcpStream能够按行进行读写操作，下面是Server端按行读写客户端的一个示例：

```rust
use futures_util::stream::{SplitSink, SplitStream};
use futures_util::{SinkExt, StreamExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::mpsc;
use tokio_util::codec::{Framed, LinesCodec};

type LineFramedStream = SplitStream<Framed<TcpStream, LinesCodec>>;
type LineFramedSink = SplitSink<Framed<TcpStream, LinesCodec>, String>;

#[tokio::main]
async fn main() {
    let server = TcpListener::bind("127.0.0.1:8888").await.unwrap();
    while let Ok((client_stream, _client_addr)) = server.accept().await {
        tokio::spawn(async move {
            process_client(client_stream).await;
        });
    }
}

async fn process_client(client_stream: TcpStream) {
  	// 将TcpStream转换为Framed
    let framed = Framed::new(client_stream, LinesCodec::new());
    // 将Framed分离，可得到独立的读写端
    let (frame_writer, frame_reader) = framed.split::<String>();
    // 当Reader从客户端读取到数据后，发送到通道中，
    // 另一个异步任务读取该通道，从通道中读取到数据后，将内容按行写给客户端
    let (msg_tx, msg_rx) = mpsc::channel::<String>(100);

    // 负责读客户端的异步子任务
    let mut read_task = tokio::spawn(async move {
        read_from_client(frame_reader, msg_tx).await;
    });

    // 负责向客户端写行数据的异步子任务
    let mut write_task = tokio::spawn(async move {
        write_to_client(frame_writer, msg_rx).await;
    });

    // 无论是读任务还是写任务的终止，另一个任务都将没有继续存在的意义，因此都将另一个任务也终止
    if tokio::try_join!(&mut read_task, &mut write_task).is_err() {
        eprintln!("read_task/write_task terminated");
        read_task.abort();
        write_task.abort();
    };
}

async fn read_from_client(mut reader: LineFramedStream, msg_tx: mpsc::Sender<String>) {
    loop {
        match reader.next().await {
            None => {
                println!("client closed");
                break;
            }
            Some(Err(e)) => {
                eprintln!("read from client error: {}", e);
                break;
            }
            Some(Ok(str)) => {
                println!("read from client. content: {}", str);
                // 将内容发送给writer，让writer响应给客户端，
                // 如果无法发送给writer，继续从客户端读取内容将没有意义，因此break退出
                if msg_tx.send(str).await.is_err() {
                    eprintln!("receiver closed");
                }
            }
        }
    }
}

async fn write_to_client(mut writer: LineFramedSink, mut msg_rx: mpsc::Receiver<String>) {
    while let Some(str) = msg_rx.recv().await {
        if writer.send(str).await.is_err() {
            eprintln!("write to client failed");
            break;
        }
    }
}
```

## 实现codec的Encoder和Decoder

严格来说，`tokio_util::codec`已经提供的几种Codec中，只有`LinesCodec`和`AnyDelimiterCodec`是比较常用且通用的，而提供的`BytesCodec`只在某些特殊场景下适合使用。

如果使用二进制数据进行通信(LinesCodec自然是以字符串方式进行通信的)，很可能需要自己去实现codec的Decoder和Encoder。

- Encoder：将指定的Rust类型转换为二进制字节数据帧  
- Decoder：将二进制字节数据帧转换为指定的Rust类型  

一个简单、通用且常用的二进制通信协议格式是`| data_len | data |`。即：

- 在编码时(Encoder)，先计算待发送的二进制数据data的长度，并将长度大小放在帧首，实际数据放在长度的后面，这是一个完整的帧  
- 在解码时(Decoder)，先读取长度大小，根据读取的长度大小再先后读取指定数量的字节，从而读取一个完整的帧，再将其转换为指定的数据类型  

下面是一个完整的示例。

先定义客户端和服务端通信的数据类型：

```rust
/// 请求
#[derive(Debug, Serialize, Deserialize)]
pub struct Request {
    pub sym: String,
    pub from: u64,
    pub to: u64,
}

/// 响应
#[derive(Debug, Serialize, Deserialize)]
pub struct Response(pub Option<Klines>);

/// 对请求和响应的封装，之后客户端和服务端都将通过Sink和Stream来基于该类型通信
#[derive(Debug, Serialize, Deserialize)]
pub enum RstResp {
    Request(Request),
    Response(Response),
}
```

再自定义一个Codec并实现`codec::Encoder`和`codec::Decoder`。

```rust
/// 自己定义一个Codec，
/// 并实现Encoder和Decoder，完成 RstResp => &[u8] => RstResp 之间的转换
pub struct RstRespCodec;
impl RstRespCodec {
    /// 最多传送1G数据
    const MAX_SIZE: usize = 1024 * 1024 * 1024 * 8;
}

/// 实现Encoder，将RstResp转换为字节数据
/// 对于codec而言，直接将二进制数据写入 `dst: &mut BytesMut` 即可
impl codec::Encoder<RstResp> for RstRespCodec {
    type Error = bincode::Error;
    // 本示例中使用bincode将RstResp转换为&[u8]，也可以使用serde_json::to_vec()，前者效率更高一些
    fn encode(&mut self, item: RstResp, dst: &mut BytesMut) -> Result<(), Self::Error> {
        let data = bincode::serialize(&item)?;
        let data = data.as_slice();
      
        // 要传输的实际数据的长度
        let data_len = data.len();
        if data_len > Self::MAX_SIZE {
            return Err(bincode::Error::new(bincode::ErrorKind::Custom(
                "frame is too large".to_string(),
            )));
        }
      
        // 最大传输u32的数据(可最多512G)，
        // 表示数据长度的u32数值占用4个字节
        dst.reserve(data_len + 4);
      
        // 先将长度值写入dst，即帧首，
        // 写入的字节序是大端的u32，读取时也要大端格式读取，
        // 也有小端的方法`put_u32_le()`，读取时也得小端读取
        dst.put_u32(data_len as u32);
      
        // 再将实际数据放入帧尾
        dst.extend_from_slice(data);
        Ok(())
    }
}

/// 实现Decoder，将字节数据转换为RstResp
impl codec::Decoder for RstRespCodec {
    type Item = RstResp;
    type Error = std::io::Error;
    // 从不断被填充的Bytes buf中读取数据，并将其转换到目标类型
    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error> {
        let buf_len = src.len();

        // 如果buf中的数据量连长度声明的大小都不足，则先跳过等待后面更多数据的到来
        if buf_len < 4 { return Ok(None); }

        // 先读取帧首，获得声明的帧中实际数据大小
        let mut length_bytes = [0u8; 4];
        length_bytes.copy_from_slice(&src[..4]);
        let data_len = u32::from_be_bytes(length_bytes) as usize;
        if data_len > Self::MAX_SIZE {
            return Err(std::io::Error::new(
                std::io::ErrorKind::InvalidData,
                format!("Frame of length {} is too large.", data_len),
            ));
        }

        // 帧的总长度为 4 + frame_len
        let frame_len = data_len + 4;

        // buf中数据量不够，跳过，并预先申请足够的空闲空间来存放该帧后续到来的数据
        if buf_len < frame_len {
            src.reserve(frame_len - buf_len);
            return Ok(None);
        }

        // 数据量足够了，从buf中取出数据转编成帧，并转换为指定类型后返回
        // 需同时将buf截断(split_to会截断)
        let frame_bytes = src.split_to(frame_len);
        match bincode::deserialize::<RstResp>(&frame_bytes[4..]) {
            Ok(frame) => Ok(Some(frame)),
            Err(e) => Err(std::io::Error::new(std::io::ErrorKind::InvalidData, e)),
        }
    }
}
```

最后，通过使用`Sink`和`Stream`，让客户端和服务端基于`RstResp`来通信。读写RstResp的代码大概如下：

```rust
let framed = Framed::new(client_stream, RstRespCodec);
let (frame_writer, frame_reader) = framed.split::<RstResp>();

// 负责写的
let resp = RstResp::Response(resp);
if frame_writer.send(resp).await.is_err() {
    error!("write failed");
}

// 负责读的
loop {
    match frame_reader.next().await {
        None => {
            debug!("peer closed");
            break;
        }
        Some(Err(e)) => {
            error!("read peer error: {}", e);
            break;
        }
        Some(Ok(req_resp)) => {
            match req_resp {
                RstResp::Request(_) => _,
                RstResp::Response(_) => _,
            };
        }
    }
}
```


## 客户端和服务端的Codec设计思路

当然，因读取的客户端请求类型(ReqType)和写给客户端的响应类型(RespType)往往是不同的类型，一种处理方式是通过enum合并请求和响应，然后为合并后的类型做Codec的解析。

```rust
#[derive(Serialize, Deserialize)]
enum MyType {
  ReqType(Req),
  RespType(Resp),
}

pub struct MyTypeCodec;

impl Encoder<MyType> for MyTypeCodec {
  type Error = ;
  fn encode(&mut self, item: MyType, dst: &mut BytesMut) -> Result<(), Self::Error> {
        ...
  }
}

impl Decoder for MyTypeCodec {
  type Item = MyType;
  type Error = ;
  fn decode(&mut self, src: &mut BytesMut)->Result<Option<Self::Item>,Self::Error> {
      ...
  }
}
```

然后将一个TcpStream(以TcpStream为例)构造成MyTypeCodec的FramedRead端和FramedWrite端：

```rust
let framed = Framed::new(tcp_stream, MyTypeCodec::new());
let (frame_writer, mut frame_reader) = framed.split::<MyType>();
```

这样处理后，`framed_reader`端读取时只处理MyType的ReqType分支，RespType端是多余的。

另一种处理方式是，单独为客户端的请求Req和客户端的响应Resp做各自的Codec解析，然后将TcpStream的读端构造为ReqCodec端，将写端构造成RespCodec端。

例如：

```rust
/// 为请求类型Req做的Codec解析
pub struct ReqCodec;
impl Encoder<Req> for ReqCodec {
  type Error = ;
  fn encode(&mut self, item: Req, dst: &mut BytesMut) -> Result<(), Self::Error> {
        ...
  }
}
impl Decoder for ReqCodec {
  type Item = Req;
  type Error = ;
  fn decode(&mut self, src: &mut BytesMut)->Result<Option<Self::Item>,Self::Error> {
      ...
  }
}

/// 为响应类型Resp做的Codec解析
pub struct RespCodec;
impl Encoder<Resp> for RespCodec {
  type Error = ;
  fn encode(&mut self, item: Resp, dst: &mut BytesMut) -> Result<(), Self::Error> {
        ...
  }
}
impl Decoder for RespCodec {
  type Item = Resp;
  type Error = ;
  fn decode(&mut self, src: &mut BytesMut)->Result<Option<Self::Item>,Self::Error> {
      ...
  }
}

fn main(){
  // 将TcpStream的读端和写端分离，分别构造对应的Codec
  let (rd, wr) = stream.into_split();
  let mut frame_reader = FramedRead::new(rd, ReqCodec);
  let frame_writer = FramedWrite::new(wr, RespCodec);
}
```

个人觉得，第二种方式要更清晰一些，后续的处理代码也比第一种方式更简单一些。

