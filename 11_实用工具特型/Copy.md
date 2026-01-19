# Copy

`Copy` trait 让简单类型在赋值或传参时自动复制，不转移所有权。

它特殊在哪？不需要你写任何代码，就是个「标记」，告诉编译器「这玩意儿可以按位复制」。

## 核心概念

默认情况下，Rust 的赋值会转移所有权：

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 的所有权转移给 s2
// println!("{}", s1);  // 错误！s1 已失效
```

但对于简单类型，这样太麻烦了：

```rust
let x = 5;
let y = x;  // x 自动复制，还能用
println!("{}", x);  // 没问题
```

这就是 `Copy`：赋值时自动复制，原来的值还在。

### Copy vs Clone

| 特性 | Copy | Clone |
|------|------|-------|
| 调用方式 | 自动（隐式） | 手动调用 `.clone()` |
| 成本 | 廉价（按位复制） | 可能昂贵（深拷贝） |
| 适用类型 | 简单类型（栈数据） | 任何类型 |
| trait 类型 | 标记 trait（无方法） | 普通 trait（有 `clone` 方法） |
| 实现方式 | `#[derive(Copy, Clone)]` | `#[derive(Clone)]` 或手动实现 |

### Copy trait 定义

```rust
pub trait Copy: Clone {
    // 空的！
}
```

两个关键点：

- **Copy 继承自 Clone**：想实现 `Copy` 必须先实现 `Clone`
- **没有方法**：编译器看到这个标记就知道可以按位复制了

## 底层原理

### 按位复制是啥？

就是把内存里的字节原样复制到新位置，像复印机：

```rust
let x: i32 = 42;  // 内存：00 00 00 2A（4 字节）
let y = x;        // 把这 4 个字节复制到新位置
```

快得很，一条 CPU 指令搞定。所以 Rust 允许它隐式发生。

### String 为啥不能 Copy？

`String` 在栈上存指针、长度、容量，真正的数据在堆上：

```
栈：[ptr | len: 5 | cap: 5]
      ↓
堆：  [h][e][l][l][o]
```

如果按位复制：

```rust
let s1 = String::from("hello");
let s2 = s1;  // 假设能 Copy
// 现在 s1 和 s2 的指针都指向同一块堆内存
// 离开作用域时，s1 释放内存，s2 也释放
// 💥 double free！
```

所以 Copy 有个铁律：**按位复制后，原值和新值必须能独立存在**。

## 哪些类型能 Copy？

### 能 Copy 的

| 类型 | 示例 |
|------|------|
| 所有整数类型 | `i8`, `i32`, `u64`, `isize` 等 |
| 所有浮点类型 | `f32`, `f64` |
| 布尔类型 | `bool` |
| 字符类型 | `char` |
| 不可变引用 | `&T`（注意：`&mut T` 不能 Copy） |
| 元组（所有元素都 Copy） | `(i32, bool)` |
| 数组（元素类型 Copy） | `[i32; 10]` |
| 函数指针 | `fn(i32) -> i32` |

### 不能 Copy 的

| 类型 | 原因 |
|------|------|
| `String` | 拥有堆内存 |
| `Vec<T>` | 拥有堆内存 |
| `Box<T>` | 拥有堆内存 |
| `Rc<T>`, `Arc<T>` | 需要管理引用计数 |
| `&mut T` | 可变引用必须唯一 |
| 实现了 `Drop` 的类型 | 需要自定义清理逻辑 |

### 自定义类型咋办？

所有字段都是 Copy，就能派生：

```rust
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // 自动复制
println!("{}, {}", p1.x, p2.x);  // 都能用
```

记得同时派生 `Clone`，因为 `Copy` 继承自它。

## 使用场景

### 小型数据结构

```rust
// 坐标点：8 字节，适合 Copy
#[derive(Copy, Clone)]
struct Point {
    x: f32,
    y: f32,
}

fn distance(p1: Point, p2: Point) -> f32 {
    // 按值传递，自动复制
    let dx = p1.x - p2.x;
    let dy = p1.y - p2.y;
    (dx * dx + dy * dy).sqrt()
}

fn main() {
    let origin = Point { x: 0.0, y: 0.0 };
    let point = Point { x: 3.0, y: 4.0 };
    
    let d = distance(origin, point);
    // origin 和 point 仍然可用！
    println!("距离: {}", d);
}
```

### 配置参数

