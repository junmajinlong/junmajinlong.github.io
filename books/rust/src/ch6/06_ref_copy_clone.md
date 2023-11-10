## 引用类型的Copy和Clone

引用类型是可Copy的，所以引用类型在Move的时候都会Copy一个引用的副本，Copy前后的引用都指向同一个目标值，这很容易理解。
```rust
let a = "hello world".to_string();

// b和c都是a的引用
let b = &a;
let c = b;  // Copy引用
```

引用类型也是可Clone的(实现Copy的时候要求也必须实现Clone，所以可Copy的类型也是可Clone的)，但是引用类型的clone()需注意。

例如：
```rust
struct Person;

let a = Person;
let b = &a;
let c = b.clone();  // c的类型是&Person
```

如果使用了clippy工具检查代码，clippy将对上面的`b.clone()`给出错误提示：
```
using `clone` on a double-reference; this will copy the reference of type `&strategy::Strategy::run::Person` instead of cloning the inner type
```

提示说明，对引用clone()时，将拷贝引用类型本身，而不是去拷贝引用所指向的数据本身，所以变量c的类型是`&Person`。这里引用的clone()逻辑，看上去似乎没有问题，但是却发出了错误提示。

但如果，在引用所指向的类型上去实现Clone，再去clone()引用类型，将没有错误提示。
```rust
#[derive(Clone)]
struct Person;

let a = Person;
let b = &a;
let c = b.clone();  // c的类型是Person，而不是&Person
```

注意上面`b.clone()`得到的类型是引用所指向数据的类型，即`Person`，而不是像之前示例中的那样得到`&Person`。

前后两个示例的区别，仅在于引用所指向的类型`Person`有没有实现Clone。所以得到结论：  

- **没有实现Clone时，引用类型的clone()将等价于Copy，但cilppy工具的错误提示说明这很可能不是我们想要的克隆效果**  
- **实现了Clone时，引用类型的clone()将克隆并得到引用所指向的类型**  

同一种类型的同一个方法，调用时却产生两种效果，之所以会有这样的区别，是因为：  

- 方法调用的符号`.`会自动解引用  
- 方法调用前会先查找方法，查找方法时有优先级，找得到即停。由于解引用的前和后是两种类型(解引用前是引用类型，解引用后是引用指向的类型)，如果这两种类型都实现了同一个方法(比如`clone()`)，Rust编译器将按照方法查找规则来决定调用哪个类型上的方法，参考(<https://rustc-dev-guide.rust-lang.org/method-lookup.html?highlight=lookup#method-lookup>)  

为什么clone引用的时候，clippy工具会提示这很可能不是我们想要的行为呢？一方面，拷贝一个引用得到另一个引用副本是很常见的需求，但是这个需求有Copy就够了，另一方面，正如clippy所提示的，能够拷贝引用背后的数据也是非常有必要的。

例如，某方法要求返回Person类型，但在该方法内部却只能取得Person的引用类型(比如从HashMap的`get()`方法只能返回值的引用)，所以需要将引用`&Person`转换为`Person`，直接解引用是一种可行方案，但是对未实现Copy的类型去解引用，将会执行Move操作，很多时候这是不允许的，比如不允许将已经存入HashMap中的值Move出来，此时最简单的方式，就是通过克隆引用的方式得到Person类型。

> 提醒：正因为从集合(比如HashMap、BTreeMap等)中取数据后很有可能需要对取得的数据进行克隆，因此建议不要将大体量的数据存入集合，如果确实需要克隆集合中的数据的话，这将会严重影响性能。
>
> 作为建议，可以考虑先将大体量的数据封装在智能指针(比如Box、Rc等)的背后，再将智能指针存入集合。
>
> 其它语言中集合类型的使用可能非常简单直接，但Rust中需要去关注这一点。