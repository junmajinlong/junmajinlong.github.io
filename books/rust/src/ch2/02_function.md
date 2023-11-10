## Rust中定义函数

Rust中使用`fn`关键字定义函数，定义函数时需指定参数的数据类型，如果有返回值，则需要指明返回值的数据类型。

fn关键字、函数名、函数参数及其类型、返回值类型组成**函数签名**。例如`fn fname(a: i32, b: i32)->i32`是一个函数签名。

定义函数参见如下几个简单的示例：

```rust
// 没有参数、没有返回值
fn f0(){
  println!("first function_0");
  println!("first function_1");
}

// 有参数，没有返回值
fn f1(a: i32, b: i32) {
  println!("a: {}, b: {}", a, b);
}

// 有参数，有返回值
fn f2(a: i32, b: i32) -> i32 {
  return a + b;
}

// 调用函数
fn main(){
  f0();
  f1(1,2);
  f2(3,4);
}
```

函数也可以直接定义在函数内部。例如在函数a中定义函数b，这样函数b就只能在函数a中访问或调用：

```rust
fn f0(){
  println!("first function_0");
  println!("first function_1");
  
  fn f1(a: i32, b: i32) {
    println!("a: {}, b: {}", a, b);
  }
  
  f1(2,3);
}

fn main(){
  f0();
}
```

Rust有两种方式指定函数返回值：  
- 使用return来指定返回值，此时return后要加上分号结尾，使得return成为一个语句  
  - return关键字不指定返回值时，默认返回`()`  
- 不使用return，将返回最后一条执行的表达式计算结果，该表达式尾部不能带分号  
  - 不使用return，但如果最后一条执行的是一个分号结尾的语句，则返回`()`  

参考如下函数定义：

```rust
fn f0(a: i32) -> i32{
  if a > 0 {
    // 使用return来返回，结尾处必须不能缺少分号
    return a * 2;
  }
  
  // 最后执行的一条代码，使用表达式的结果作为函数返回值
  // 结尾必须不能带分号
  a * 2
}
```
