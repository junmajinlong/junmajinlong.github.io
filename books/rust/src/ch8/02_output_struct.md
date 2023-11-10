## 调试输出Struct

在开发过程中，很多时候会想要查看某个struct实例中的数据，但直接输出是不行的：

```rust
struct Person{
  name: String,
  age: i32,
}

fn main(){
  let p = Person{
    name: String::from("junmajinlong"),
    age: 23,
  };

  // 直接输出p会报错
  println!("{}", p);
}
```

这时需要在`struct Person`前加上`#[derive(Debug)]`，然后使用`{:?}`或`{:#?}`进行调试输出。
```rust
#[derive(Debug)]
struct Person{
  name: String,
  age: i32,
}

fn main(){
  let p = Person{
    name: String::from("junmajinlong"),
    age: 23,
  };

  println!("{:?}", p);
  println!("{:#?}", p);
}
```

输出结果：
```
Person { name: "junmajinlong", age: 23 }
Person {
    name: "junmajinlong",
    age: 23,
}
```
