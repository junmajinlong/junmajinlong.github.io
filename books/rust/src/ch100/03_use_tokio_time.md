## 使用tokio Timer

本篇介绍tokio的计时器功能：Timer。

每一个异步框架都应该具备计时器功能，tokio的计时器功能在开启了`time`特性后可用。

```toml
tokio = {version = "1.13", features = ["rt", "rt-multi-thread", "time"]}
```

tokio的time模块包含几个功能：  
- `Duration`类型：是对`std::time::Duration`的重新导出，两者等价。它用于描述持续时长，例如睡眠3秒的3秒是一个时长，每隔3秒的3秒也是一个时长  
- `Instant`类型：从程序运行开始就单调递增的时间点，仅结合Duration一起使用。例如，此刻是处在某个时间点A，下一次(例如某个时长过后)，处在另一个时间点B，时间点B一定不会早于时间点A，即便修改了操作系统的时钟或硬件时钟，它也不会时光倒流的现象  
- `Sleep`类型：是一个Future，通过调用`sleep()`或`sleep_until()`返回，该Future本身不做任何事，它只在到达某个时间点(Instant)时完成  
- `Interval`类型：是一个流式的间隔计时器，通过调用`interval()`或`interval_at()`返回。Interval使用`Duration`来初始化，表示每隔一段时间(即指定的Duration时长)后就产生一个值  
- `Timeout`类型：封装异步任务，并为异步任务设置超时时长，通过调用`timeout()`或`timeout_at()`返回。如果异步任务在指定时长内仍未完成，则异步任务被强制取消并返回Error

### 时长: tokio::time::Duration

`tokio::time::Duration`是对`std::time::Duration`的Re-exports，它两完全等价，因此可在tokio上下文中使用任何一种Duration。

Duration类型描述了一种时长，该结构包含两部分：秒和纳秒。
```rust
pub struct Duration {
    secs: u64,
    nanos: u32,
}
```

可使用`Duration::new(Sec, Nano_sec)`来构建Duration。例如，`Duration::new(5, 30)`构建了一个5秒30纳秒的时长，即总共`5_000_000_030`纳秒。

如果Nano_sec部分超出了纳秒范围(1秒等于10亿纳秒)，将进位到秒单位上，例如第二个参数指定为500亿纳秒，那么会向秒部分加50秒。

注意，构建时长时，这两部分的值可能会超出范围，例如计算后的秒部分的值超出了u64的范围，或者计算得到了负数。对此，Duration提供了几种不同的处理方式。

特殊地，如果两个参数都指定为0，那么表示时长为0，可用`is_zero()`来检测某个Duration是否是0时长。0时长可用于上下文切换(yield)，例如sleep睡眠0秒，表示不用睡眠，但会交出CPU使得发生上下文切换。

还可以使用如下几种简便的方式构建各种单位的时长：
- Duration::from_secs(3)：3秒时长  
- Duration::from_millis(300)：300毫秒时长  
- Duration::from_micros(300)：300微秒时长  
- Duration::from_nanos(300)：300纳秒时长  
- Duration::from_secs_f32(2.3)：2.3秒时长  
- Duration::from_secs_f64(2.3)：2.3秒时长  

对于构建好的Duration实例`dur = Duration::from_secs_f32(2.3)`，可以使用如下几种方法方便地提取、转换它的秒、毫秒、微秒、纳秒。
- dur.as_secs()：转换为秒的表示方式，2  
- dur.as_millis(): 转换为毫秒表示方式，2300  
- dur.as_micros(): 转换为微秒表示方式，2_300_000  
- dur.as_nanos(): 转换为纳秒表示方式，2_300_000_000  
- dur.as_secs_f32(): 小数秒表示方式，2.3  
- dur.as_secs_f64(): 小数秒表示方式，2.3  
- dur.subsec_millis(): 小数部分转换为毫秒精度的表示方式，300  
- dur.subsec_micros(): 小数部分转换为微秒精度的表示方式，300_000  
- dur.subsec_nanos(): 小数部分转换为纳秒精度的表示方式，300_000_000  

