# 布尔类型

Rust中的Boolean类型有两个值：true和false。

类似于if、while等的控制语句以及逻辑运算符`|| && !`都需要进行条件判断，Rust只允许在条件判断处使用布尔类型。

例如，要判断x是否等于0，在其他语言中可能允许如下写法：

```rust
if x {
  ...
}
```

但在Rust中，不允许上面的写法(除非x的值自身就是true或false)。

**Rust中必须得在条件判断处写返回值为true/false的表达式**。例如写成如下形式：

```rust
if x == 0 {
  ...
}
```

**Rust的布尔值可以使用as操作符转换为各种数值类型，false对应0，true对应1**。但数值类型不允许转换为bool值。再次提醒，as操作符常用于原始数据类型之间的类型转换。

```rust
fn main() {
  println!("{}", true as u32);
  println!("{}", false as u8);
  // println!("{}", 1_u8 as bool);  // 编译错误
}
```



