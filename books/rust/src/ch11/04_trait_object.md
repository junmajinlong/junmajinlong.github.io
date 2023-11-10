## 理解Trait Object和vtable

Trait的另一个作用是Trait Object。

理解Trait Object也简单：当Car、Boat、Bus实现了Trait Drivable后，在需要Drivable类型的地方，都可以使用实现了Drivable的任意类型，如Car、Boat、Bus。从场景需求来说，需要Drivable的地方，其要求的是具有可驾驶功能，而实现了Drivable的Car、Bus等类型都具有可驾驶功能。

所以，只要能保护唐僧去西天取经，是选孙悟空还是选六耳猕猴，这是无关紧要的，重要的是要求具有保护唐僧的能力。

这和鸭子模型(Duck Typing)有点类似，只要叫起来像鸭子，它就可以当成鸭子来使用。也就是说，真正需要的不是鸭子，而是鸭子的叫声。

再看Rust的Trait Object。按照上面的说法，当B、C、D类型实现了Trait A后，就可以将类型B、C、D当作Trait A来使用。这在概念上来说似乎是正确的，但根据Rust语言的特性，Rust没有直接实现这样的用法。原因之一是，**Rust中不能直接将Trait当作数据类型来使用**。

例如，Audio类型实现了Trait Playable，在创建Audio实例对象时不能将数据类型指定为Trait Playable。

```rust
// Trait Playable不能作为数据类型
let x: Playable = Audio{
  name: "telephone.mp3".to_string(),
  duration: 3.42,
};
```

这很容易理解，因为一种类型可能实现了很多种Trait，将其实现的其中一种Trait作为数据类型，显然无法代表该类型。

Rust真正支持的用法是：**虽然Trait自身不能当作数据类型来使用，但Trait Object可以当作数据类型来使用。因此，可以将实现了Trait A的类型B、C、D当作Trait A的Trait Object来使用**。也就是说，Trait Object是Rust支持的一种数据类型，它可以有自己的实例数据，就像Struct类型有自己的实例对象一样。

可以将Trait Object和Slice做对比，它们在不少方面有相似之处。

- 对于类型T，写法`[T]`表示类型T的Slice类型，由于Slice的大小不固定，因此几乎总是使用Slice的引用方式`&[T]`，Slice的引用保存在栈中，包含两份数据：Slice所指向数据的起始指针和Slice的长度。

- 对于Trait A，写法`dyn A`表示Trait A的Trait Object类型，由于Trait Object的大小不固定，因此几乎总是使用Trait Object的引用方式`&dyn A`，Trait Object的引用保存在栈中，包含两份数据：Trait Object所指向数据的指针和指向一个虚表vtable的指针。

上面所描述的Trait Object，还有几点需要解释：  

- Trait Object大小不固定：这是因为，对于Trait A，类型B可以实现Trait A，类型C也可以实现Trait A，因此Trait Object没有固定大小  
- 几乎总是使用Trait Object的引用方式：  
  - 虽然Trait Object没有固定大小，但它的引用类型的大小是固定的，它由两个指针组成，因此占用两个指针大小，即两个机器字长  
  - **一个指针指向实现了Trait A的具体类型的实例**，也就是当作Trait A来用的类型的实例，比如B类型的实例、C类型的实例等  
  - **另一个指针指向一个虚表vtable，vtable中保存了B或C类型的实例对于可以调用的实现于A的方法**。当调用方法时，直接从vtable中找到方法并调用。之所以要使用一个vtable来保存各实例的方法，是因为实现了Trait A的类型有多种，这些类型拥有的方法各不相同，当将这些类型的实例都当作Trait A来使用时(此时，它们全都看作是Trait A类型的实例)，有必要区分这些实例各自有哪些方法可调用   
  - Trait Object的引用方式有多种。例如对于Trait A，其Trait Object类型的引用可以是`&dyn A`、`Box<dyn A>`、`Rc<dyn A>`等   

简而言之，当类型B实现了Trait A时，**类型B的实例对象b可以当作A的Trait Object类型来使用，b中保存了作为Trait Object对象的数据指针(指向B类型的实例数据)和行为指针(指向vtable)**。

