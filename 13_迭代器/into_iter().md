# into_iter()

`into_iter()` 是 Rust 迭代器模式的核心方法，它会**消费**一个值并将其转换为迭代器。名称中的 "into" 暗示了所有权转移。

## 1. 核心概念

### 所有权转移

`into_iter()` 接收 `self`（不是 `&self` 或 `&mut self`），会获得所有权：

```rust
let vec = vec![1, 2, 3];
let iter = vec.into_iter(); // vec 的所有权被转移

// println!("{:?}", vec); // 错误！vec 已被移动
```

### IntoIterator Trait

```rust
pub trait IntoIterator {
    type Item;                                  // 迭代器产生的元素类型
    type IntoIter: Iterator<Item = Self::Item>; // 返回的迭代器类型

    fn into_iter(self) -> Self::IntoIter;
}
```

## 2. 三种迭代方式对比

| 方法 | 签名 | 产生类型 | 原集合 |
| --- | --- | --- | --- |
| `into_iter()` | `self` | `T`（值） | 被消费 |
| `iter()` | `&self` | `&T`（引用） | 可用 |
| `iter_mut()` | `&mut self` | `&mut T`（可变引用） | 可用 |

```rust
let mut vec = vec![1, 2, 3];

// 按值迭代 - 消费集合
for x in vec.into_iter() { /* x: i32 */ }

// 按引用迭代 - 保留集合
let vec2 = vec![1, 2, 3];
for x in vec2.iter() { /* x: &i32 */ }

// 按可变引用迭代 - 可修改元素
let mut vec3 = vec![1, 2, 3];
for x in vec3.iter_mut() { *x *= 2; }
```

## 3. for 循环的魔法

`for` 循环会自动调用 `into_iter()`：

```rust
let items = vec!["a", "b", "c"];

// 这个 for 循环
for item in items {
    println!("{}", item);
}

// 等价于
let mut iter = IntoIterator::into_iter(items);
while let Some(item) = iter.next() {
    println!("{}", item);
}
```

## 4. 引用类型的行为

对引用调用 `into_iter()` 不会消费原数据：

```rust
let data = vec![1, 2, 3];

// 对 &Vec<T> 调用 into_iter() 等价于 iter()
for x in &data {      // x: &i32，data 仍可用
    println!("{}", x);
}

// 对 &mut Vec<T> 调用 into_iter() 等价于 iter_mut()
let mut data2 = vec![1, 2, 3];
for x in &mut data2 { // x: &mut i32
    *x *= 2;
}
```

## 5. 性能优势

```rust
let vec = vec![1, 2, 3, 4, 5];

// iter() 需要解引用
let doubled: Vec<_> = vec.iter()
    .map(|&x| x * 2)
    .collect();

// into_iter() 直接移动元素，更高效
let doubled: Vec<_> = vec.into_iter()
    .map(|x| x * 2)
    .collect();
```

## 6. 自定义类型实现

```rust
struct Countdown(u32);

impl IntoIterator for Countdown {
    type Item = u32;
    type IntoIter = CountdownIter;

    fn into_iter(self) -> Self::IntoIter {
        CountdownIter(self.0)
    }
}

struct CountdownIter(u32);

impl Iterator for CountdownIter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.0 > 0 {
            let current = self.0;
            self.0 -= 1;
            Some(current)
        } else {
            None
        }
    }
}

// 使用
for num in Countdown(3) {
    println!("{}...", num); // 3... 2... 1...
}
```

## 7. 常见陷阱

### for 循环中的隐式移动

```rust
let matrix = vec![vec![1, 2], vec![3, 4]];

for row in matrix {  // 错误！自动调用 into_iter()，matrix 被移动
    println!("{:?}", row);
}
// println!("{:?}", matrix); // 编译错误

// 正确做法：使用引用
for row in &matrix {
    println!("{:?}", row);
}
println!("{:?}", matrix); // matrix 仍可用
```

### 链式操作后无法访问原集合

```rust
let items = vec!["a".to_string(), "b".to_string()];

let first = items.into_iter().next(); // items 被消费

// println!("{:?}", items); // 错误！

// 如需保留原集合，用 iter()
let items2 = vec!["a".to_string(), "b".to_string()];
let first = items2.iter().next(); // items2 仍可用
```

## 8. 最佳实践

| 场景 | 推荐方法 | 原因 |
| --- | --- | --- |
| 不再需要原集合 | `into_iter()` | 避免克隆，性能最优 |
| 只读访问 | `iter()` | 保留原集合 |
| 修改元素 | `iter_mut()` | 原地修改 |
| 链式转换 | `into_iter()` | 直接移动元素 |

## 总结

1. `into_iter()` 消费集合，产生值的所有权
2. `for x in collection` 会隐式调用 `into_iter()`
3. 对引用调用 `into_iter()` 不会消费原数据
4. 需要保留集合时用 `&collection` 或 `iter()`
