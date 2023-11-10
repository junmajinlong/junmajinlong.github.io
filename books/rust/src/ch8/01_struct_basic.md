## Struct的基本使用

使用`struct`关键字定义Struct类型。

### 具名Struct

具名Struct(named Struct)表示有字段名称的Struct。Struct的字段(Field)也可以称为Struct的属性(Attribute)。

例如，定义一个名为Person的Struct结构体，Person包含三个属性，分别是name、age和email，每个属性都需要指定数据类型，这样可以限制各属性允许存放什么类型的数据。
```rust
struct Person{
  name: String,
  age: u32,
  email: String, // 最后一个字段的逗号可省略，但建议保留
}
```

定义Struct后，可创建Struct的实例对象，为其各个属性指定对应的值即可。

例如，构造Person结构体的实例对象user1，
```rust
let user1 = Person {
  name: String::from("junmajinlong"),
  email: String::from("junmajinlong@xx.com"),
  age: 23,
};
```

创建user1实例对象后，可以通过`user1.name`访问它的name字段的值，`user1.age`访问它的age字段的值。

以下是一段完整的代码：
```rust
struct Person{
  name: String,
  age: u32,
  email: String,
}

fn main(){
  let user1 = Person{
    name: String::from("junmajinlong"),
    email: String::from("junmajinlong@xx.com"),
    age: 23,
  };
  // 访问user1实例name字段、age字段和email字段的值
  println!(
    "name: {}, age: {}, email: {}",
    user1.name, user1.age, user1.email
  );
}
```

### 构造struct的简写方式

当要构造的Struct实例的字段值来自于变量，且这个变量名和字段名相同，则可以简写该字段。

```rust
struct Person{
  name: String,
  age: u32,
  email: String,
}

fn main(){
  let name = String::from("junmajinlong");
  let email = String::from("junmajinlong@xx.com");

  let user1 = Person{
    name,      // 简写，等价于name: name
    email,     // 简写，等价于email: email
    age: 23,
  };
}
```

有时候会基于一个Struct实例构造另一个Struct实例，Rust允许通过`..xx`的方式来简化构造struct实例的写法：
```rust
let name = String::from("junmajinlong");
let email = String::from("junmajinlong@xx.com");
let user1 = Person{
  name,
  email,
  age: 23,
};

let mut user2 = Person{
  name: String::from("gaoxiaofang"),
  email: String::from("gaoxiaofang@yy.com"),
  ..user1
};
```

上面的`..user1`表示让user2借用或拷贝user1的某些字段值，由于user2中已经手动定义了name和email字段，因此`..user1`只借用了user1的age字段，即`user2.age`也是23。

注意，如果`..base`借用于base的字段是可Copy的，那么在借用时会自动Copy，这样在借用字段之后，base中的字段仍然有效。但如果借用的字段不是Copy的，那么在借用时会将base中字段的所有权转移走，使得base中的该字段无效。

例如，同时借用user1中的age字段和email字段，由于age是i32类型，是Copy的，所以`user1.age`仍然可用，但由于String类型不是Copy的，所以`user1.email`不可用。
```rust
let name = String::from("junmajinlong");
let email = String::from("junmajinlong@xx.com");
let user1 = Person{
  name,
  email,
  age: 23,
};

let mut user2 = Person{
  name: String::from("gaoxiaofang"),
  ..user1
};

// 报错，user1.email字段值的所有权已借给user2
// println!("{}", user1.email);
// println!("{}", user1);       // 报错
println!("{}", user1.name);     // 正确
println!("{}", user1.age);      // 正确
```

如果确实要借用user1的email属性，可以使用`..user1.clone()`先拷贝堆内存中的user1，这样就不会借用原始的user1中的email所有权。

```rust
let user2 = Person{
  name: String::from("ggg"),
  ..user1.clone()
}
```

### tuple struct

除了named struct外，Rust还支持没有字段名的struct结构体，称为元组结构体(tuple struct)。

例如：
```rust
struct Color(i32, i32, i32); 
struct Point(i32, i32, i32); 

let black = Color(0, 0, 0); 
let origin = Point(0, 0, 0);
```

black和origin值的类型不同，因为它们是不同的结构体的实例。在其他方面，元组结构体实例类似于元组：可以将其解构，也可以使用`.`后跟索引来访问单独的值，等等。

### unit-like struct

类单元结构体(unit-like struct)是没有任何字段的空struct。
```rust
struct St;
```


