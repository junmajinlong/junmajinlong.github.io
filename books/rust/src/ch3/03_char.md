## char类型

> char官方手册：<https://doc.rust-lang.org/beta/std/primitive.char.html>

char类型是Rust的一种基本数据类型，用于存放**单个unicode字符**，占用4字节空间(32bit)。

在存储char类型数据时，会将其转换为UTF-8编码的数据(即Unicode代码点)进行存储。

char字面量是**单引号**包围的任意单个字符，例如`'a'、'我'`。注意：char和单字符的字符串String是不同的类型。

允许使用反斜线对某些特殊字符转义：

```
字符名      字节字面量
--------------------
单引号      '\''
反斜线      '\\'
换行符      '\n'
换页符      '\r'
制表符      '\t'
```

Rust不会自动将char类型转换为其他类型，但可以进行显式转换：

- 可使用`as`将char转为各种整数类型，目标类型小于4字节时，将从高位截断  
- 可使用`as`将u8类型转char  
  - 之所以不支持其他整数类型，是因为其他整数类型的值可能无法转换为char(即不在UTF-8编码表范围的整数值)  
- 可使用`std::char::from_u32`将u32整数类型转char，返回值`Option<char>`  
  - 如果传递的u32数值不是有效的Unicode代码点，则`from_u32`返回None  
  - 否则返回`Some(c)`，c就是char类型的字符  
- 可使用`std::char::from_digit(INT, BASE)`将十进制的INT转换为BASE进制的char  
  - 如果INT参数不是有效的进制数，返回None  
  - 如果BASE超出进制数的合理范围`[1,36]`，将panic  
  - 否则返回`Some(c)`，c就是char类型的字符  

例如：
```rust
// char -> Integer
println!("{}", '我' as i32);     // 25105
println!("{}", '是' as u16);     // 26159
println!("{}", '是' as u8);      // 47，被截断了

// u8 -> char
println!("{}", 97u8 as char);    // a

// std::char
use std::char;

println!("{}", char::from_u32(0x2764).unwrap());  // ❤
assert_eq!(char::from_u32(0x110000), None);  // true

println!("{}", char::from_digit(4,10).unwrap());  // '4'
println!("{}", char::from_digit(11,16).unwrap()); // 'b'
assert_eq!(char::from_digit(11,10),None); // true
```

