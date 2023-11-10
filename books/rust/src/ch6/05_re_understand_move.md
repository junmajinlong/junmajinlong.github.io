## 再次理解Move

前面对Move、Copy和所有权相关的内容做了详细的解释，相信变量赋值、函数传参时的所有权问题应该不再难理解。

但是，所有权的转移并不仅仅只发生在这两种相对比较明显的情况下。例如，解引用操作也需要转移所有权。

```rust
let v = &vec![11, 22];
let vv = *v;
```

上面会报错：

```
error[E0507]: cannot move out of `*v` which is behind a shared reference
```

从位置表达式和值的角度来思考也不难理解：**当产生了一个位置，且需要向位置中放入值，就会发生移动**([Moved and copied types](https://doc.rust-lang.org/reference/expressions.html#moved-and-copied-types))。只不过，这个值可能来自某个变量，可能来自计算结果(即来自于中间产生的临时变量)，这个值的类型可能实现了Copy Trait。

对于上面的示例来说，`&vec![11, 22]`中间产生了好几个临时变量，但最终有一个临时变量是vec的所有者，然后对这个变量进行引用，将引用赋值给变量v。使用`*v`解引用时，也产生了一个临时变量保存解引用得到的值，而这里就出现了问题。因为变量v只是vec的一个引用，而不是它的所有者，它无权转移值的所有权。

下面几个示例，将不难理解：

```rust
let a = &"junmajinlong.com".to_string();
// let b = *a;         // (1).取消注释将报错
let c = (*a).clone();  // (2).正确
let d = &*a;           // (3).正确

let x = &3;
let y = *x;      // (4).正确
```

注意，不要使用`println!("{}", *a);`或类似的宏来测试，这些宏不是函数，它们真实的代码中使用的是`&(*a)`，因此不会发生所有权的转移。

虽说【当产生了一个位置，且需要向位置中放入值，就会发生移动】这句话很容易理解，但有时候很难发现深层次的移动行为。

## 被丢弃的move

下面是一个容易令人疑惑的示例：

```rust
fn main(){
  let x = "hello".to_string();
  x;   // 发生Move
  println!("{}", x);  // 报错：value borrowed here after move
}
```

从这个示例来看，【当值需要放进位置的时候，就会发生移动】，这句话似乎不总是正确，第三行的`x;`取得了x的值，但是它直接被丢弃了，所以x也被消耗掉了，使得println中使用x报错。实际上，这里也产生了位置，它等价于`let _tmp = x;`，即将值移动给了一个临时变量。

如果上面的示例不好理解，那下面有时候会排上用场的示例，会有助于理解：

```rust
fn main() {
    let x = "hello".to_string();
    let y = {
        x // 发生Move，注意没有结尾分号
    };
    println!("{}", x); // 报错：value borrowed here after move
}
```

从结果上来看，语句块将x通过返回值的方式移出来赋值给了y，所以认为x的所有权被转移给了y。实际上，语句块中那唯一的一行代码本身就发生了一次移动，将x的所有权移动给了临时变量，然后返回时又发生了一次移动。

<a name="whenmove"></a>

## 什么时候Move：使用值的时候

上面的结论说明了一个问题：虽然多数时候产生位置的行为是比较明确的，但少数时候却非常难发现，也难以理解。

可以换个角度来看待：**当使用值的时候，就会产生位置，就会发生移动**。

如果翻阅`Rust Reference`文档，就会经常性地看到类似这样的说法(例如[Negation operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html#negation-operators))：

```
xxx are evaluated in value expression context so are moved or copied.
```

这里需要明确：`value expression`表示的是会产生值的表达式，`value expression context`表示的是使用值的上下文。

有哪些地方会使用值呢？除了比较明显的会移动的情况，还有一些隐式的移动(或Copy)：  

- 方法调用的真实接收者，如`a.meth()`，a会被移动(注意，a可能会被自动加减引用，此时a不是方法的真实接收者)  
- 解引用时会Move(注意，解引用会得到那个值，但不一定会消耗这个值，有可能只是借助这个值去访问它的某个字段、或创建这个值的引用，这些操作可以看作是借值而不是使用值)  
- 字段访问时会Move那个字段  
- 索引访问时，会Move那个元素  
- 大小比较时，会Move(注意，`a > b`比较时会先自动取a和b的引用，然后再增减a和b的引用直到两边类型相同，因此实际上Move(Copy)的是它们的某个引用，而不会Move变量本身)  

更完整更细致的描述，参考[Expression - Rust Reference](https://doc.rust-lang.org/reference/expressions.html)。

下面是几个比较常见的容易疑惑的移动示例：

```rust
struct User {name: String}
let user = User {name: "junmajinlong".to_string()};
let name = (&user).name;  // 报错，想要移动name字段，但user正被引用着，此刻不允许移走它的一部分

let user1 = *(&user);  // 报错，解引用临时变量时触发移动，此时user正被引用着
let u = &user;
let user2 = &(*u);  // 不报错，解引用得到值后，对这个值创建引用，不会消耗值

impl User {
  fn func(&self) {
    let xx = *self; // 报错，解引用报错，self自身不是所有者，例如user.func()时，user才是所有者
    
    if (*self).name < "hello".to_string(){} // 不报错，比较时会转换为&((*self).name) < &("hello".to_string())
  }
}
```



