## tracing简要说明

- tracing可以记录结构化的日志，可以按区间(span)记录日志，例如一个函数可以作为一个区间单元，也可以自行指定何时进入(span.enter())区间单元  
- tracing有`TRACE DEBUG INFO WARN ERROR`共5个日志级别，其中TRACE是最详细的级别  
- tracing crate提供了最基本的核心功能：  
   - span：区间单元，具有区间的起始时间和区间的结束位置，是一个有时间跨度的区间  
   - span!()创建一个span区间，span.enter()表示进入span区间，drop span的时候退出span区间  
   - event: 每一次事件，都是一条记录，也可以看作是一条日志  
   - event!()记录某个指定日志级别的日志信息，`event!(Level::INFO, "something happened!");`  
   - trace!() debug!() info!() warn!() error!()，是event!()的语法糖，可以无需再指定日志级别  
- 记录日志时，可以记录结构化数据，以`key=value`的方式提供和记录。例如：`trace!(num = 33, "hello world")`，将记录为`"num = 33 hello worl"`。支持哪些格式，参考<https://docs.rs/tracing/latest/tracing/index.html#recording-fields>  
- tracing crate自身不会记录日志，它只是发出event!()或类似宏记录的日志，发出日志后，还需要通过tracing subscriber来收集  
- 在可执行程序(例如main函数)中，需要初始化subscriber，而在其它地方(如库或函数中)，只需使用那些宏来发出日志即可。发日志和收集记录日志分开，使得日志的处理逻辑非常简洁  
- 初始化subscriber的时候，可筛选收集到的日志(例如指定过滤哪些级别的日志)、格式化收集到的日志(例如修改时间格式)、指定日志的输出位置，等等  
- 默认清空下，subscribe的默认输出位置是标准输出，但可以在初始化时改变目标位置。如果需要写入文件，可使用tracing_appender crate  
   - 需注意，使用tracing_appender时，因为是先缓冲日志信息，而不是直接写入文件。他要求必须在main()函数中使用Guard，否则guard被丢弃将不会写入任何信息到文件   

## 示例

Cargo.toml：

```toml
[package]
name = "log"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
chrono = "0.4.19"
tracing = "0.1.29"
tracing-appender = "0.2.0"
tracing-subscriber = { version = "0.3.3" }
```
src/main.rs：
```rust
use chrono::Local;
use std::io;
use tracing::*;
use tracing_subscriber::fmt::format::Writer;
use tracing_subscriber::{self, fmt::time::FormatTime};

// 用来格式化日志的输出时间格式
struct LocalTimer;

impl FormatTime for LocalTimer {
    fn format_time(&self, w: &mut Writer<'_>) -> std::fmt::Result {
        write!(w, "{}", Local::now().format("%FT%T%.3f"))
    }
}

// 通过instrument属性，直接让整个函数或方法进入span区间，且适用于异步函数async fn fn_name(){}
// 参考：https://docs.rs/tracing/latest/tracing/attr.instrument.html
// #[tracing::instrument(level = "info")]
#[instrument]
fn test_trace(n: i32) {
    // #[instrument]属性表示函数整体在一个span区间内，因此函数内的每一个event信息中都会额外带有函数参数
    // 在函数中，只需发出日志即可
    event!(Level::TRACE, answer = 42, "trace2: test_trace");
    trace!(answer = 42, "trace1: test_trace");
    info!(answer = 42, "info1: test_trace");
}

// 在可执行程序中，需初始化tracing subscriber来收集、筛选并按照指定格式来记录日志
fn main() {
    // 直接初始化，采用默认的Subscriber，默认只输出INFO、WARN、ERROR级别的日志
    // tracing_subscriber::fmt::init();

    // 使用tracing_appender，指定日志的输出目标位置
    // 参考: https://docs.rs/tracing-appender/0.2.0/tracing_appender/
    // 如果不是在main函数中，guard必须返回到main()函数中，否则不输出任何信息到日志文件
    let file_appender = tracing_appender::rolling::daily("/tmp", "tracing.log");
    let (non_blocking, _guard) = tracing_appender::non_blocking(file_appender);

    // 设置日志输出时的格式，例如，是否包含日志级别、是否包含日志来源位置、设置日志的时间格式
    // 参考: https://docs.rs/tracing-subscriber/0.3.3/tracing_subscriber/fmt/struct.SubscriberBuilder.html#method.with_timer
    let format = tracing_subscriber::fmt::format()
        .with_level(true)
        .with_target(true)
        .with_timer(LocalTimer);

    // 初始化并设置日志格式(定制和筛选日志)
    tracing_subscriber::fmt()
        .with_max_level(Level::TRACE)
        .with_writer(io::stdout) // 写入标准输出
        .with_writer(non_blocking) // 写入文件，将覆盖上面的标准输出
        .with_ansi(false)  // 如果日志是写入文件，应将ansi的颜色输出功能关掉
        .event_format(format)
        .init();  // 初始化并将SubScriber设置为全局SubScriber

    test_trace(33);
    trace!("tracing-trace");
    debug!("tracing-debug");
    info!("tracing-info");
    warn!("tracing-warn");
    error!("tracing-error");
}
```

