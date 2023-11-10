## Option类型

在定义函数时，函数可能会返回具体的值，也有可能会返回表示空的值(None、Null、Nil等)。

在其他语言中，这两种可能的返回值是独立的，需要用户在调用函数时自己去判断其所返回的是具体值还是空值(表示空值的通常是Null、Nil、None等)。例如，下面是一段Ruby代码：

```ruby
# 如果x和y都是整数，则返回x和y之和，否则返回空值nil
def ff(x,y)
  if x.is_a?(Integer) && y.is_a?(Integer)
    return x + y
  end
  
  return nil
end

# 调用ff()，判断返回结果是否是nil
# 如果返回nil，则报错退出，否则输出a、b之和
if ff(a, b) 
  puts "a+b: #{a+b}"
else
  raise "error"
end
```

这样直接使用表示空值的Nil(或None或Null)时，会让程序中充斥着这类是否为空的判断代码。

Rust没有其他语言中用来表示空的类型，但这种问题也需要解决。解决起来也简单：**如果一个函数可能返回具体值，也有可能返回空值，那么就让它返回`Option<T>`**。`Option<T>`是一种归纳了这两种可能的Enum类型：

```rust
enum Option<T> { 
  /// No value 
  None, 
  /// Some value `T` 
  Some(T), 
}
```

其中`None`用来代表空值，`Some(T)`用来代表具体的值。这个具体的值可以是任意类型的，因此用泛型T来表示。如果你愿意，也可以将`Option<T>`改写成稍微繁琐一点的定义：

```rust
enum Option<T>{
  None,
  Some{
    value: T,
  }
}
```

可能有的人会纠结这里的Some，想要搞清楚它的作用。实际上，Option中的Some没有额外意义，仅仅只是因为Enum类型必须得由枚举成员来携带数据，因此，Some仅仅只是用来在Enum类型中携带任意类型的具体数据而提供的枚举成员。也许，通过下面这种错误的写法可能更容易理解为什么要加一个Some：

```rust
enum Option<T>{
  None,
  T,    // 泛型T作为成员？
}
```

有了`Option<T>`后，当一个函数可能返回具体数据，可能返回空值时，那就让它返回`Option<T>`。

例如，下面定义一个返回`Option<i32>`的Rust函数，如果x小于0或y小于0，则返回空，否则返回两者之和。

```rust
fn ff(x: i32, y: i32) -> Option<i32> {
  if x < 0 || y < 0 {
    return None;
  }
  Some(x + y)
}
```

有时候也会直接使用Option类型的成员值。例如：

```rust
// a的类型：Option<i32>
let a = Some(3);

// 通过None无法推断b的类型，因此要指定类型
let b:Option<i32> = None;
```

另外，Option类型实际上是由标准库std中提供的，而不是Rust核心自带的，它的完整路径是`std::option::Option`。但Option太常用了，因此它被加入了std的prelude中，使得编写代码时可以直接使用Option类型，而不需要写全路径或事先导入。

### 处理Option类型的返回值

虽然Rust将空值和任意一种值绑定在一起提升成为一种数据类型`Option<T>`，但并不代表Rust对待空值的行为比其他语言更好。因为无论是Rust还是其他语言，都需要在定义函数时考虑是返回具体值还是空值，也需要在调用函数时处理可能是空值的情况。这是逻辑上的事实，不是语言层面上的问题，不同的语言只是提供不同的方案。

但Rust可以通过所返回的`Option<T>`类型来提醒程序员：该函数需要处理可能返回的空值。如果不做处理，Rust会编译报错。

处理`Option<T>`类型的返回值有多种方式，最原始的处理方式是使用Rust的模式匹配功能，匹配该返回值中包含的是`None`还是`Some(T)`。

例如，对于上面定义的函数`ff()`，使用模式匹配来处理其`Option<T>`类型的返回值。

```rust
fn main(){
  let res = match ff(33, 44) {
    // 分支一：
    // 如果ff()的返回值是Some()，则将Some
    // 中封装的值赋值给局部变量x，x返回给变量res
    // 于是res变量就获取到了ff()所返回的具体值
    Some(x) => x,
    
    // 分支二：
    // 如果ff()的返回值是None，则直接报错退出
    None => panic!("error"),
  };

  println!("{}", res);
}
```

除了使用模式匹配的方式来穷举Option中的两个成员，Rust也为此提供了一些方便的方法。

### 处理Option的常用方法













