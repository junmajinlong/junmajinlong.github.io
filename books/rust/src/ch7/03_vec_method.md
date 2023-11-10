## vec的常用方法

vec自身有很多方法，另外vec还可以调用所有Slice类型的方法。

下面是vec自身提供的一些常见的方法，更多方法和它们更详细的用法，参考官方手册：<https://doc.rust-lang.org/std/vec/struct.Vec.html>。

- len()：返回vec的长度(元素数量)  
- is_empty()：vec是否为空  
- push()：在vec尾部插入元素  
- pop()：删除并返回vec尾部的元素，vec为空则返回None  
- insert()：在指定索引处插入元素  
- remove()：删除指定索引处的元素并返回被删除的元素，索引越界将panic报错退出  
- clear()：清空vec  
- append()：将另一个vec中的所有元素追加移入vec中，移动后另一个vec变为空vec  
- truncate()：将vec截断到指定长度，多余的元素被删除  
- retain()：保留满足条件的元素，即删除不满足条件的元素  
- drain()：删除vec中指定范围的元素，同时返回一个迭代该范围所有元素的迭代器  
- split_off()：将vec从指定索引处切分成两个vec，索引左边(不包括索引位置处)的元素保留在原vec中，索引右边(包括索引位置处)的元素在返回的vec中  

这些方法的用法都非常简单，下面举一些示例来演示它们。

len()和is_empty()：  
```rust
let v = vec![11,22,33];
assert_eq!(v.len(), 3);
assert!(!v.is_empty());
```

push()、pop()、insert()、remove()和clear()：  
```rust
let mut v = vec![11,22];

v.push(33);      // [11,22,33]

assert_eq!(v.pop(), Some(33));
assert_eq!(v.pop(), Some(22));
assert_eq!(v.pop(), Some(11));
assert_eq!(v.pop(), None);

v.insert(0, 111); // [111]
v.insert(1, 222); // [111,222]
v.insert(2, 333); // [111,222,333]
assert_eq!(v.remove(1), 222);

v.clear();  // []
```

append()：  
```rust
let mut v = vec![11,22];
let mut vv = [33,44,55].to_vec();

v.append(&mut vv);
println!("{:?}", v);   // [11,22,33,44,55]
println!("{:?}", vv);  // []
```

truncate()：截断到指定长度，多余的元素被删除，如果目标长度大于当前长度，则不做任何事  
```rust
let mut v = vec![11,22,33,44];
v.truncate(2);
println!("{:?}", v); // [11, 22]

v.truncate(5);  // 不做任何事
```

retain()：
```rust
let mut v = vec![11, 22, 33, 44];

v.retain(|x| *x > 20);
println!("{:?}", v);      // [22,33,44]
```

drain()：删除指定范围的元素，同时返回该范围所有元素的迭代器。如果删除迭代器，则丢弃迭代器中剩余的元素  
```rust
let mut v = vec![11, 22, 33, 44, 55];
let mut vv = v.clone();

// 删除中间3个元素，同时获取到这些元素的迭代器
// 直接丢弃迭代器，所以迭代器中的元素也直接被丢弃
// 这相当于直接删除指定范围的元素
v.drain(1..=3);
println!("{:?}", v);  // [11, 55]

// 将迭代器中的元素转换为Vec<i32>
let a: Vec<i32> = vv.drain(1..=3).collect();
println!("{:?}", a);  // [22, 33, 44]
println!("{:?}", vv); // [11, 55]
```

split_off()：  
```rust
let mut v = vec![11, 22, 33, 44, 55];
let vv = v.split_off(2);
println!("{:?}", v);   // [11, 22]
println!("{:?}", vv);  // [33, 44, 55]
```

