## tuple类型

Rust的tuple类型可以存放0个、1个或多个任意数据类型的数据。使用`tup.N`的方式可以访问索引为N的元素。

```rust
let n = (11, 22, 33);
println!("{}", n.0);  // 11
println!("{}", n.1);  // 22
println!("{}", n.2);  // 33
```

注意，**访问tuple元素的索引必须是编译期间就能确定的数值**，而不能是变量。

```rust
let n = (11, 22, 33);
let a: usize = 2;
println!("{}", n.a);  // 错误
```

实际上，`n.a`会被Rust解析为对Struct类型的变量n的a字段的访问。

tuple通常用来作为简单的数据组合体。

例如：

```rust
fn main(){
  // 表示一个人的name和age
  let p_name = "junmajinlong";
  let p_age = 23;
  println!("{}, {}", p_name, p_age);
  
  // 与其将有关联的数据分开保存到多个变量中，
  // 不如保存在一个结构中
  let p = ("junmajinlong", 23); // 同时存放&str和i32类型的数据
  println!("{}, {}", p.0, p.1);
}
```

Rust中经常会将tuple类型的各元素赋值给各变量，方式如下：

```rust
fn main(){
  let p = ("junmajinlong", 23);
  
  // 也可以类型推导：let (name,age) = p;
  let (name, age): (&str, i32) = p;
  // 比 let name = p.0; let age = p.1; 更简洁
  println!("{}, {}", name, age);
}
```

有时候tuple里只会保存一个元素，此时必须不能省略最后的逗号：

```rust
let p = ("junmajinlong",);
```

## unit类型

不保存任何数据的tuple表示为`()`。在Rust中，它是特殊的，它有自己的类型：unit。

unit类型的写法为`()`，该类型也只有一个值，写法仍然是`()`。参考下面的写法应该能搞清楚。

```rust
// 将值()保存到类型为()的变量x中
//    类型    值
let x: ()  =  ();
```

unit类型通常用在那些不关心返回值的函数中。在其他语言中，那些不写return语句或return不指定返回内容的的函数，一般表示不关心返回值。在Rust中可将这种需求写为`return ()`。