## 为枚举类型定义方法

和Struct类型一样，也可以使用`impl`关键字为枚举类型定义方法。

例如，定义包含星期一到星期日的枚举类型Week，然后定义一个方法来判断给定的某一天是否是周末。

```rust
#[derive(Copy, Clone)]
enum Week {
  Monday = 1,
  Tuesday,
  Wednesday,
  Thursday,
  Friday,
  Saturday,
  Sunday,
}

impl Week {
  fn is_weekend(&self) -> bool {
    if (*self as u8) > 5 {
      return true;
    }
    false
  }
}

fn main(){
  let d = Week::Thursday;
  println!("{}", d.is_weekend());
}
```


