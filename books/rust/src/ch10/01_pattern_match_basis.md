## 模式匹配的基本使用

可在如下几种情况下使用模式匹配：

- let变量赋值  
- 函数参数传值时的模式匹配  
- match分支  
- if let  
- while let  
- for迭代的模式匹配  

### let变量赋值时的模式匹配

let变量赋值时的模式匹配：

```rust
let PATTERN = EXPRESSION;
```

变量是一种最简单的模式，变量名位于Pattern位置，赋值时的过程：**将表达式与模式进行比较匹配，并将任何模式中找到的变量名进行对应的赋值**。

例如：

```rust
let x = 5;
let (x, y) = (1, 2);
```

第一条语句，变量x是一个模式，在执行该语句时，将表达式5赋值给找到的变量名x。变量赋值总是可以匹配成功。

第二条语句，将表达式`(1,2)`和模式`(x,y)`进行匹配，匹配成功，于是为找到的变量x和y进行赋值：`x=1,y=2`。

如果模式中的元素数量和表达式中返回的元素数量不同，则匹配失败，编译将无法通过。

```rust
let (x,y,z) = (1,2);  // 失败
```

### 函数参数传值时的模式匹配

为函数参数传值和使用let变量赋值是类似的，本质都是在做模式匹配的操作。

例如：

```rust
fn f1(i: i32){
  // xxx
}

fn f2(&(x, y): &(i32, i32)){
  // yyy
}
```

函数`f1`的参数`i`就是模式，当调用`f1(88)`时，88是表达式，将赋值给找到的变量名i。

函数`f2`的参数`&(x,y)`是模式，调用`f2( &(2,8) )`时，将表达式`&(2,8)`与模式`&(x,y)`进行匹配，并为找到的变量名x和y进行赋值：`x=2,y=8`。

### match分支匹配

match分支匹配的用法非常灵活，此处只做基本的用法介绍，后文还会继续深入其用法。

它的语法为：

```rust
match VALUE {
  PATTERN1 => EXPRESSION1,
  PATTERN2 => EXPRESSION2,
  PATTERN3 => EXPRESSION3,
}
```

其中`=>`左边的是各分支的模式，VALUE将与这些分支逐一进行匹配，`=>`右边的是各分支匹配成功后执行的代码。每个分支后使用逗号分隔各分支，最后一个分支的结尾逗号可以省略(但建议加上)。

**match会从前先后匹配各分支，一旦匹配成功则不再继续向下匹配**。

例如：

```rust
let x = (11, 22);
match x {
  (22, a) => println!("(22, {})", a),   // 匹配失败
  (a, b) => println!("({}, {})", a, b), // 匹配成功，停止匹配
  (a, 11) => println!("({}, 11)", a),   // 匹配失败
}
```

如果某分支对应的要执行的代码只有一行，则直接编写该行代码，如果要执行的代码有多行，则需加上大括号包围这些代码。**无论加不加大括号，每个分支都是一个独立的作用域**。

因此，上述match的语法可衍生为如下两种语法：

```rust
match VALUE {
  PATTERN1 => code1,
  PATTERN2 => code2,
  PATTERN3 => code3,
}

match VALUE {
  PATTERN1 => { 
    code line 1
    clod line 2
  },
  PATTERN2 => { 
    code line 1
    clod line 2
  },
  PATTERN3 => code1,
}
```

另外，match结构自身也是表达式，它有返回值，且可以赋值给变量。match的返回值由每个分支最后执行的那行代码决定。Rust要求match的每个分支返回值类型必须相同，且如果是一个单独的match表达式而不是赋值给变量时，每个分支必须返回`()`类型。

例如：

```rust
let x = (11,22);

// 正确，match没有赋值给变量，分支必须返回Unit值()
match x {
  (a, b) => println!("{}, {}", a, b), // 返回Unit值()
  // 其他正确写法：{println!("{}, {}", a, b);}, 
  // 错误写法：     println!("{}, {}", a, b);, 
}

// 正确，每个分支都返回Unit值()
match x {
  (a,11) => println!("{}", a),  // 该分支匹配失败
  (a,b) => println!("{}, {}", a, b), // 将匹配该分支
}

// match返回值赋值给变量，每个分支必须返回相同的类型：i32
let y = match x {
  (a,11) => {
    println!("{}", a);
    a      // 该分支的返回值：i32类型
  },
  (a,b) => {
    println!("{}, {}", a, b);
    a + b  // 该分支的返回值：i32类型
  },
};
```

**match也经常用来穷举Enum类型的所有成员。此时要求穷尽所有成员，如果有遗漏成员，编译将失败**。可以将`_`作为最后一个分支的PATTERN，它将匹配剩余所有成员。

```rust
enum Direction {
  Up,
  Down,
  Left,
  Right,
}
fn main(){
  let dir = match Direction::Down {
    Direction::Up => 1,
    Direction::Down => 2,
    Direction::Right => 3,
    _ => 4,
  };
  println!("{}", dir);
}
```

### if let

`if let`是match的一种特殊情况的语法糖：当只关心一个match分支，其余情况全部由`_`负责匹配时，可以将其改写为更精简`if let`语法。

```rust
if let PATTERN = EXPRESSION {
  // xxx
}
```

这表示将EXPRESSION的返回值与PATTERN模式进行匹配，如果匹配成功，则为找到的变量进行赋值，这些变量在大括号作用域内有效。如果匹配失败，则不执行大括号中的代码。

例如：

```rust
let x = (11, 22);

// 匹配成功，因此执行大括号内的代码
// if let是独立作用域，变量a b只在大括号中有效
if let (a, b) = x {
  println!("{},{}", a, b);
}

// 等价于如下代码
let x = (11, 22);
match x {
  (a, b) => println!("{},{}", a, b),
  _ => (),
}
```

`if let`可以结合`else if`、`else if let`和`else`一起使用。

```rust
if let PATTERN = EXPRESSION {
  // XXX
} else if {
  // YYY
} else if let PATTERN = EXPRESSION {
  // zzz
} else {
  // zzzzz
}
```

这时候它们和match多分支类似。但实际上有很大的不同：使用match分支匹配时，要求分支之间是有关联(例如枚举类型的各个成员)且穷尽的，但Rust编译器不会检查`if let`的模式之间是否有关联关系，也不检查`if let`是否穷尽所有可能情况，因此，即使在逻辑上有错误，Rust也不会给出编译错误提醒。

例如，下面是一个使用了`if let..else if let`的示例，该示例穷举了Enum类型的所有成员，还包括该枚举类型之外的情况，但即使去掉任何一个分支，也都不会报错。

```rust
enum Direction {
  Up,
  Down,
  Left,
  Right,
}

fn main() {
  let dir = Direction::Down;

  if let Direction::Left = dir {
    println!("Left");
  } else if let Direction::Right = dir {
    println!("Right");
  } else {
    println!("Up or Down or wrong");
  }
}
```

### while let

只要`while let`的模式匹配成功，就会一直执行while循环内的代码。

例如：

```rust
let mut stack = Vec::new();
stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
  println!("{}", top);
}
```

当`stack.pop`成功时，将匹配`Some(top)`成功，并将pop返回的值赋值给top，当没有元素可pop时，返回None，匹配失败，于是while循环退出。

### for迭代

for迭代也有模式匹配的过程：为控制变量赋值。例如：

```rust
let v = vec!['a','b','c'];
for (idx, value) in v.iter().enumerate(){
  println!("{}: {}", idx, value);
}
```

