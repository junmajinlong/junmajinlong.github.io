## vec的基本使用

创建向量有几种方式：  

- `Vec::new()`创建空的vec  
- `Vec::with_capacity()`创建空的vec，并将其容量设置为指定的数量  
- `vec![]`宏创建并初始化vec(中括号可以换为小括号或大括号)  
- `vec![v;n]`创建并初始化vec，共n个元素，每个元素都初始化为v  

```rust
fn main(){
  let mut v1 = Vec::new();
  // 追加元素时，将根据所追加的元素推导v1的数据类型Vec<i32>
  v1.push(1);  // push()向vec尾部追加元素
  v1.push(2);
  v1.push(3);
  v1.push(4);
  assert_eq!(v1, [1,2,3,4]) // vec可以直接和数组进行比较

  // v2的类型推导为：Vec<i32>
  let v2 = vec![1,2,3,4];
  assert_eq!(v2, [1,2,3,4]);
  
  let v3 = vec!(3;4);  // 等价于vec![3,3,3,3]
  assert_eq!(v3, [3,3,3,3]);
  
  // 创建容量为10的空vec
  let mut v4 = Vec::with_capacity(10);
  v4.push(33);
}
```

## 访问和遍历vec

可以使用索引来访问vec中的元素。索引越界访问时，将在运行时panic报错。

索引是usize类型的值，因此不接受负数索引。

```rust
fn main(){
  let v = vec![11,22,33,44];
  let n: usize = 3;
  println!("{},{}", v[0], v[n]);
  
  // 越界，报错
  // 运行错误而非编译错误，因为运行期间才知道vec长度
  // println!("{}", v[9]);
}
```

如果不想要在越界访问vec时panic中断程序，可使用：

- `get()`来获取指定索引处的元素引用或范围内元素的引用，如果索引越界，返回`None`。  
- `get_mut()`来获取元素的可变引用或范围内元素的可变引用，如果索引越界，返回`None`。  

这两个方法的返回值可能是所取元素的引用，也可能是`None`，此处不对`None`展开介绍，相关的细节要留到`Option`类型中介绍。这里只需要知道，当所调用函数的返回值可能是一个具体值，也可能是None时，需要对这两种可能的返回值进行处理。比较简单的一种处理方式是在该函数返回结果上使用`unwrap()`方法：当成功返回具体值时，unwrap()将返回该值，当返回None时， unwrap()将panic报错退出。

例如：

```rust
fn main(){
  let v = [11,22,33,44];
  // 取得index=3处元素，成功，于是unwrap()提取得到44
  let n = v.get(3).unwrap();
  println!("{}", n);
  
  // 取得index=4处元素，失败，于是panic报错
  // let nn = v.get(4).unwrap(); 
}
```

另外，Vec是可迭代的，可以直接使用`for x in vec {}`来遍历vec。

```rust
let v = vec![11,22,33,44];
for i in v {
  println!("{}", i);
}
```

