## 定义Enum的完整语法

enum创建枚举类型有多种方式，其每个成员的定义都类似于创建Struct结构的语法。

例如：
```rust
enum E {
  F1,             // 该成员类似于unit-like struct
  F2(i32, u64),   // 该成员类似于tuple struct
  F3{x: i32, y: u64}, // 该成员类似于named struct
}
```

F1成员这种定义方式自无需再多做介绍，前文定义的枚举类型都是这种类型的成员。

F2成员的定义类似于tuple struct，F2成员包含两个字段，这两个字段类型分别是i32和u64。也就是说，枚举类型E的F2成员，是一个包含了具体数据的成员。

F3成员的定义类似于named struct，F3成员包含x和y两个字段，字段类型分别是i32和u64。也就是说，枚举类型E的F3成员，也是一个包含了具体数据的成员。

正是因为枚举类型允许定义F2和F3这种包含数据的成员，使得枚举类型在Rust中扮演的角色变得更为重要。

例如，Rust要实现一个Json解析工具，只需定义一个枚举类型去枚举Json允许的数据类型，参考如下代码。

```rust
enum Json {
  Null,
  Boolean(bool),
  Number(f64),
  String(String),
  Array(Vec<Json>),
  Object(Box<HashMap<String, Json>>),
}
```

不可否认，Rust语言的表达能力很强。例如这里的枚举类型，仅仅这样一个简单的数据结构就表达出很多内容，而在其它语言中，这可能需要定义很多方法来表达出这些内容。

