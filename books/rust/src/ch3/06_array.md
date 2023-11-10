## Array类型

Rust中的数组和其他语言中的数组不太一样，Rust数组长度固定、元素类型相同。

数组的数据类型表示方式为`[Type; N]`，其中：

- Type是该数组要存储什么类型的数据，数组中的所有元素类型都必须是Type  
- N是数组的长度，Rust不会自动伸缩数组的长度  

数组字面量使用中括号`[]`表示，例如`[1,2,3]`。还有一种特殊的表示数组字面量的方式是`[val; N]`，这有点像数组类型的描述方式`[Type; N]`，不过这里表示的是该数组长度为N，并且这N个元素的值都初始化为val。

例如：

```rust
fn main(){
  // 自动推导类型为：[i32; 4]
  let _arr = [11,22,33,44];
  
  let _arr1: [&str; 3] = ["junma", "jinlong", "gaoxiao"];
  
  // 自动推导类型为：[u8; 1024]
  // 该数组初始化为1024个u8类型的0
  // 可将之当作以0填充的1K的buf空间
  let _arr2 = [0_u8; 1024]; 
}
```

注意，`[Type; N]`是用来描述数据类型的，所以其中的N必须在编译期间就能确认，因此N不能是一个变量。

```rust
fn main(){
  let n = 3;
  // 编译错误，提示n不是常量值
  let _arr1: [&str; n] = ["junma", "jinlong", "gaoxiao"];
}
```

可以迭代数组，不过不能直接`for i in arr{}`，而是`for i in &arr{}`或者`for i in arr.iter(){}`。例如：

```rust
fn main(){
  let arr = [11,22,33,44];
  for i in arr.iter() {
    println!("{}", i);
  }
}
```

数组有很多方法可以使用，例如`len()`方法可以获取数组的长度。

```rust
fn main(){
  let arr = [11,22,33,44];
  println!("{}", arr.len());    // 4
}
```

实际上，数组的方法都来自Slice类型。Slice类型后面会详细介绍。
