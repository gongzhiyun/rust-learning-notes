# `into_iter()`

`into_iter()`是Rust迭代器模式的核心方法，它的名称暗示了关键行为：消费（consume）一个值，并将其转换为迭代器。

## 核心概念

### 1. 所有权转移（`Move`）

`into_inter()`接收`self`（不是`&self` 或` &mut self`），这意味着它会获得所有权：

```rust
let vec = vec![1, 2, 3];
let iter = vec.into_iter(); // vec 的所有权被转移到into_iter()

// println!("{:?}", vec); // 错误！ vec 已经被移动
```



### 2. `IntoIterator Trait`

`into_iter()`是`IntoInterator trait`的方法：

```rust
pub trait IntoInterator {
  type Item;      // 迭代器产生的元素类型
  type IntoIter: Iterator<Item = Self::Item> // 返回的迭代器类型
  
  fn into_iter(self) -> Self::IntoIter;
}
```



## 三种迭代器模式对比

```rust
let mut vec = vec![1, 2, 3];

// 1. into_iter() -> 按值迭代（所有权）
let iter1 = vec.into_iter();
// vec 被消耗， 后续不能再用
// 产生：i32 （值）

// 2. iter() - 按不可变引用迭代
let vec2 = vec![1, 2, 3];
let iter2 = vec2.iter();
// vec2 仍然可用
// 产生： &i32 (引用)

// 3. iter_mut() - 按可变引用迭代
let mut vec3 = vec![1, 2, 3];
let iter3 = vec3.iter_mut();
// vec3 仍然可用
// 产生： &mut i32 （可变引用）
```



## 不同容器的行为

### 1. 集合类型（`Vec`, `HashMap`等）

```rust
// Vec<T> 消耗整个向量
let v = vec![1, 2, 3];
let mut iter = v.into_iter();

assert_eq!(iter.next(), Some(1)); // 产生值
assert_eq!(iter.next(), Some(2));
assert_eq!(iter.next(), Some(3));
assert_eq!(iter.next(), None);

// 迭代后，原始向量已被消耗
```



### 2. 引用类型

```rust
// &[T] 或 &Vec<T> 不会消耗数据
let data = vec![1, 2, 3];
let slice_ref = &data;

// 对引用的 into_iter() 实际上调用 iter()
let iter = slice_ref.into_iter();  // 等价于 slice_ref.iter()
// 产生 &i32，data 仍然可用
```



## 在`for`循环中的魔法

```rust
let items = vec!["a", "b", "c"];

// 这个 for 循环
for item in items {
    println!("{}", item);
}

// 会被 Rust 转换为：
let mut iter = IntoIterator::into_iter(items);
while let Some(item) = iter.next() {
    println!("{}", item);
}
```



## 自定义类型的实现

```rust
struct Countdown {
    count: u32,
}

// 实现 IntoIterator
impl IntoIterator for Countdown {
    type Item = u32;
    type IntoIter = CountdownIterator;
    
    fn into_iter(self) -> Self::IntoIter {
        CountdownIterator { count: self.count }
    }
}

// 实现迭代器
struct CountdownIterator {
    count: u32,
}

impl Iterator for CountdownIterator {
    type Item = u32;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.count > 0 {
            let current = self.count;
            self.count -= 1;
            Some(current)
        } else {
            None
        }
    }
}

// 使用
let countdown = Countdown { count: 3 };
for num in countdown {  // 调用 into_iter()
    println!("{}...", num);
}
// 输出: 3... 2... 1...
```

## 与 `iter()`的自动转换

```rust
fn process_items<T>(items: T)
where
	T: IntoInterator.
	T::Item: std::fmt::Display;
{
  for item in items {
    println!("{}", item);
  }
}

// 可以接受多种类型
let vec = vec![1, 2, 3];
process_items(vec);    // 实用 into_iter()

let array = [4, 5, 6];
process_items(&array); // 自动调用iter() 因为 &[T] 实现了IntoInterator

let slice = &[7, 8, 9];
process_items(slice);   // 同样工作
```



## 性能优势

```rust
// 场景：转换向量并收集结果
let vec = vec![1, 2, 3, 4, 5];

// 方法1: 使用iter() - 需要克隆
let doubled: Vec<_> = vec.iter()
	  .map(|&x| x * 2) // 需要解引用
    .collect();

// 方法2: 使用into_iter() - 移动元素，无需克隆
let doubled: Vec<_> = vec.into_iter()
	  .map(|x| x * 2)    // 直接使用
	  .collect();
```



## 实用模式

### 1. 解构元组向量

```rust
let pairs = vec![("a", 1), ("b", 2), ("c", 3)];

// 使用into_iter()获取所有权
for (key, value) in pairs.into_iter() {
  println!("{}:{}", key, value);
}
```



### 2. 转换元素类型

```rust
let strings = vec!["1".to_string, "2".to_string(), "3".to_string()];

// 转换为整数向量
let numbers: Vec[i32] = strings.into_inter()
    .map(|s| s.parse::<i32>().unwrap())
    .collect();
```



## 常见陷阱

### 1. 陷阱1： 意外移动

```rust
let items = vec!["a".to_string(), "b".to_string()];

// 错误：使用 into_iter() 后 items 被移动
let first_char = items.into_iter()
    .next()
    .and_then(|s| s.chars().next());

// println!("{:?}", items);  // 编译错误：值已被移动
```

### 2. 陷阱2: for 循环中的隐藏移动

```rust
let matrix = vec![vec![1, 2], vec![3, 4]];

// 每次迭代都会移动行！
for row in matrix {  // 自动调用 into_iter()
    println!("{:?}", row);
}

// println!("{:?}", matrix);  // 错误！matrix 已被移动

// 应该使用：
for row in &matrix {  // 使用 iter() 而不是 into_iter()
    println!("{:?}", row);
}
```



## 与相关trait的关系

```rust
// 完整的迭代器生态系统
trait IntoIterator {  // 转换为迭代器
    fn into_iter(self) -> Self::IntoIter;
}

trait Iterator {  // 迭代功能
    fn next(&mut self) -> Option<Self::Item>;
}

trait FromIterator<A> {  // 从迭代器构建
    fn from_iter<T>(iter: T) -> Self
    where
        T: IntoIterator<Item = A>;
}
```



## 最佳实践

1. **需要所有权时用 `into_iter()`**：当你不再需要原始集合时
2. **只读访问用 `iter()`**：当只需要读取数据时
3. **修改元素用 `iter_mut()`**：当需要修改集合中的元素时
4. **链式操作优先用 `into_iter()`**：通常更高效
5. **注意生命周期**：`into_iter()`会结束值的生命周期