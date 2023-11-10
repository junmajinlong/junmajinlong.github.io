## 理解Rust中的变量赋值

**Rust中使用`let`声明变量**：

```rust
fn main(){
  // 声明变量name并初始化赋值
  let name = "junmajinlong.com";
  println!("{}", name);  // println!()格式化输出数据
}
```

**Rust会对未使用的变量发出警告信息**。如果确实想保留从未被使用过的变量，可在变量名前加上`_`前缀。

```rust
fn main(){
  let name = "junmajinlong.com";
  println!("{}", name);

  let gender = "male";   // 警告，gender未使用
  let _age = 18;     // 加_前缀的变量不被警告
}
```

**Rust允许声明未被初始化(即未被赋值)的变量，但不允许使用未被赋值的变量**。多数情况下，都是声明的时候直接初始化的。

```rust
fn main() {
  let name;  // 只声明，未初始化
  // println!("{}", name);  // 取消该行注释，将编译错误
  
  name = "junmajinlong.com";
  println!("{}", name);
}
```

Rust允许**重复声明**同名变量，后声明的变量将**遮盖**(shadow)前面已声明的变量。需注意的是，遮盖不是覆盖，被遮盖的变量仍然存在，而如果是被覆盖则不再存在(也即，覆盖时，原数据会被销毁)。

```rust
fn main() {
  let name = "junmajinlong.com";
  // 注释下行，将警告：name变量未被使用
  // 因为name仍然存在，只是被遮盖了
  println!("{}", name);  
  
  let name = "gaoxiaofang.com";  // 遮盖已声明的name变量
  println!("{}", name);
}
```

变量遮盖示意图：

```
注：下图内存布局并不完全正确，此图仅为说明变量遮盖
         +---------+       +--------------------+
         |  Stack  |       |        Heap        |
         +---------+       +--------------------+
name --> | 0x56789 |  ---> | "gaoxiaofang.com"  |
         |         |       +--------------------+
name --> | 0x01234 |  ---> | "junmajinlong.com" |
         +---------+       +--------------------+
```

**变量初始化后，默认不允许再修改该变量**。注意，修改变量是直接给变量赋值，而不是再次let声明该变量，再次声明变量是允许的，它会遮盖原变量。

```rust
fn main() {
  let name = "junmajinlong.com";
  // 取消下行注释将编译错误，默认不允许修改变量
  // name = "gaoxiaofang.com";
  
  let name = "gaoxiaofang.com";  // 再次声明变量，遮盖变量
  println!("{}", name);
}
```

**如果想要修改变量的值，需要在声明变量时加上`mut`标记**(mutable)表示该变量是可修改的。

```rust
fn main() {
  let mut name = "junmajinlong.com";
  println!("{}", name);
  
  name = "gaoxiaofang.com";   // 修改变量
  println!("{}", name);
}
```

Rust不仅对未被使用过的变量发出警告，**还对赋值过但未被使用过的值发出警告**。比如变量赋值后，尚未读取该变量，就重新赋值了。

```rust
fn main() {
  let mut name = "junmajinlong.com"; // 警告值未被使用过
  name = "gaoxiaofang.com"; 
  println!("{}", name);
}
```

Rust是静态语言，**声明变量时需指定该变量将要保存的值的数据类型，这样编译器编译时才知道为该变量将要保存的数据分配多少内存、允许存放什么类型的数据以及如何存放数据**。但Rust编译器会根据所保存的值来推导变量的数据类型，推导得到确定的数据类型之后(比如第一次为该变量赋值之后)，就不再允许存放其他类型的数据。

```rust
fn main() {
  // 根据保存的值推导数据类型
  // 推导结果：变量name为 &str 数据类型
  let mut name = "junmajinlong.com"; 
  //name = 32;  // 再让name保存i32类型的数据，报错
}
```

**当Rust无法推导类型时，或者声明变量时就明确知道该变量要保存声明类型的数据时，可明确指定该变量的数据类型**。

```rust
fn main() {
  // 指定变量数据类型的语法：在变量名后加": TYPE"
  let age: i32 = 32;  // 明确指定age为i32类型
  println!("{}", name);
  
  // i32类型的变量想存储u8类型数据，不允许
  // age = 23_u8;
}
```

虽然Rust是基于表达式的语言，但变量声明的let代码是语句而非表达式。这意味着**let操作没有返回值，因此无法使用let来连续赋值**。

```rust
fn main(){
  let a = (let b = 1);  // 错误
}
```

**可以使用tuple的方式同时为多个变量赋值，并且可以使用下划线`_`占位表示忽略某个变量的赋值过程**。

```rust
// x = 11, y = 22, 忽略33
let (x, y, _) = (11, 22, 33);
```

事实上，`_`占位符比想象中还更会【偷懒】，其他语言中`_`表达的含义可能是丢弃其赋值结果(甚至不丢弃)，**但Rust中的`_`会直接忽略变量赋值的过程。这导致了这样一种看似奇怪的现象：使用普通变量名会导致报错的变量赋值行为，使用`_`却不会报错**。

例如，下面(1)不会报错，而(2)会报错。这里涉及到了后面所有权转移的内容，如果看不懂请先跳过，只需记住结论：`_`会直接忽略赋值的过程。

```rust
// (1)
let s1 = "junmajinlong.com".to_string();
let _ = s1;
println!("{}", s1); // 不会报错

// (2)
let s2 = "junmajinlong.com".to_string();
let ss = s2;
println!("{}", s2); // 报错
```

最后要说明的是，Rust中变量赋值操作实际上是Rust中的一种模式匹配，在后面的章节中将更系统、更详细地介绍Rust模式匹配功能。
