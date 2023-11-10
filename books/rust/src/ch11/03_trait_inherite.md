## Trait继承

通过让某个类型去实现某个Trait，使得该类型具备该Trait的功能，是组合(composite)的方式。

经常和组合放在一起讨论的是继承(inheritance)。**继承通常用来描述属于同种性质的父子关系(is a)，而组合用来描述具有某功能(has a)**。

例如，支持继承的语言，可以让轿车类型(Car)继承交通工具类型(Vehicle)，表明轿车是一种(is a)交通工具，它们是同一种性质的东西。而如果是支持组合的语言，可以定义可驾驶功能Drivable，然后将Driveable组合到轿车类型、轮船类型、飞机类型、卡车类型、玩具车类型，等等，表明这些类型具有(has a)驾驶功能。

Rust除了支持组合，还支持继承。但**Rust只支持Trait之间的继承**，比如Trait A继承Trait B。实现继承的方式很简单，在定义Trait A时使用冒号加上Trait B即可。

例如：

```rust
trait B{}
trait A: B{}
```

如果Trait A继承Trait B，当类型C想要实现Trait A时，将要求同时也要去实现B。

```rust
trait B{
  fn func_in_b(&self);
}

// Trait A继承Trait B
trait A: B{
  fn func_in_a(&self);
}

struct C{}
// C实现Trait A
impl A for C {
  fn func_in_a(&self){
    println!("impl: func_in_a");
  }
}
// C还要实现Trait B
impl B for C {
  fn func_in_b(&self){
    println!("impl: func_in_b");
  }
}
```

现在，C的实例对象将可以调用`func_in_a()`和`func_in_b()`：

```rust
fn main(){
  let c = C{};
  c.func_in_a();
  c.func_in_b();
}
```

