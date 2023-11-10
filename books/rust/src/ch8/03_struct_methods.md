## 定义Struct的方法

Struct就像面向对象的类一样，Rust允许为Struct定义实例方法和关联方法，实例方法可被所有实例对象访问调用，关联方法类似于其他语言中的类方法或静态方法。

定义Struct的方法的语法为`impl Struct_Name {}`，所有方法定义在大括号中。

### 定义Struct的实例方法

实例方法是所有实例对象可访问、调用的方法。

例如：
```rust
struct Rectangle{
  width: u32,
  height: u32,
}

impl Rectangle {
  fn area(&self) -> u32 {
    self.width * self.height
  }

  fn perimeter(&self) -> u32 {
    (self.width + self.height) * 2
  }
}

fn main() {
  let rect1 = Rectangle{width: 30, height: 50};
  println!("{},{}", rect1.area(), rect1.perimeter());
}
```

也可以将方法定义在多个`impl Struct_Name {}`中。如下：

```rust
impl Rectangle {
  fn area(&self) -> u32 {
    self.width * self.height
  }

  fn perimeter(&self) -> u32 {
    (self.width + self.height) * 2
  }
}

impl Rectangle {
  fn include(&self, other: &Rectangle) -> bool {
    self.width > other.width && self.height > other.height
  }
}
```

所有Struct的实例方法的第一个参数都是self(的不同形式)。self表示调用方法时的Struct实例对象(如`rect1.area()`时，self就是rect1)。有如下几种self形式：  
- `fn f(self)`：当`obj.f()`时，转移obj的所有权，调用f方法之后，obj将无效  
- `fn f(&self)`：当`obj.f()`时，借用而非转移obj的只读权，方法内部不可修改obj的属性，调用f方法之后，obj依然可用  
- `fn f(&mut self)`：当`obj.f()`时，借用obj的可写权，方法内部可修改obj的属性，调用f方法之后，obj依然可用  

定义方法时很少使用第一种形式`fn f(self)`，因为这会使得调用方法后对象立即消失。但有时候也能派上场，例如可用于替换对象：调用方法后原对象消失，但返回另一个替换后的对象。

如果仔细观察的话，会发现方法的第一个参数self(或其他形式)没有指定类型。实际上，在方法的定义中，self的类型为`Self`(首字母大写)。例如，为Rectangle定义方法时，Self类型就是Rectangle类型。因此，下面几种定义方法的方式是等价的：

```rust
fn f(self)
fn f(self: Self)

fn f(&self)
fn f(self: &Self)

fn f(&mut self)
fn f(self: &mut Self)
```

### Rust的自动引用和解引用

在C/C++语言中，有两个不同的运算符来调用方法：`.`直接在对象上调用方法，`->`在一个对象指针上调用方法，这时需要先解引用(dereference)指针。

换句话说，如果obj是一个指针，那么`obj->something()`就像`(*obj).something()`一样。更典型的是Perl，Perl的对象总是引用类型，因此它调用方法时总是使用`obj->m()`形式。

**Rust不会自动引用或自动解除引用**，但有例外：当使用`.`运算符和比较操作符(如`= > >=`)时，Rust会**自动创建引用和解引用**，并且会尽可能地解除多层引用：   

- (1).方法调用`v.f()`会自动解除引用或创建引用  
- (2).属性访问`p.name`或`p.0`会自动解除引用  
- (3).比较操作符的两端如果**都是**引用类型，则自动解除引用  
- (4).能自动解除的引用包括普通引用`&x`、`Box<T>`、`Rc<T>`等  

对于(1)，方法调用时的自动引用和自动解除引用，它是这样工作的：当使用`ob.something()`调用方法时，Rust会根据所调用方法的签名进行推断(即根据方法的接收者self参数的形式进行推断)，然后自动为object添加`&, &mut`来创建引用或添加`*`来自动解除引用，其目的是让obj与方法签名相匹配。

也就是说，当distance方法的第一个形参是`&self`或`&mut self`时，下面代码是等价的，但第一行看起来简洁的多：
```rust
p1.distance(&p2); 
(&p1).distance(&p2);
```

## 关联函数(associate functions)

关联函数是指第一个参数不是self(的各种形式)但和Struct有关联关系的函数。关联方法类似于其他语言中类方法或静态方法的概念。

调用关联方法的语法`StructName::func()`。例如，`String::from()`就是在调用String的关联方法from()。

例如，可以定义一个专门用于构造实例对象的关联函数new。

```rust
struct Rectangle {
  width: u32,
  height: u32,
}

impl Rectangle {
  // 关联方法new：构造Rectangle的实例对象
  fn new(width: u32, height: u32) -> Rectangle {
    Rectangle { width, height }
  }
}

impl Rectangle {
  fn area(&self) -> u32 { self.width * self.height }
}

fn main() {
  // 调用关联方法
  let rect1 = Rectangle::new(30, 50);
  let rect2 = Rectangle::new(20, 50);
  println!("{}", rect1.area());
  println!("{}", rect2.area());
}
```

实际上，实例方法也属于关联方法，也可以采用关联方法的形式去调用，只不过这时需要手动传递第一个self参数。例如：

```rust
// 调用Rectangle的area方法，并传递参数&self
Rectangle::area(  &rect1  );
```

