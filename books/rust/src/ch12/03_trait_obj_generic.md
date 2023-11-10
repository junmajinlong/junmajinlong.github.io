## Trait对象和泛型

对比一下Trait对象和泛型：  

- Trait对象可以被看作一种数据类型，它总是以引用的方式被使用，在运行期间，它在栈中保存了具体类型的实例数据和实现自该Trait的方法。  
- 泛型不是一种数据类型，它可被看作是数据类型的参数形式或抽象形式，在编译期间会被替换为具体的数据类型  

Trait Objecct方式也称为**动态分派**(dynamic dispatch)，它在程序运行期间动态地决定具体类型。而Rust泛型是**静态分派**，它在编译期间会代码膨胀，将泛型参数转变为使用到的每种具体类型。

例如，类型Square和类型Rectangle都实现了Trait Area以及方法get_area，现在要创建一个vec，这个vec中包含了任意能够调用get_area方法的类型实例。这种需求建议采用Trait Object方式：

```rust
fn main(){
  let mut sharps: Vec<&dyn Area> = vec![];
  sharps.push(&Square(3.0));
  sharps.push(&Rectangle(3.0, 2.0));
  println!("{}", sharps[0].get_area());
  println!("{}", sharps[1].get_area());
}

trait Area{
  fn get_area(&self)->f64;
}

struct Square(f64);
struct Rectangle(f64, f64);
impl Area for Square{
  fn get_area(&self) -> f64 {self.0 * self.0}
}
impl Area for Rectangle{
  fn get_area(&self) -> f64 {self.0 * self.1}
}
```

在上面的示例中，Vec sharps用于保存多种不同类型的数据，只要能调用get_area方法的数据都能存放在此，而调用get_area方法的能力，来自于Area Trait。因此，使用动态的类型`dyn Area`来描述所有这类数据。当sharps中任意一个数据要调用get_area方法时，都会从它的vtable中查找该方法，然后调用。

但如果改一下上面示例的需求，不仅要为f64实现上述功能，还要为i32、f32、u8等类型实现上述功能，这时候使用Trait Object就很冗余了，要为每一个数值类型都实现一次。

使用泛型则可以解决这类因数据类型而导致的冗余问题。

```rust
fn main(){
  let sharps: Vec<Sharp<_>> = vec![
    Sharp::Square(3.0_f64),
    Sharp::Rectangle(3.0_f64, 2.0_f64),
  ];
  sharps[0].get_area();
}

trait Area<T> {
  fn get_area(&self) -> T;
}

enum Sharp<T>{
  Square(T),
  Rectangle(T, T),
}

impl<T> Area<T> for Sharp<T>
  where T: Mul<Output=T> + Clone + Copy
{
  fn get_area(&self) -> T {
    match *self {
      Sharp::Rectangle(a, b) => return a * b,
      Sharp::Square(a) => return a * a,
    }
  }
}
```

上面使用了泛型枚举，在这个枚举类型上实现Area Trait，就可以让泛型枚举统一各种类型，使得这些类型的数据都具有`get_area`方法。