Duration实例可以直接进行大小比较以及加减乘除运算：
- checked_add(): 时长的加法运算，超出Duration范围时返回None  
- checked_sub(): 时长的减法运算，超出Duration范围时返回None  
- checked_mul(): 时长的乘法运算，超出Duration范围时返回None  
- checked_div(): 时长的除法运算，超出Duration范围时(即分母为0)返回None  
- saturating_add()：饱和式的加法运算，超出范围时返回Duration支持的最大时长  
- saturating_mul()：饱和式的乘法运算，超出范围时返回Duration支持的最大时长  
- saturating_sub()：饱和式的减法运算，超出范围时返回0时长  
- mul_f32()：时长乘以小数，得到的结果如果超出范围或无效，则panic  
- mul_f64()：时长乘以小数，得到的结果如果超出范围或无效，则panic  
- div_f32()：时长除以小数，得到的结果如果超出范围或无效，则panic  
- div_f64()：时长除以小数，得到的结果如果超出范围或无效，则panic  

### 时间点: tokio::time::Instant

Instant用于表示时间点，主要用于两个时间点的比较和相关运算。

`tokio::time::Instant`是对`std::time::Instant`的封装，添加了一些对齐功能，使其能够适用于tokio runtime。

Instant是严格单调递增的，绝不会出现时光倒流的现象，即之后的时间点一定晚于之前创建的时间点。但是，tokio time提供了`pause()`函数可暂停时间点，还提供了`advance()`函数用于向后跳转到某个时间点。

`tokio::time::Instant::now()`用于创建代表此时此刻的时间点。Instant可以直接进行大小比较，还能执行`+`、`-`操作。
```rust
use tokio;
use tokio::time::Instant;
use tokio::time::Duration;

#[tokio::main]
async fn main() {
    // 创建代表此时此刻的时间点
    let now = Instant::now();
    
    // Instant 加一个Duration，得到另一个Instant
    let next_3_sec = now + Duration::from_secs(3);
    // Instant之间的大小比较
    println!("{}", now < next_3_sec);  // true
    
    // Instant减Duration，得到另一个Instant
    let new_instant = next_3_sec - Duration::from_secs(2);
    
    // Instant减另一个Instant，得到Duration
    // 注意，Duration有它的有效范围，因此必须是大的Instant减小的Instant，反之将panic
    let duration = next_3_sec - new_instant;
}
```

此外`tokio::time::Instant`还有以下几个方法：
- from_std(): 将`std::time::Instant`转换为`tokio::time::Instant`  
- into_std(): 将`tokio::time::Instant`转换为`std::time::Instant`  
- elapsed(): 指定的时间点实例，距离此时此刻的时间点，已经过去了多久(返回Duration)  
- duration_since(): 两个Instant实例之间相差的时长，要求`B.duration_since(A)`中的B必须晚于A，否则panic  
- checked_duration_since(): 两个时间点之间的时长差，如果计算返回的Duration无效，则返回None  
- saturating_duration_since(): 两个时间点之间的时长差，如果计算返回的Duration无效，则返回0时长的Duration实例  
- checked_add(): 为时间点加上某个时长，如果加上时长后是无效的Instant，则返回None  
- checked_sub(): 为时间点减去某个时长，如果减去时长后是无效的Instant，则返回None  

tokio顶层也提供了一个`tokio::resume()`方法，功能类似于`tokio::time::from_std()`，都是将`std::time::Instant::now()`保存为`tokio::time::Instant`。不同的是，后者用于创建tokio time Instant时间点，而`resume()`是让tokio的Instant的计时系统与系统的计时系统进行一次同步更新。

### 睡眠: tokio::time::Sleep

`tokio::time::sleep()`和`tokio::time::sleep_until()`提供tokio版本的睡眠任务：
```rust
use tokio::{self, runtime::Runtime, time};

fn main(){
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        // 睡眠2秒
        time::sleep(time::Duration::from_secs(2)).await;

        // 一直睡眠，睡到2秒后醒来
        time::sleep_until(time::Instant::now() + time::Duration::from_secs(2)).await;
    });
}
```

注意，`std::thread::sleep()`会阻塞当前线程，而tokio的睡眠不会阻塞当前线程，实际上tokio的睡眠在进入睡眠后不做任何事，仅仅只是立即放弃CPU，并进入任务轮询队列，等待睡眠时间终点到了之后被Reactor唤醒，然后进入就绪队列等待被调度。

