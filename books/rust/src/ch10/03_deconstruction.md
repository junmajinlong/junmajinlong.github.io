## 模式解构赋值

模式匹配时可用于解构赋值，可解构的类型包括struct、enum、tuple、slice等等。

解构赋值时，可使用`_`作为某个变量的占位符，使用`..`作为剩余所有变量的占位符(使用`..`时不能产生歧义，例如`(..,x,..)`是有歧义的)。当解构的类型包含了命名字段时，可使用`fieldname`简化`fieldname: fieldname`的书写。

### 解构struct

解构Struct时，会将待解构的struct各个字段和Pattern中的各个字段进行匹配，并为找到的字段变量进行赋值。

当Pattern中的字段名和字段变量同名时，可简写。例如`P{name: name, age: age}`和`P{name, age}`是等价的Pattern。

```rust
struct Point2 {
  x: i32,
  y: i32,
}

struct Point3 {
  x: i32,
  y: i32,
  z: i32,
}

fn main(){
  let p = Point2{x: 0, y: 7};
  
  // 等价于 let Point2{x: x, y: y} = p;
  let Point2{x, y} = p;
  println!("x: {}, y: {}", x, y);
  // 解构时可修改字段变量名: let Point2{x: a, y: b} = p;
  // 此时，变量a和b将被赋值
   
  let ori = Point{x: 0, y: 0, z: 0};
  match ori{
    // 使用..忽略解构后剩余的字段
    Point3 {x, ..} => println!("{}", x),
  }
}
```

### 解构enum

例如：

```rust
enum IPAddr {
  IPAddr4(u8,u8,u8,u8),
  IPAddr6(String),
}

fn main(){
  let ipv4 = IPAddr::IPAddr4(127,0,0,1);
  match ipv4 {
    // 丢弃解构后的第四个值
    IPAddr::IPAddr4(a,b,c,_) => println!("{},{},{}", a,b,c),
    IPAddr::IPAddr6(s) => println!("{}", s),
  }
}
```

### 解构元组

```rust
let ((feet, inches), Point {x, y}) = ((3, 1), Point { x: 3, y: -1 });
```

### @绑定变量名

当解构后进行模式匹配时，如果某个值没有对应的变量名，则可以使用`@`手动绑定一个变量名。

例如：

```rust
struct S(i32, i32);

match S(1, 2) {
  // 如果匹配1成功，将其赋值给变量z
  // 如果匹配2成功，也将其赋值给变量z
    S(z @ 1, _) | S(_, z @ 2) => assert_eq!(z, 1),
    _ => panic!(),
}
```

再例如，匹配并解构一个数组：

```rust
let arr = ["x", "y", "z"];
match arr {
  [.., "!"] => println!("!!!"),
  // 匹配成功，start = ["x", "y"]
  [start @ .., "z"] => println!("starts: {:?}", start),
  ["a", end @ ..] => println!("ends: {:?}", end),
  rest => println!("{:?}", rest),
}
```

## ref和mut修饰模式中的变量

当进行解构赋值时，很可能会将变量拥有的所有权转移出去，从而使得原始变量变得不完整或直接失效。

```rust
struct Person{
  name: String,
  age: i32,
}

fn main(){
  let p = Person{name: String::from("junmajinlong"), age: 23};
  let Person{name, age} = p;
  
  println!("{}", name);
  println!("{}", age);
  println!("{}", p.name);  // 错误，name字段所有权已转移
}
```

如果想要在解构赋值时不丢失所有权，有以下几种方式：

```rust
// 方式一：解构表达式的引用
let Person{name, age} = &p;

// 方式二：解构表达式的克隆，适用于可调用clone()方法的类型
// 但Person struct没有clone()方法

// 方式三：在模式的某些字段或元素上使用ref关键字修饰变量
let Person{ref name, age} = p;
let Person{name: ref n, age} = p;
```

在模式中使用`ref`修饰变量名相当于对被解构的**字段或元素**上使用`&`进行引用。

```rust
let x = 5_i32;         // x的类型：i32
let x = &5_i32;        // x的类型：&i32
let ref x = 5_i32;     // x的类型：&i32
let ref x = &5_i32;    // x的类型：&&i32
```

因此，使用ref修饰了模式中的变量后，解构赋值时对应值的所有权就不会发生转移，而是以只读的方式借用给该变量。

如果想要对解构赋值的变量具有数据的修改权，需要使用`mut`关键字修饰模式中的变量，但这样会转移原值的所有权，此时可不要求原变量是可变的。

