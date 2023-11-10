 ## 使用泛型的位置

不仅仅是函数的参数可以指定泛型，任何需要指定数据类型的地方，都可以使用泛型来替代具体的数据类型，以此来表示此处可以使用比某种具体类型更为通用的数据类型。

而且，可以同时使用多个泛型，只要将不同的泛型定义为不同的名称即可。例如，HashMap类型是保存键值对的类型，它的key是一种泛型类型，它的值也是一种泛型类型。它的定义如下：

```rust
// 使用了三个泛型，分别是K、V、S，并且泛型S的默认类型是RandomState
struct HashMap<K, V, S = RandomState> {
  base: base::HashMap<K, V, S>,
}
```

实际上，Struct、Enum、impl、Trait等地方都可以使用泛型，仍然要注意的是，需要在类型的名称后或者impl后先声明泛型，才能使用已声明的泛型。

下面是一些简单的示例。

Struct使用泛型：

```rust
struct Container_tuple<T> (T)
struct Container_named<T: std::fmt::Display> {
  field: T,
}
```

例如，Vec类型就是泛型的Struct类型，其官方定义如下：

```rust
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}
```

Enum使用泛型：

```rust
enum Option<T> {
  Some(T),
  None,
}
```

impl实现类型的方法时使用泛型：

```rust
struct Container<T>{
  item: T,
}

// impl后的T是声明泛型T
// Container<T>的T对应Struct类型Container<T>
impl<T> Container<T> {
  fn new(item: T) -> Self {
    Container {item}
  }
}
```

Trait使用泛型：

```rust
// 表示将某种类型T转换为当前类型
trait From<T> { 
  fn from(T) -> Self; 
}
```

某数据类型impl实现Trait时使用泛型：

```rust
use std::fmt::Debug;

trait Eatable {
  fn eat_me(&self);
}

#[derive(Debug)]
struct Food<T>(T);

impl<T: Debug> Eatable for Food<T> {
  fn eat_me(&self) {
    println!("Eating: {:?}", self);
  }
}
```

注意，上面impl时指定了`T: Debug`，它表示了`Food<T>`类型的T必须实现了Debug。为什么不直接在定义Struct时，将Food定义为`struct Food<T: Debug>`而是在impl Food时才限制泛型T呢？

通常，**应当尽量不在定义类型时限制泛型的范围**，除非确实有必要去限制。这是因为，泛型本就是为了描述更为抽象、更为通用的类型而存在的，限制泛型将使得类型变得更具体化，适用场景也更窄。但是在impl类型时，应当去限制泛型，并且遵守缺失什么功能就添加什么限制的规范，这样可以使得所定义的方法不会过度泛化，也不会过度具体化。

简单来说，尽量不去限制类型是什么，而是限制类型能做什么。

另一方面，即使定义`struct Food<T: Debug>`，在`impl Food<T>`时，也**仍然要在impl时指定泛型的限制，否则将编译错误**。

```rust
#[derive(Debug)]
struct Food<T: Debug>(T);
impl<T: Debug> Eatable for Food<T> {}
```

也就是说，**如果某个泛型类型有对应的impl，那么在定义类型时指定的泛型限制很可能是多余的。但如果没有对应的impl，那么可能有必要在定义泛型类型时加上泛型限制**。