一定要注意，此时的**b被当作A的Trait Object的实例数据，而不再是B的实例对象，而且，b的vtable只包含了实现自Trait A的那些方法，因此b只能调用实现于Trait A的方法，而不能调用类型B本身实现的方法和B实现于其他Trait的方法**。也就是说，当作哪个Trait Object来用，它的vtable中就包含哪个Trait的方法。

其实，可以对比着来理解Trait Object，比如v是包含i32类型数据的Vec，v的类型是Vec而不是i32，但v中保存了i32类型的实例数据，另外v也只能调用Vec部分的方法，而不能调用i32相关的方法。

例如：

```rust
trait A{
  fn a(&self){println!("from A");}
}

trait X{
  fn x(&self){println!("from X");}
}

// 类型B同时实现trait A和trait X
// 类型B还定义自己的方法b
struct B{}
impl B {fn b(&self){println!("from B");}}
impl A for B{}
impl X for B{}

fn main(){
  // bb是A的Trait Object实例，
  // bb保存了指向类型B实例数据的指针和指向vtable的指针
  let bb: &dyn A = &B{};
  bb.a();  // 正确，bb可调用实现自Trait A的方法a()
  bb.x();  // 错误，bb不可调用实现自Trait X的方法x()
  bb.b();  // 错误，bb不可调用自身实现的方法b()
}
```

### 使用Trait Object类型

了解Trait Object之后，使用它就不再难了，它也只是一种数据类型罢了。

例如，前文的Audio类型和Video类型都实现Trait Playable：

```rust
// 为了排版，调整了代码格式
trait Playable {
  fn play(&self);
  fn pause(&self) {println!("pause");}
  fn get_duration(&self) -> f32;
}

// Audio类型，实现Trait Playable
struct Audio {name: String, duration: f32}
impl Playable for Audio {
  fn play(&self) {println!("listening audio: {}", self.name);}
  fn get_duration(&self) -> f32 {self.duration}
}

// Video类型，实现Trait Playable
struct Video {name: String, duration: f32}
impl Playable for Video {
  fn play(&self) {println!("watching video: {}", self.name);}
  fn pause(&self) {println!("video paused");}
  fn get_duration(&self) -> f32 {self.duration}
}
```

现在，将Audio的实例或Video的实例当作Playable的Trait Object来使用：

```rust
fn main() {
  let x: &dyn Playable = &Audio{
    name: "telephone.mp3".to_string(),
    duration: 3.42,
  };
  x.play();
  
  let y: &dyn Playable = &Video{
    name: "Yui Hatano.mp4".to_string(),
    duration: 59.59,
  };
  y.play();
}
```

此时，x的数据类型是Playable的Trait Object类型的引用，它在栈中保存了一个指向Audio实例数据的指针，还保存了一个指向包含了它可调用方法的vtable的指针。同理，y也一样。

再比如，有一个Playable的Trait Object类型的数组，在这个数组中可以存放所有实现了Playable的实例对象数据：

```rust
use std::fmt::Debug;

fn main() {
  let a:&dyn Playable = &Audio{
    name: "telephone.mp3".to_string(),
    duration: 3.42,
  };
  
  let b: &dyn Playable = &Video {
    name: "Yui Hatano.mp4".to_string(),
    duration: 59.59,
  };
  
  let arr: [&dyn Playable;2] = [a, b];
  println!("{:#?}", arr);
}

trait Playable: Debug {}

#[derive(Debug)]
struct Audio {}
impl Playable for Audio {}

#[derive(Debug)]
struct Video {...}
impl Playable for Video {...}
```

注意，上面为了使用`println!`的调试输出格式`{:#?}`，要让Playable实现名为`std::fmt::Debug`的Trait，因为Playable自身也是一个Trait，所以使用Trait继承的方式来继承Debug。继承Debug后，要求实现Playable Trait的类型也都要实现Debug Trait，因此在Audio和Video之前使用`#[derive(Debug)]`来实现Debug Trait。

上面实例的输出结果：

```rust
[
    Audio {
        name: "telephone.mp3",
        duration: 3.42,
    },
    Video {
        name: "Yui Hatano.mp4",
        duration: 59.59,
    },
]
```

