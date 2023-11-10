## 流程控制结构

流程控制结构包括：  
- if条件判断结构  
- loop循环  
- while循环  
- for..in迭代  

除此之外，还有其他几种本节暂不介绍的控制结构。

需要说明的是，Rust中这些结构都是表达式，它们都有默认的返回值`()`，且if结构和loop循环结构可以指定返回值。

> 注：【这些结构的默认返回值是`()`】的说法是不严谨的
>
> 之所以可以看作是默认返回`()`，是因为Rust会在每个分号结尾的语句后自动加上小括号`()`，使得语句看上去也有了返回值。
>
> 为了行文简洁，下文将直接描述为默认返回值。

### if..else

if语句的语法如下：

```rust
if COND1 {
  ...
} else if COND2 {
  ...
} else {
  ...
}
```

其中，条件表达式COND不需要加括号，且COND部分只能是布尔值类型。另外，`else if`分支是可选的，且可以有多个，`else`分支也是可选的，但最多只能有一个。

由于if结构是表达式，**它有返回值，所以可以将if结构赋值给一个变量**(或者其他需要值的地方)。

但是要注意，if结构默认返回Unit类型的`()`，这个返回值是没有意义的。如果要指定为其他有意义的返回值，要求：  
- **分支最后执行的那一行代码不使用分号结尾**，这表示将最后执行的这行代码的返回值作为if结构的返回值  
- **每个分支的返回值类型相同**，这意味着每个分支最后执行的代码都不能使用分号结尾  
- **必须要有else分支**，否则会因为所有分支条件判断都不通过而直接返回if的默认返回值`()`  

下面用几个示例来演示这几个要求。

首先是一段正确的代码片段：  

```rust
let x = 33;

// 将if结构赋值给变量a
// 下面if的每个分支，其返回值类型都是i32类型
let a = if x < 20 {
  // println!()不是该分支最后一条语句，要加结尾分号
  println!("x < 20");
  // x+10是该分支最后一条语句，
  // 不加分号表示将其计算结果返回，返回类型为i32
  x + 10  
} else if x < 30 {
  println!("x < 30");
  x + 5   // 返回x + 5的计算结果，返回类型为i32
} else {
  println!("x >= 30");
  x     // 直接返回x，返回类型为i32
};  // if最后一个闭大括号后要加分号，这是let的分号
```

下面是一段将if默认返回值`()`赋值给变量的代码片段：

```rust
let x = 33;

// a被赋值为`()`
let a = if x < 20 {
  println!("x < 20");
};
println!("{:?}", a);  // ()
```

下面不指定else分支，将报错：

```rust
let x = 33;

// if分支返回i32类型的值
// 但如果没有执行if分支，则返回默认值`()`
// 这使得a的类型不是确定的，因此报错
let a = if x < 20 {
  x + 3   // 该分支返回i32类型
};
```

下面if分支和else if分支返回不同类型的值，将报错：

```rust
let x = 33;

let a = if x < 20 {
  x + 3      // i32类型
} else if x < 30 {
  "hello".to_string()  // String类型
} else {
  x   // i32类型
};
```

由于if的条件表达式COND部分要求必须是布尔值类型，因此不能像其他语言一样编写类似于`if "abc" {}`这样的代码。但是，却可以在COND部分加入其他语句，只要保证COND部分的返回值是bool类型即可。

例如下面的代码。注意下面使用大括号`{}`语句块包围了if的COND部分，使得可以先执行其他语句，在语句块的最后才返回bool值作为if的分支判断条件。

```rust
let mut x = 0;
if {x += 1; x < 3} {
  println!("{}", x);
}
```

这种用法在if结构上完全是多此一举的，但COND的这种用法也适用于while循环，有时候会有点用处。

### while循环

while循环的语法很简单：

```rust
while COND {
  ...
}
```

其中，条件表达式COND和if结构的条件表达式规则完全一致。

如果要中途退出循环，可使用`break`关键字，如果要立即进入下一轮循环，可使用`continue`关键字。

例如：

```rust
let mut x = 0;

while x < 5 {
  x += 1;
  println!("{}", x);
  if x % 2 == 0 {
    continue;
  }
}
```

根据前文对if的条件表达式COND的描述，COND部分允许加入其他语句，只要COND部分最后返回bool类型即可。例如：

```rust
let mut x = 0;

// 相当于do..while
while {println!("{}", x);x < 5} {
  x += 1;
  if x % 2 == 0 {
    continue;
  }
}
```

最后，while虽然有默认返回值`()`，但`()`作为返回值是没有意义的。因此，不考虑while的返回值问题。

### loop循环

loop表达式是一个无限循环结构。只有在loop循环体内部使用`break`才能终止循环。另外，也使用`continue`可以直接跳入下一轮循环。

例如，下面的循环结构将输出1、3。

```rust
let mut x = 0;
loop {
  x += 1;
  if x == 5 {
    break;
  }
  if x % 2 == 0 {
    continue;
  }
  println!("{}", x);
}
```

loop也有默认返回值`()`，可以将其赋值给变量。例如，直接将上例的loop结构赋值给变量a：

```rust
let mut x = 0;
let a = loop {
  ...
};

println!("{:?}", a);   // ()
```

作为一种特殊情况，当**在loop中使用break时，break可以指定一个loop的返回值**。

```rust
let mut x = 0;
let a = loop {
  x += 1;
  if x == 5 {
    break x;    // 返回跳出循环时的x，并赋值给变量a
  }
  if x % 2 == 0 {
    continue;
  }
  println!("{}", x);
};
println!("var a: {:?}", a); // 输出 var a: 5
```

注意，只有loop中的break才能指定返回值，在while结构或for迭代结构中使用的break不具备该功能。

### for迭代

Rust中的for只具备迭代功能。迭代是一种特殊的循环，每次从数据的集合中取出一个元素是一次迭代过程，直到取完所有元素，才终止迭代。

例如，Range类型是支持迭代的数据集合，Slice类型也是支持迭代的数据集合。

但和其他语言不一样，Rust数组不支持迭代，要迭代数组各元素，需将数组转换为Slice再进行迭代。

```rust
// 迭代Range类型：1..5
for i in 1..5 {
  println!("{}", i);
}

let arr = [11, 22, 33, 44];
// arr是数组，&arr转换为Slice，Slice可迭代
for i in &arr {
  println!("{}", i);
}
```

### 标签label

可以为loop结构、while结构、for结构指定标签，break和continue都可以指定标签来确定要跳出哪一个层次的循环结构。

例如：

```rust
// 'outer和'inner是标签名
'outer: loop {
  'inner: while true {
    break 'outer;  // 跳出外层循环
  }
}
```

需注意，loop结构中的break可以同时指定标签和返回值，语法为`break 'label RETURN_VALUE`。

例如：

```rust
let x = 'outer: loop {
  'inner: while true {
    break 'outer 3;
  }
};

println!("{}", x);   // 3
```