```rust
#[derive(Copy, Clone)]
struct Config {
    width: u32,
    height: u32,
    fullscreen: bool,
}

fn apply_config(config: Config) {
    // 按值传递，不需要引用
    println!("设置分辨率: {}x{}", config.width, config.height);
}

fn main() {
    let config = Config {
        width: 1920,
        height: 1080,
        fullscreen: true,
    };
    
    apply_config(config);  // 自动复制
    apply_config(config);  // 可以多次使用
}
```

### 数学运算

```rust
#[derive(Copy, Clone)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Color {
    fn blend(self, other: Color) -> Color {
        // 按值接收，自动复制
        Color {
            r: (self.r as u16 + other.r as u16) as u8 / 2,
            g: (self.g as u16 + other.g as u16) as u8 / 2,
            b: (self.b as u16 + other.b as u16) as u8 / 2,
        }
    }
}

fn main() {
    let red = Color { r: 255, g: 0, b: 0 };
    let blue = Color { r: 0, g: 0, b: 255 };
    
    let purple = red.blend(blue);  // red 和 blue 仍然可用
}
```

## 常见坑

### 小类型不等于能 Copy

```rust
// 错！就算只有一个字段，String 不是 Copy
struct Name {
    value: String,
}

// 编译错误
// #[derive(Copy, Clone)]
// struct Name { value: String }
```

一个字段不是 Copy，整个结构体就不行。

### Copy 和 Drop 不能共存

```rust
#[derive(Copy, Clone)]
struct Resource {
    id: i32,
}

// 错！Copy 类型不能实现 Drop
impl Drop for Resource {
    fn drop(&mut self) {
        println!("释放资源 {}", self.id);
    }
}
```

为啥？Copy 类型会被隐式复制多次，有 Drop 的话同一个资源会被释放多次。

### 大结构体别 Copy

```rust
// 别这么干！1024 字节
#[derive(Copy, Clone)]
struct BigData {
    buffer: [u8; 1024],
}

fn process(data: BigData) {
    // 每次调用复制 1024 字节
}
```

超过 16-32 字节的，用引用。

### 可变引用不能 Copy

```rust
let mut x = 5;
let r1 = &mut x;
let r2 = r1;  // 所有权转移

// println!("{}", r1);  // 错！r1 失效了
```

可变引用必须唯一，能 Copy 就会有多个可变引用指向同一个值，违反借用规则。

## Copy vs Clone 怎么选？

| 情况 | 用啥 | 示例 |
|------|------|------|
| 简单数值 | `Copy` | `i32`, `f64`, `bool` |
| 小结构体（≤16 字节） | `Copy` | `Point`, `Color` |
| 有堆数据 | 只 `Clone` | `String`, `Vec<T>` |
| 大结构体（>16 字节） | 只 `Clone` 或引用 | 配置对象 |
| 需要自定义清理 | 只 `Clone` | 实现了 `Drop` 的 |

## 最佳实践

### 能 Copy 就 Copy

满足条件就实现，代码简洁：

```rust
// 有 Copy：简洁
fn add(a: Point, b: Point) -> Point { ... }

// 没 Copy：得用引用或 clone
fn add(a: &Point, b: &Point) -> Point { ... }
```

### 保持小巧

想 Copy 就别超过 16 字节：

```rust
// 好：8 字节
#[derive(Copy, Clone)]
struct Point { x: f32, y: f32 }

// 别：1024 字节
#[derive(Copy, Clone)]
struct BigBuffer { data: [u8; 1024] }
```

### 写文档

实现了 Copy 就说一声：

```rust
/// 2D 坐标点（Copy，复制很便宜）
#[derive(Copy, Clone)]
struct Point {
    x: f32,
    y: f32,
}
```

### 别滥用

类型代表「资源」或「唯一实体」，别 Copy：

```rust
// 别 Copy：代表唯一的文件句柄
struct File {
    fd: i32,
}

// 可以 Copy：就是个数值
#[derive(Copy, Clone)]
struct FileDescriptor(i32);
```

## 总结

- Copy 是隐式的按位复制，赋值时自动复制
- 只有简单类型能 Copy：不能有堆数据、不能实现 Drop
- Copy 必须配合 Clone，因为它继承自 Clone
- 适合小型栈数据，超过 16 字节考虑用引用

Copy 让简单类型用起来像其他语言的基本类型一样自然，但 Rust 的规则保证了这种便利是安全的。
