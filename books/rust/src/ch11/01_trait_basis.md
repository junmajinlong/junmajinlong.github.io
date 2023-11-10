## Trait的基本用法

Trait最基本的作用是从多种类型中抽取出共性的属性或方法，并定义这些方法的规范(即方法签名)。

例如，对于Audio类型和Video类型，它们有几个具有共性的方法：  

- play方法用于播放  
- pause方法用于暂停  
- get_duration方法用于显示媒体的总时长  

为了抽取这些共性方法，可定义一个名为Playable的Trait，并在其中规范好这些方法的签名。

自定义Trait类型时，使用`trait`关键字。如：

```rust
trait Playable {
  fn play(&self);
  fn pause(&self) {
    println!("pause");
  }
  fn get_duration(&self) -> f32;
}
```

注意上面的play方法和get_duration方法都仅仅只规范了它们的方法签名，并没有为它们定义方法体，而pause方法则指定了函数签名且定义了方法体，这个方法体是pause方法的默认方法体。

定义好Playable Trait后，先让Audio类型去实现Playable：

```rust
struct Audio {
  name: String,
  duration: f32,
}

impl Playable for Audio {
  fn play(&self) {
    println!("listening audio: {}", self.name);
  }
  fn get_duration(&self) -> f32 {
    self.duration
  }
}
```

注意，上面`impl Playable for Audio`表示为Audio类型实现Playable Trait。Audio实现Playable Trait时，Trait中的所有没有提供默认方法体的方法(即play方法和get_duration方法)都需要实现。对于提供了默认方法体的方法，可实现可不实现，如果实现了则覆盖默认方法体，如果没有实现，则使用默认方法体。

下面再为Video类型实现Playable Trait，这里也实现了有默认方法体的pause方法：

```rust
struct Video {
  name: String,
  duration: f32,
}

impl Playable for Video {
  fn play(&self) {
    println!("watching video: {}", self.name);
  }
  fn pause(&self) {
    println!("video paused");
  }
  fn get_duration(&self) -> f32 {
    self.duration
  }
}
```

当Audio类型和Video类型实现了Playable Trait后，这两个类型的实例对象自然可以去调用它们各自定义的方法。而对于Audio没有定义的pause方法，则会从其所实现的Trait中寻找。

```rust
fn main() {
  let audio = Audio{
    name: "telephone.mp3".to_string(),
    duration: 4.32,
  };
  audio.play();
  audio.pause();
  println!("{}", audio.get_duration());

  let video = Video {
    name: "Yui Hatano.mp4".to_string(),
    duration: 59.59,
  };
  video.play();
  video.pause();
  println!("{}", video.get_duration());
}
```

