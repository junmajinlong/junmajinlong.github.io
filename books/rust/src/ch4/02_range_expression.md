## 范围(Range)表达式

Rust支持范围操作符，有以下几种表示范围的操作符：


| 范围表达式    | 类型                       | 表示的范围        |
| ----------- | -------------------------- | --------------- |
| start..end  | std::ops::Range            | start ≤ x < end |
| start..     | std::ops::RangeFrom        | start ≤ x       |
| ..end       | std::ops::RangeTo          | x < end         |
| ..          | std::ops::RangeFull        | -               |
| start..=end | std::ops::RangeInclusive   | start ≤ x ≤ end |
| ..=end      | std::ops::RangeToInclusive | x ≤ end         |

例如，`1..5`表示1、2、3、4共四个整数，`1..=5`表示1、2、3、4、5共五个整数。

需注意的是其中表示全范围的表达式`..`，它表示可以尽可能地生成下一个数，直到无法生成为止。

在生成Slice的时候，需要使用到范围表达式。例如，从数组生成Slice：

```rust
let arr = [11, 22, 33, 44, 55];
let s1 = &arr[0..3];    // [11,22,33]
let s2 = &arr[1..=3];   // [22, 33, 44]
let s3 = &arr[..];      // [11, 22, 33, 44, 55]
```

范围表达式也常被用于迭代操作。例如`for`语句：

```rust
for i in 1..5 {
  println!("{}", i);  // 1 2 3 4
}
```

另外，范围表达式和对应类型的实例是等价的。例如，下面两个表示范围的方式是等价的：

```rust
let x = 0..5;
let y = std::ops::Range {start: 0, end: 5};
```