可以简单理解异步睡眠：调用睡眠后，记录睡眠的终点时间点，之后在轮询到该任务时，比较当前时间点是否已经超过睡眠终点，如果超过了，则唤醒该睡眠任务，如果未超过终点，则不管。

注意，tokio的sleep的睡眠精度是毫秒，因此无法保证也不应睡眠更低精度的时间。例如不要睡眠100微秒或100纳秒，这时无法保证睡眠的时长。

下面是一个睡眠10微秒的例子，多次执行，会发现基本上都要1毫秒多，去掉执行指令的时间，实际的睡眠时长大概是1毫秒。另外，将睡眠10微秒改成睡眠100微秒或1纳秒，结果也是接近的。
```rust
use tokio::{self, runtime::Runtime, time};

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let start = time::Instant::now();
        // time::sleep(time::Duration::from_nanos(100)).await;
        // time::sleep(time::Duration::from_micros(100)).await;
        time::sleep(time::Duration::from_micros(10)).await;
        println!("sleep {}", time::Instant::now().duration_since(start).as_nanos());
    });
}
```

执行的多次，输出结果：
```
sleep 1174300
sleep 1202900
sleep 1161200
sleep 1393200
sleep 1306400
sleep 1285300
```

`sleep()`或`sleep_until()`都返回`time::Sleep`类型，它有3个方法可调用：  
- deadline(): 返回Instant，表示该睡眠任务的睡眠终点  
- is_elapsed(): 可判断此时此刻是否已经超过了该sleep任务的睡眠终点  
- reset()：可用于重置睡眠任务。如果睡眠任务未完成，则直接修改睡眠终点，如果睡眠任务已经完成，则再次创建睡眠任务，等待新的终点  

需要注意的是，`reset()`要求修改睡眠终点，因此Sleep实例需要是mut的，但这样会消费掉Sleep实例，更友好的方式是使用`tokio::pin!(sleep)`将sleep给pin在当前栈中，这样就可以调用`as_mut()`方法获取它的可修改版本。
```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

#[allow(dead_code)]
fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("start: {}", now());
        let slp = time::sleep(time::Duration::from_secs(1));
        tokio::pin!(slp);

        slp.as_mut().reset(time::Instant::now() + time::Duration::from_secs(2));

        slp.await;
        println!("end: {}", now());
    });
}
```

输出：
```
start: 2021-11-02 21:57:42
end: 2021-11-02 21:57:44
```

重置已完成的睡眠实例：
```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

#[allow(dead_code)]
fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("start: {}", now());
        let slp = time::sleep(time::Duration::from_secs(1));
        tokio::pin!(slp);
        
        //注意调用slp.as_mut().await，而不是slp.await，后者会move消费掉slp
        slp.as_mut().await;
        println!("end 1: {}", now());

        slp.as_mut().reset(time::Instant::now() + time::Duration::from_secs(2));

        slp.await;
        println!("end 2: {}", now());
    });
}
```

输出结果：
```
start: 2021-11-02 21:59:25
end 1: 2021-11-02 21:59:26
end 2: 2021-11-02 21:59:28
```

### 任务超时: tokio::time::Timeout

`tokio::time::timeout()`或`tokio::time::timeout_at()`可设置一个异步任务的完成超时时间，前者接收一个Duration和一个Future作为参数，后者接收一个Instant和一个Future作为参数。这两个函数封装异步任务之后，返回`time::Timeout`，它也是一个Future。

