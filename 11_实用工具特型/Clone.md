# Clone

简单说，`Clone` trait 让你能显式地复制一个值。和 `Copy` 不同，克隆可能很昂贵（比如复制整个 `Vec`），所以 Rust 要求你明确调用 `.clone()` 方法。

## 1. 核心概念

`Clone` 和 `Copy` 都是用来复制值的，但有本质区别：

### Copy：隐式、廉价的按位复制

```rust
let x = 5;
let y = x;  // 自动复制，x 仍然可用
println!("{}", x);  // OK
```

- **特点**：编译器自动复制，不需要你操心
- **限制**：只能用于简单类型（整数、浮点数、布尔值等）
- **成本**：按位复制，非常快

> **什么是按位复制？**
> 就是把内存中的字节原样复制到新位置，像复印机一样。比如 `i32` 占 4 个字节，复制时就把这 4 个字节的二进制数据一模一样地拷贝一份。因为是简单的内存拷贝（一条 CPU 指令），所以非常快。

### Clone：显式、可能昂贵的深拷贝

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // 必须显式调用
println!("{}", s1);   // OK，s1 仍然有效
```

- **特点**：必须显式调用 `.clone()`
- **适用**：任何类型都可以实现
- **成本**：可能涉及堆分配，比较慢

## 2. 底层原理

### Clone trait 定义

```rust
pub trait Clone {
    fn clone(&self) -> Self;
    
    // 可选方法，用于批量克隆
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone();
    }
}
```

### 为什么需要显式调用？

Rust 的设计哲学：**让昂贵的操作显而易见**。

```rust
// 这段代码一眼就能看出性能开销
let big_vec = vec![1; 1_000_000];
let copy1 = big_vec.clone();  // 复制 100 万个元素！
let copy2 = big_vec.clone();  // 又复制 100 万个！
```

如果像 `Copy` 那样隐式复制，你可能不小心就写出性能杀手。

### 自动派生 Clone

大多数情况下，你不需要手写实现：

```rust
#[derive(Clone)]
struct Point {
    x: i32,
    y: i32,
}

// 编译器自动生成：
// impl Clone for Point {
//     fn clone(&self) -> Self {
//         Point {
//             x: self.x.clone(),
//             y: self.y.clone(),
//         }
//     }
// }
```

**派生条件**：所有字段都必须实现 `Clone`。

## 3. 使用场景

| 场景 | 推荐 | 原因 |
|------|------|------|
| 需要保留原值 | `Clone` | 避免所有权转移 |
| 构建新的数据结构 | `Clone` | 基于现有数据创建副本 |
| 多线程共享数据 | `Clone` + `Arc` | 每个线程持有独立副本 |
| 简单类型（i32, bool） | `Copy` | 自动复制更方便 |

### 实际例子

```rust
// 场景：你有一个购物车，想基于它创建一个新订单
fn main() {
    let cart = vec!["苹果", "香蕉", "橙子"];
    
    // 错误做法：直接传递所有权
    // let order = cart;  // cart 被移动，后面不能再用了！
    
    // 正确做法：克隆一份
    let order = cart.clone();  // 复制一份给订单
    
    println!("购物车: {:?}", cart);   // 购物车还在
    println!("订单: {:?}", order);    // 订单也有了
    
    // 两个独立的 Vec，互不影响
}
```

```rust
// 场景：修改配置但保留原始值
fn update_config(config: &Vec<String>) -> Vec<String> {
    let mut new_config = config.clone();  // 克隆原配置
    new_config.push("new_setting".to_string());  // 添加新设置
    new_config  // 返回修改后的版本
}

fn main() {
    let original = vec!["setting1".to_string(), "setting2".to_string()];
    let updated = update_config(&original);
    
    println!("原配置: {:?}", original);  // ["setting1", "setting2"]
    println!("新配置: {:?}", updated);   // ["setting1", "setting2", "new_setting"]
}
```

## 4. 常见陷阱

### 陷阱 1：忘记 Clone 的成本

```rust
// 错误！每次循环都克隆整个 Vec
let data = vec![1, 2, 3, 4, 5];
for _ in 0..1000 {
    let copy = data.clone();  // 性能杀手！
    // 使用 copy...
}
```

**正确做法**：

```rust
// 如果只是读取，用引用
let data = vec![1, 2, 3, 4, 5];
for _ in 0..1000 {
    let reference = &data;  // 零成本
    // 使用 reference...
}
```

### 陷阱 2：Clone 不是深拷贝的万能药

```rust
use std::rc::Rc;

let data = Rc::new(vec![1, 2, 3]);
let copy = data.clone();  // 只克隆了 Rc，不是 Vec！

// data 和 copy 指向同一个 Vec
println!("{}", Rc::strong_count(&data));  // 输出 2
```

**理解**：`Rc::clone()` 只增加引用计数，不复制内部数据。

### 陷阱 3：手动实现 Clone 时忘记深拷贝

```rust
struct Container {
    data: Vec<i32>,
}

// 错误实现！
impl Clone for Container {
    fn clone(&self) -> Self {
        Container {
            data: self.data,  // 错误！Vec 没有 Copy
        }
    }
}
```

**正确做法**：

```rust
impl Clone for Container {
    fn clone(&self) -> Self {
        Container {
            data: self.data.clone(),  // 正确：克隆 Vec
        }
    }
}
```

## 5. 最佳实践

1. **优先用 `#[derive(Clone)]`**：除非有特殊需求，让编译器生成
2. **Clone 前三思**：问自己「真的需要复制吗？引用行不行？」
3. **文档标注成本**：如果 `clone()` 很昂贵，在注释里说明
4. **考虑 `Rc`/`Arc`**：多个所有者共享数据时，比克隆更高效

### Clone vs 引用的选择

```rust
// 场景 1：需要修改副本 → 用 Clone
fn modify_copy(data: &Vec<i32>) -> Vec<i32> {
    let mut copy = data.clone();
    copy.push(42);
    copy
}

// 场景 2：只读访问 → 用引用
fn read_only(data: &Vec<i32>) -> i32 {
    data.iter().sum()
}

// 场景 3：多个所有者 → 用 Rc/Arc
use std::rc::Rc;
fn share_ownership() -> (Rc<Vec<i32>>, Rc<Vec<i32>>) {
    let data = Rc::new(vec![1, 2, 3]);
    (data.clone(), data.clone())  // 只增加引用计数
}
```

## 总结

1. **Clone 是显式的深拷贝**：必须调用 `.clone()`，让昂贵操作可见
2. **和 Copy 的区别**：Copy 隐式且廉价，Clone 显式且可能昂贵
3. **使用原则**：能用引用就别克隆，能用 `Rc`/`Arc` 就别克隆多份