输出(上面设置了输出到文件/tmp/tracing.log)：

```
// 其中出现的log，是target
2021-12-01T15:09:55.797 TRACE test_trace{n=33}: log: trace1: test_trace answer=42
2021-12-01T15:09:55.797 TRACE test_trace{n=33}: log: trace2: test_trace answer=42
2021-12-01T15:09:55.797 TRACE log: tracing-trace
2021-12-01T15:09:55.797 DEBUG log: tracing-debug
2021-12-01T15:09:55.797  INFO log: tracing-info
2021-12-01T15:09:55.797  WARN log: tracing-warn
2021-12-01T15:09:55.797 ERROR log: tracing-error
```

## 不同区域使用不同日志

有时候有需求区分不同的日志，或者不同区域使用不同格式的日志。例如：

```rust
// 先创建一个SubScriber，准备作为默认的全局SubScriber
// finish()将完成SubScriber的构建，返回一个SubScriber
let default_logger = tracing_subscriber::fmt().with_max_level(Level::DEBUG).finish();

// 这段代码不记录任何日志，因为还未开启全局SubScriber
{
    info!("nothing will log");
}

// 从此开始，将default_logger设置为全局SubScriber
default_logger.init(); 

// 创建一个无颜色显示的且只记录ERROR级别的SubScriber
let tmp_logger = tracing_subscriber::fmt()
    .with_ansi(false)
    .with_max_level(Level::ERROR)
    .finish();
// 使用with_default，可将某段代码使用指定的SubScriber而非全局的SubScriber进行日志记录
tracing::subscriber::with_default(tmp_logger, || {
    error!("log with tmp_logger, only log ERROR logs");
});
info!("log with Global logger");
```

因为`init()`方法会调用`set_global_default()`，使得指定的Subscriber被设置为所有线程的全局Subscriber，所有的日志记录都会被该全局Subscriber给接收。如果想要在某个局部位置改变日志记录的目标(Subscriber)，甚至在某个位置丢弃所有日志记录，可以参考如下方式：

```rust
// 设置全局Subscriber，从WARN级别开始记录
tracing_subscriber::fmt()
    .with_max_level(tracing::Level::WARN)
    .init();

// 创建一个不调用init()方法的Subscriber，从TRACE级别开始记录
let sc = tracing_subscriber::fmt()
    .with_max_level(Level::TRACE)
    .finish();
// 指定局部范围的Subscriber
tracing::subscriber::with_default(sc, || {
    // 即便全局Subscriber是WARN级别的，但这条记录仍然会被记录
    tracing::info!("This will be logged to stdout");
});

// NoSubscriber是不记录任何日志记录的Subscriber，它会丢弃所有数据
let sc = tracing::subscriber::NoSubscriber::default();
tracing::subscriber::with_default(sc, || {
    // 不会被记录
    tracing::info!("This will be logged to stdout");
});
```

