## 泛型的基本使用

通过泛型系统，可以减少很多冗余代码。

例如，不使用泛型时，定义一个参数允许为u8、i8、u16、i16、u32、i32......等类型的double函数时：

```rust
fn double_u8(i: u8) -> u8 { i + i }
fn double_i8(i: i8) -> i8 { i + i }
fn double_u16(i: u16) -> u16 { i + i }
fn double_i16(i: i16) -> i16 { i + i }
fn double_u32(i: u32) -> u32 { i + i }
fn double_i32(i: i32) -> i32 { i + i }
fn double_u64(i: u64) -> u64 { i + i }
fn double_i64(i: i64) -> i64 { i + i }

fn main(){
  println!("{}",double_u8(3_u8));
  println!("{}",double_i16(3_i16));
}
```

上面定义了一堆double函数，函数的逻辑部分是完全一致的，仅在于类型的不同。

泛型可以用于解决这样因类型而代码冗余的问题。使用泛型时：

```rust
use std::ops::Add;
fn double<T>(i: T) -> T
  where T: Add<Output=T> + Clone + Copy {
  i + i
}

fn main(){
  println!("{}",double(3_i16));
  println!("{}",double(3_i32));
}
```

上面的字母T就是泛型(和变量x的含义是相同的)，它用来代表各种可能的数据类型。多数时候泛型使用单个大写字母来表示，但也可以使用多个字母来表示。

对于double函数签名的前一部分：

```rust
fn double<T>(i: T) -> T 
```

函数名称后面的`<T>`表示在函数作用域内定义一个泛型T，这个泛型只能在函数签名和函数体内使用，就跟在一个作用域内定义一个变量，这个变量只能在该作用域内使用是一样的。而且，泛型本就是代表各种数据类型的变量。

参数部分`i: T`表示参数i的类型是泛型T。

返回值部分`-> T`表示该函数的返回值类型是泛型T。

因此，上面这部分函数签名表达的含义是：传入某种数据类型的参数，也返回这种数据类型的返回值，且这种数据类型可以是任意的类型。

对于第一次接触泛型的人来说，这可能很难理解。但是，换成类似的使用普通变量的代码，可能就容易理解了：

```
# 伪代码：传入一个数据，返回这个数据
function f(x) {return x}
```

### 对泛型进行限制

但注意，double函数期待的是对数值进行加法操作，但泛型却可以代表各种类型，因此，还需要对泛型T进行限制，否则在调用double函数时就允许传递字符串类型、Vec类型、Person类型等值作为函数参数，这偏离了期待。

例如，在double的函数体内需要对泛型T的值i进行加法操作，但只有实现了`std::ops::Add` Trait的类型才能使用`+`进行加法操作。因此要限制泛型T是那些实现了`std::ops::Add`的数据类型。

限制泛型也叫做Trait绑定(Trait Bound)，其语法有两种：  

- 在定义泛型类型T时，使用类似于`T: Trait_Name`这种语法进行限制  
- 在返回值后面、大括号前面使用where关键字，如`where T: Trait_Name`  

因此，下面两种写法是等价的：

```rust
fn f<T: Clone + Copy>(i: T) -> T{}

fn f<T>(i: T) -> T
  where T: Clone + Copy {}

// 更复杂的示例：
fn query<M: Mapper + Serialize, R: Reducer + Serialize>(
    data: &DataSet, map: M, reduce: R) -> Results 
{
    ...
}

// 此时，下面写法更友好、可读性更高
fn query<M, R>(data: &DataSet, map: M, reduce: R) -> Results 
    where M: Mapper + Serialize,
          R: Reducer + Serialize
{
    ...
}
```

其中，`T: Trait_Name`表示将泛型T限制为那些实现了Trait_Name Trait的数据类型。因此`T: std::ops::Add`表示泛型T只能代表那些实现了`std::ops::Add` Trait的数据类型，比如各种数值类型都实现了Add Trait，因此T可以代表数值类型，而Vec类型没有实现Add Trait，因此T不能代表Vec类型。

观察指定变量数据类型的写法`i: i32`和限制泛型的写法`T: Trait_Name`，由此可知，Trait其实是泛型的数据类型，Trait限制了泛型所能代表的类型，正如数据类型限制了变量所能存放的数据。

有时候需要对泛型做多重限制，这时使用`+`即可。例如`T: Add<Output=T>+Copy+Clone`，表示限制泛型T只能代表那些同时实现了Add、Copy、Clone这三种Trait的数据类型。

之所以要做多重限制，是因为有时候限制少了，泛型所能代表的类型不够精确或者缺失某种功能。比如，只限制泛型T是实现了`std::ops::Add` Trait的类型还不够，还要限制它实现了Copy Trait以便函数体内的参数i被转移所有权时会自动进行Copy，但Copy Trait是Clone Trait的子Trait，即Copy依赖于Clone，因此限制泛型T实现Copy的同时，还要限制泛型T同时实现Clone Trait。

简而言之，要对泛型做限制，一方面的原因是函数体内需要某种Trait提供的功能(比如函数体内要对i执行加法操作，需要的是`std::ops::Add`的功能)，另一方面的原因是要让泛型T所能代表的数据类型足够精确化(如果不做任何限制，泛型将能代表任意数据类型)。

### 泛型的引用类型

如果参数是一个引用，且又使用泛型，则需要使用泛型的引用`&T`或`&mut T`。

例如：

```rust
use std::fmt::Display;

fn f<T: Display>(i: &T) {
  println!("{}", *i);
}
```

### 零运行开销的泛型：泛型代码膨胀

rustc在编译代码时，会将所有的泛型替换成它所代表的具体数据类型，就像编译期间会将变量名替换成它所代表数据的内存地址一样。

例如，对于下面这个泛型函数：

```rust
use std::ops::Add;
fn double_me<T>(i: T) -> T
  where T: Add<Output=T> + Clone + Copy {
  i + i
}

fn main() {
  println!("{}", double_me(3u32));
  println!("{}", double_me(3u8));
  println!("{}", double_me(3i8));
}
```

在编译期间，rustc会根据调用double_me()时传递的具体数据类型进行替换。上面示例使用了u32、u8和i8三种类型的值传递给泛型参数，那么编译期间，编译器会对应生成三个double_me()函数，它们的参数类型分别是u32、u8和i8。

```bash
$ rustc src/main.rs
$ strings main | grep "double_me"
_ZN4main9double_me17h6d861a9e8ab36c42E
_ZN4main9double_me17ha214a9977249a1bfE
_ZN4main9double_me17hbc458c5fab68c203E
```

由于编译期间，编译器会对泛型类型进行替换，这会导致泛型代码膨胀(code bloat)，从一个函数膨胀为零个、一个或多个具体数据类型的函数。有时候这种膨胀会导致编译后的程序文件变大很多。不过，多数情况下，代码膨胀的问题都不是大问题。

另一方面，由于编译期间已经将泛型替换成了具体的数据类型，因此，在程序运行期间，直接调用对应类型的函数即可，不需要再消耗任何额外的资源去计算泛型所代表的具体类型。因此，Rust的泛型是零运行时开销的。

