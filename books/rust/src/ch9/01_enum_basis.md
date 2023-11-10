## Enum的基本使用

Rust使用`enum`关键字定义枚举类型(Enum)。

例如，定义一个用来描述性别的枚举类型，名为Gender，它只枚举两种值：Male(表示男)，Female(表示女)。
```rust
enum Gender {
  Male,   // 男
  Female, // 女
}
```

Enum作为一种数据类型，可以用来限制允许存放的数据。比如某变量的数据类型是Gender类型，那么该变量只允许存放指定的两种值：Male或Female，不允许存放其他任何值。也就是说，枚举类型的每个实例都是从枚举类型中进行多选一的值。

```rust
let g1: Gender = Gender::Male;
let g2: Gender = Gender::Female;
// let g3: Gender = "male";  // 不允许
```

注意上面变量的类型为Gender，引用Enum内部值的方式为`Gender::Male`。

Gender类型内部的Male和Female称为枚举类型的值或者枚举类型的成员，还可以称为是枚举类型的实例。反过来思考，不管是Male成员还是Female成员，它们都属于Gender类型，是Gender类型的一种值。就像12_u8是u8类型的其中一个值，属于u8类型。

例如：
```rust
enum Gender {
  Male,
  Female,
}

// 参数类型为Gender
fn is_male(g: Gender) -> bool {
  // ...some code...
}

fn main() {
  // 可传递Gender已枚举的值作为参数
  assert!(is_male(Gender::Male));
  assert!(is_male(Gender::Female));
}
```

再比如，定义一个Choice枚举类型，用来枚举由用户所作出的所有可能选择。

```rust
enum Choice {
  One,
  Two,
  Three,
  Other,
}
```

Choice枚举四种可能的值，其中第四种`Other`表示除前三种选择之外的所有其他选择行为，包括错误的选择(比如通过某种手段选择了不存在的选项)。

Rust中经常会看到类似于Choice的这种用法，在枚举类型中额外使用一个可以归纳剩余所有可能的成员，正如上面的Other归纳了所有其他可能的选择。

其实，前文定义的枚举类型，其每个成员都有对应的**数值**。默认第一个成员对应的数值为0，第二个成员的对应的数值为1，后一个成员的数值总是比其前一个数值大1。并且，可以使用`=`为成员指定数值，但指定值时需注意，不同成员对应的数值不能相同。

例如：
```rust
enum E {
  A,       // 对应数值0
  B,       // 自动加1，对应1
  C = 33,  // 对应33
  D,       // 自动加1，对应34
}
```

定义之后，可使用`as`将enum成员转换为对应的数值。

例如，定义英文的星期和数值相对应的枚举。

```rust
enum Week {
  Monday = 1, // 1
  Tuesday,    // 2
  Wednesday,  // 3
  Thursday,   // 4
  Friday,     // 5
  Saturday,   // 6
  Sunday,     // 7
}

fn main(){
  // mon等于1
  let mon = Week::Monday as i32;
}
```

可在enum定义枚举类型的前面使用`#[repr]`来指定枚举成员的数值范围，超出范围后将编译错误。当不指定类型限制时，Rust尽量以可容纳数据大小的最小类型。例如，最大成员值为100，则用一个字节的u8类型，最大成员值为500，则用两个字节的u16。

```rust
// 最大数值不能超过255
#[repr(u8)]  // 限定范围为`0..=255`
enum E {
  A,
  B = 254,
  C,
  D,        // 256，超过255，编译报错
}
```