```rust
#[derive(Debug)]
struct Person {
  name: String,
  age: i32,
}

fn main() {
  let p = Person {
    name: String::from("junma"),
    age: 23,
  };
  match p {
    Person { mut name, age } => {
      name.push_str("jinlong");
      println!("name: {}, age: {}", name, age)
    },
  }
  //println!("{:?}", p);    // 错误
}
```

如果不想在可修改数据时丢失所有权，可在mut的基础上加上ref关键字，就像`&mut xxx`一样。

```rust
#[derive(Debug)]
struct Person {
  name: String,
  age: i32,
}

fn main() {
  let mut p = Person {   // 这里要改为mut p
    name: String::from("junma"),
    age: 23,
  };
  match p {
    // 这里要改为ref mut name
    Person { ref mut name, age } => {
      name.push_str("jinlong");
      println!("name: {}, age: {}", name, age)
    },
  }
  println!("{:?}", p);
}
```

注意，使用`ref`修饰变量只是**借用了被解构表达式的一部分值，而不是借用整个值**。如果要匹配的是一个引用，则使用`&`。

```rust
let a = &(1,2,3);       // a是一个引用
let (t1,t2,t3) = a;     // t1,t2,t3都是引用类型&i32
let &(x,y,z) = a;       // x,y,z都是i32类型
let &(ref xx,yy,zz) = a;  // xx是&i32类型，yy,zz是i32类型
```

最后，也可以将`match value{}`的value进行修饰，例如`match &mut value {}`，这样就不需要在模式中去加ref和mut了。这对于有多个分支需要解构赋值，且每个模式中都需要ref/mut修饰变量的match非常有用。

```rust
fn main() {
  let mut s = "hello".to_string();
  match &mut s {   // 对可变引用进行匹配
    // 匹配成功时，变量也是对原数据的可变引用
    x => x.push_str("world"),
  }
  println!("{}", s);
}
```

## 匹配守卫(match guard)

匹配守卫允许匹配分支添加**额外的后置条件**：当匹配了某分支的模式后，再检查该分支的守卫后置条件，如果守卫条件也通过，则成功匹配该分支。

```rust
let x = 33;
match x {
  // 先范围匹配，范围匹配成功后，再检查是否是偶数
  // 如果范围匹配没有成功，则不会检查后置条件
  0..=50 if x % 2 == 0 => {
    println!("x in [0, 50], and it is an even");
  },
  0..=50 => println!("x in [0, 50], but it is not an even"),
  _ => (),
}
```

注意，后置条件的优先级很低。例如：

```rust
// 下面两个分支的写法等价
4 | 5 | 6 if bool_expr => println!("yes"),
(4 | 5 | 6) if bool_expr => println!("yes"),
```

## 注意(1)：对引用进行解构赋值时

在解构赋值时，**如果解构的是一个引用，则被匹配的变量也将被赋值为对应元素的引用**。

```rust
let t = &(1,2,3);    // t是一个引用
let (t0,t1,t2) = t;  // t0,t1,t2的类型都是&i32
let t0 = t.0;   // t0的类型是i32而不是&i32，因为t.0等价于(*t).0
let t0 = &t.0;  // t0的类型是&i32而不是i32，&t.0等价于&(t.0)而非(&t).0
```

因此，当使用模式匹配语法`for i in t`进行迭代时：  

- 如果t不是一个引用，则t的每一个元素都会move给i  
- 如果t是一个引用，则i将是每一个元素的引用   
- 同理，`for i in &mut t`和`for i in mut t`也一样   

## 注意(2)：对解引用进行匹配时

当`match VALUE`的VALUE是一个解引用`*xyz`时(因此，xyz是一个引用)，可能会发生所有权的转移，此时可使用`xyz`或`&*xyz`来代替`*xyz`。具体原因请参考：[解引用(deref)的所有权转移问题](../ch6/05_re_understand_move.md#whenmove)。

下面是一个示例：

```rust
fn main() {
  // p是一个Person实例的引用
  let p = &Person {
    name: "junmajinlong".to_string(),
    age: 23,
  };
  
  // 使用&*p或p进行匹配，而不是*p
  // 使用*p将报错，因为会转移所有权
  match &*p {
    Person {name, age} =>{
      println!("{}, {}",name, age);
    },
    _ => (),
  }
}

struct Person {
  name: String,
  age: u8,
}
```