## 不同crate使用不同级别的日志

有时候我们自己想设置日志级别为debug来调试我们自己的程序，但这也会把其它所引用依赖的crate中的debug的日志输出出来。特别是有些时候其它crate的输出内容非常非常多，这很不友好。

通过开启tracing-subscriber的`env-filter`特性，可以按crate来设置不同日志级别。

```toml
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

然后设置：

```rust
tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .with_target(false)
        .with_file(true)
        .with_line_number(true)
        .with_ansi(false)
        .init();
```

使用`EnvFilter::from_default_env()`后，将读取`RUST_LOG`环境变量来设置日志级别。

可以是简单的格式，也可以是复杂的格式。假设我们自己的crate名叫`crate_a`，引用了一个名为`crate_b`的包。示例如下：

```bash
# 我们自己的以及所依赖的所有crate的日志级别都设置为debug
RUST_LOG=debug cargo run --release

# 关闭其它所有crate的输出，只设置自己的crate的级别为debug
RUST_LOG=none,crate_a=debug cargo run --release

# 默认级别为info，但我们自己的crate为debug
RUST_LOG=info,crate_a=debug cargo run --release

# 默认级别为debug，但关闭crate_b的日志
RUST_LOG=debug,crate_b=off cargo run --release
```

## 修改日志记录的时区

默认情况下，tracing-subscriber记录的时间是UTC时间，而不会主动输出为本机设置的时区对应的时间。但是对于东八区的我们来说，希望记录的是东八区的时间。

前面示例中提供了一种修改为本地时间的方式，并且是基于chrono crate而不是默认的time crate来设置tracing-subscriber的日志时间的时区和格式：

```rust
use tracing_subscriber::{
    self,
    fmt::{format::Writer, time::FormatTime},
};

// 用来格式化日志的输出时间格式
struct LocalTimer;

impl FormatTime for LocalTimer {
    fn format_time(&self, w: &mut Writer<'_>) -> std::fmt::Result {
        write!(w, "{}", chrono::Local::now().format("%FT%T%.3f"))
    }
}

fn main() {
    tracing_subscriber::fmt().with_timer(LocalTimer).init();

    tracing::info!("hello world");
}
```

但注意，使用`chrono::Local::now()`的效率相对会差一些，因为每次获取时间都要探测本机的时区。因此可改进为使用Offset的方式，明确指定时区，无需探测：

```rust
struct LocalTimer;

const fn east8() -> Option<chrono::FixedOffset> {
    chrono::FixedOffset::east_opt(8 * 3600)
}

impl FormatTime for LocalTimer {
    fn format_time(&self, w: &mut Writer<'_>) -> std::fmt::Result {
        let now = chrono::Utc::now().with_timezone(&east8().unwrap());
        write!(w, "{}", now.format("%FT%T%.3f"))
    }
}
```

当然，还有其它一些方式设置`tracing-subscriber`日志时间的时区和格式。例如可以通过它的`fmt::time::OffsetTime`指定时区，不过它使用的是time crate，因此如果记录非UTC时区，则先设置Cargo.toml：

```toml
tracing-subscriber = {version = "0.3", features = ["time"]}
time = { version = "0.3", features = ["macros"] }
```

然后设置日志时间的时区和格式：

```rust
use time::macros::{format_description, offset};
use tracing_subscriber::fmt::time::OffsetTime;

fn main() {
    let time_fmt = format_description!("[year]-[month]-[day]T[hour]:[minute]:[second].[subsecond digits:3]");
    let timer = OffsetTime::new(offset!(+8), time_fmt);
    tracing_subscriber::fmt().with_timer(timer).init();

    tracing::info!("offset: {}, hello world", offset);
}
```