如果在指定的超时时间内该异步任务已完成，则返回该异步任务的返回值，如果未完成，则异步任务被撤销并返回Err。

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let res = time::timeout(time::Duration::from_secs(5), async {
            println!("sleeping: {}", now());
            time::sleep(time::Duration::from_secs(6)).await;
            33
        });

        match res.await {
            Err(_) => println!("task timeout: {}", now()),
            Ok(data) => println!("get the res '{}': {}", data, now()),
        };
    });
}
```

得到结果：
```
sleeping: 2021-11-03 17:12:33
task timeout: 2021-11-03 17:12:38
```

如果将睡眠6秒改为睡眠4秒，那么将得到结果：
```
sleeping: 2021-11-03 17:13:11
get the res '33': 2021-11-03 17:13:15
```

得到`time::Timeout`实例res后，可以通过`res.get_ref()`或者`res.get_mut()`获得Timeout所封装的Future的可变和不可变引用，使用`res.into_inner()`获得所封装的Future，这会消费掉该Future。

如果要取消Timeout的计时等待，直接删除掉Timeout实例即可。

### 间隔任务: tokio::time::Interval

`tokio::time::interval()`和`tokio::time::interval_at()`用于设置间隔性的任务。

- interval_at(): 接收一个Instant参数和一个Duration参数，Instant参数表示间隔计时器的开始计时点，Duration参数表示间隔的时长  
- interval(): 接收一个Duration参数，它在第一次被调用的时候立即开始计时  

注意，这两个函数只是定义了间隔计时器的起始计时点和间隔的时长，要真正开始让它开始计时，还需要调用它的`tick()`方法生成一个Future任务，并调用await来执行并等待该任务的完成。

例如，5秒后开始每隔1秒执行一次输出操作：
```rust
use chrono::Local;
use tokio::{self, runtime::Runtime, time::{self, Duration, Instant}};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        // 计时器的起始计时点：此时此刻之后的5秒后
        let start = Instant::now() + Duration::from_secs(5);
        let interval = Duration::from_secs(1);
        let mut intv = time::interval_at(start, interval);

        // 该计时任务"阻塞"，直到5秒后被唤醒
        intv.tick().await;
        println!("task 1: {}", now());

        // 该计时任务"阻塞"，直到1秒后被唤醒
        intv.tick().await;
        println!("task 2: {}", now());

        // 该计时任务"阻塞"，直到1秒后被唤醒
        intv.tick().await;
        println!("task 3: {}", now());
    });
}
```

输出结果：
```
before: 2021-11-03 18:52:14
task 1: 2021-11-03 18:52:19
task 2: 2021-11-03 18:52:20
task 3: 2021-11-03 18:52:21
```

上面定义的计时器，有几点需要说明清楚：  

1. interval_at()第一个参数定义的是计时器的开始时间，这样描述不准确，它表述的是最早都要等到这个时间点才开始计时。例如，定义计时器5秒之后开始计时，但在第一次tick()之前，先睡眠了10秒，那么该计时器将在10秒后才开始，但如果第一次tick之前只睡眠了3秒，那么还需再等待2秒该tick计时任务才会完成。  
2. 定义计时器时，要将其句柄(即计时器变量)声明为mut，因为每次tick时，都需要修改计时器内部的下一次计时起点。  
3. 不像其它语言中的间隔计时器，tokio的间隔计时器需要手动调用tick()方法来生成临时的异步任务。  
4. 删除计时器句柄可取消间隔计时器。

再看下面的示例，定义5秒后开始的计时器，但在第一次开始计时前，先睡眠10秒。

```rust
use chrono::Local;
use tokio::{
    self,
    runtime::Runtime,
    time::{self, Duration, Instant},
};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        let start = Instant::now() + Duration::from_secs(5);
        let interval = Duration::from_secs(1);
        let mut intv = time::interval_at(start, interval);

        time::sleep(Duration::from_secs(10)).await;
        intv.tick().await;
        println!("task 1: {}", now());
        intv.tick().await;
        println!("task 2: {}", now());
    });
}
```

输出结果：
```
before: 2021-11-03 19:00:10
task 1: 2021-11-03 19:00:20
task 2: 2021-11-03 19:00:20
```

注意输出结果中的task 1和task 2的时间点是相同的，说明第一次tick之后，并没有等待1秒之后再执行紧跟着的tick，而是立即执行之。

简单解释一下上面示例中的计时器内部的工作流程，假设定义计时器的时间点是19:00:10：  
- 定义5秒后开始的计时器intv，该计时器内部有一个字段记录着下一次开始tick的时间点，其值为19:00:15  
- 睡眠10秒后，时间点到了19:00:20，此时第一次执行`intv.tick()`，它将生成一个异步任务，执行器执行时发现此时此刻的时间点已经超过该计时器内部记录的值，于是该异步任务立即完成并进入就绪队列等待调度，同时修改计时器内部的值为19:00:16  
- 下一次执行tick的时候，此时此刻仍然是19:00:20，已经超过了该计时器内部的19:00:16，因此计时任务立即完成  

这是tokio Interval在遇到计时延迟时的默认计时策略，叫做`Burst`。tokio支持三种延迟后的计时策略。可使用`set_missed_tick_behavior(MissedTickBehavior)`来修改计时策略。

**1.Burst策略，冲刺型的计时策略，当出现延迟后，将尽量快地完成接下来的tick，直到某个tick赶上它正常的计时时间点**。

例如，5秒后开始的每隔1秒的计时器，第一次开始tick前睡眠了10秒，那么10秒后将立即进行如下几次tick，或者说瞬间完成如下几次tick：
- 第一次tick，它本该在第五秒的时候被执行  
- 第二次tick，它本该在第六秒的时候被执行  
- 第三次tick，它本该在第七秒的时候被执行  
- 第四次tick，它本该在第八秒的时候被执行  
- 第五次tick，它本该在第九秒的时候被执行  
- 第六次tick，它本该在第十秒的时候被执行  

而第七次tick的时间点，将回归正常，即在第十一秒的时候开始被执行。

**2.Delay策略，延迟性的计时策略，当出现延迟后，仍然按部就班地每隔指定的时长计时**。在内部，这种策略是在每次执行tick之后，都修改下一次计时起点为`Instant::now() + Duration`。因此，这种策略下的任何相邻两次的tick，其中间间隔的时长都至少达到Duration。

例如：
```rust
use chrono::Local;
use tokio::{self, runtime::Runtime};
use tokio::time::{self, Duration, Instant, MissedTickBehavior};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        let mut intv = time::interval_at(
            Instant::now() + Duration::from_secs(5),
            Duration::from_secs(2),
        );
        intv.set_missed_tick_behavior(MissedTickBehavior::Delay);

        time::sleep(Duration::from_secs(10)).await;

        println!("start: {}", now());
        intv.tick().await;
        println!("tick 1: {}", now());
        intv.tick().await;
        println!("tick 2: {}", now());
        intv.tick().await;
        println!("tick 3: {}", now());
    });
}
```

输出结果：
```
before: 2021-11-03 19:31:02
start: 2021-11-03 19:31:12
tick 1: 2021-11-03 19:31:12
tick 2: 2021-11-03 19:31:14
tick 3: 2021-11-03 19:31:16
```

**3.Skip策略，忽略型的计时策略，当出现延迟后，仍然所有已经被延迟的计时任务**。这种策略总是以定义计时器时的起点为基准，类似等差数量，每一次执行tick的时间点，一定符合`Start + N * Duration`。

```rust
use chrono::Local;
use tokio::{self, runtime::Runtime};
use tokio::time::{self, Duration, Instant, MissedTickBehavior};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before: {}", now());

        let mut intv = time::interval_at(
            Instant::now() + Duration::from_secs(5),
            Duration::from_secs(2),
        );
        intv.set_missed_tick_behavior(MissedTickBehavior::Skip);

        time::sleep(Duration::from_secs(10)).await;

        println!("start: {}", now());
        intv.tick().await;
        println!("tick 1: {}", now());
        intv.tick().await;
        println!("tick 2: {}", now());
        intv.tick().await;
        println!("tick 3: {}", now());
    });
}
```

输出结果：
```
before: 2021-11-03 19:34:53
start: 2021-11-03 19:35:03
tick 1: 2021-11-03 19:35:03
tick 2: 2021-11-03 19:35:04
tick 3: 2021-11-03 19:35:06
```

注意上面的输出结果中，第一次tick和第二次tick只相差1秒而不是相差2秒。

上面通过`interval_at()`解释清楚了`tokio::time::Interval`的三种计时策略。但在程序中，更大的可能是使用`interval()`来定义间隔计时器，它等价于`interval_at(Instant::now() + Duration)`，表示计时起点从现在开始的计时器。

此外，可以使用`period()`方法获取计时器的间隔时长，使用`missed_tick_behavior()`获取当前的计时策略。
