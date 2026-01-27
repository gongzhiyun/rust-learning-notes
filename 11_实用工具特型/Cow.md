# Rust 实用工具类型：Cow

**写时克隆，性能优化的秘密武器**

简单说，`Cow`（Clone on Write）是一个智能指针枚举，让你延迟决定是借用还是拥有。不需要修改时零成本借用，需要修改时才分配内存克隆数据。这是一种「能借就借，必要时才买」的性能优化策略。

```rust
use std::borrow::Cow;

// 场景：处理用户输入，可能需要转小写
fn normalize(input: &str) -> Cow<str> {
    if input.chars().all(|c| c.is_lowercase()) {
        Cow::Borrowed(input)  // 已经是小写，直接借用（零成本）
    } else {
        Cow::Owned(input.to_lowercase())  // 需要转换，分配新字符串
    }
}

let s1 = "hello";  // 已经是小写
let result1 = normalize(s1);  // Borrowed，不分配内存 ✅

let s2 = "Hello";  // 有大写字母
let result2 = normalize(s2);  // Owned，分配内存 ⚡
```

## 1. 核心概念：为什么需要 Cow？

### 问题：总是克隆 vs 总是借用

你经常会遇到这种两难：

```rust
// 方案 1：总是克隆（浪费）
fn process_v1(input: &str) -> String {
    input.to_lowercase()  // 即使已经是小写也分配内存
}

// 方案 2：总是借用（不灵活）
fn process_v2(input: &str) -> &str {
    input  // 无法返回修改后的字符串
}
```

**困境**：

- 方案 1：`process_v1("hello")` 浪费了一次内存分配（已经是小写了）
- 方案 2：`process_v2("Hello")` 无法处理需要转换的情况

### 解决方案：Cow

Cow 让你两全其美：

```rust
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.chars().all(|c| c.is_lowercase()) {
        Cow::Borrowed(input)  // 不需要修改，零成本借用
    } else {
        Cow::Owned(input.to_lowercase())  // 需要修改，才分配内存
    }
}

// 性能对比
process("hello");  // Borrowed，零成本 ✅
process("Hello");  // Owned，按需分配 ✅
```

**核心价值**：

- 不需要修改时：零成本借用，不分配内存
- 需要修改时：按需分配，灵活处理

### Cow 的两个变体

```rust
pub enum Cow<'a, B: ?Sized + ToOwned> {
    Borrowed(&'a B),  // 借用：零成本，指向原始数据
    Owned(<B as ToOwned>::Owned),  // 拥有：分配了内存的副本
}
```

| 变体 | 成本 | 何时使用 |
| --- | --- | --- |
| `Borrowed` | 零成本（只是引用） | 数据不需要修改 |
| `Owned` | 有成本（堆分配 + 复制） | 数据需要修改 |

```rust
use std::borrow::Cow;

// Borrowed：零成本
let cow1: Cow<str> = Cow::Borrowed("hello");  // 只是一个引用

// Owned：有成本
let cow2: Cow<str> = Cow::Owned(String::from("world"));  // 分配了堆内存
```

## 2. 写时克隆：to_mut() 的魔法

`to_mut()` 是 Cow 的核心方法，实现了「写时克隆」：

```rust
use std::borrow::Cow;

let mut cow: Cow<str> = Cow::Borrowed("hello");

// 第一次修改：触发克隆
cow.to_mut().push_str(" world");  // Borrowed → Owned（分配内存）
println!("{}", cow);  // "hello world"

// 后续修改：不再克隆
cow.to_mut().push_str("!");  // 已经是 Owned，直接修改
println!("{}", cow);  // "hello world!"
```

**执行流程**：

1. `cow` 初始是 `Borrowed("hello")`（零成本）
2. 第一次调用 `to_mut()`：检测到是 `Borrowed`，克隆成 `Owned(String::from("hello"))`
3. 返回 `String` 的可变引用，执行 `push_str(" world")`
4. 第二次调用 `to_mut()`：已经是 `Owned`，直接返回可变引用，不再克隆

**关键点**：只在第一次修改时克隆，后续修改复用已分配的内存。

## 3. 常见使用场景

### 场景 1：条件性字符串修改

```rust
use std::borrow::Cow;

// 处理 URL：如果没有协议前缀就添加
fn ensure_protocol(url: &str) -> Cow<str> {
    if url.starts_with("http://") || url.starts_with("https://") {
        Cow::Borrowed(url)  // 已经有协议，零成本借用
    } else {
        Cow::Owned(format!("https://{}", url))  // 添加协议，分配内存
    }
}

let url1 = "https://example.com";
let result1 = ensure_protocol(url1);  // Borrowed，不分配 ✅

let url2 = "example.com";
let result2 = ensure_protocol(url2);  // Owned，分配内存 ⚡
```

### 场景 2：批量数据过滤

```rust
use std::borrow::Cow;

// 过滤负数：如果没有负数就不分配新 Vec
fn filter_negatives(numbers: &[i32]) -> Cow<[i32]> {
    if numbers.iter().all(|&n| n >= 0) {
        Cow::Borrowed(numbers)  // 没有负数，零成本
    } else {
        let filtered: Vec<i32> = numbers.iter()
            .filter(|&&n| n >= 0)
            .copied()
            .collect();
        Cow::Owned(filtered)  // 有负数，分配新 Vec
    }
}

let data1 = [1, 2, 3];
let result1 = filter_negatives(&data1);  // Borrowed ✅

let data2 = [1, -2, 3];
let result2 = filter_negatives(&data2);  // Owned ⚡
```

### 场景 3：配置文件路径处理

```rust
use std::borrow::Cow;
use std::path::Path;

// 处理配置路径：如果是绝对路径就直接用，相对路径就拼接
fn resolve_path<'a>(path: &'a Path, base: &Path) -> Cow<'a, Path> {
    if path.is_absolute() {
        Cow::Borrowed(path)  // 绝对路径，直接用
    } else {
        Cow::Owned(base.join(path))  // 相对路径，拼接
    }
}
```

## 4. 常见陷阱

### 陷阱 1：过度使用 into_owned()

```rust
use std::borrow::Cow;

// 错误！立即转成 Owned，失去了 Cow 的优势
fn bad_usage(input: &str) -> String {
    let cow = normalize(input);
    cow.into_owned()  // 即使是 Borrowed 也会克隆
}

// 正确：保持 Cow 类型，让调用者决定
fn good_usage(input: &str) -> Cow<str> {
    normalize(input)  // 返回 Cow，调用者按需使用
}
```

**理解**：`into_owned()` 会强制克隆 `Borrowed` 变体，失去了零成本的优势。

### 陷阱 2：不必要的 to_mut() 调用

```rust
use std::borrow::Cow;

let mut cow: Cow<str> = Cow::Borrowed("hello");

// 错误！只是读取，不需要 to_mut()
// let len = cow.to_mut().len();  // 触发不必要的克隆

// 正确：直接读取
let len = cow.len();  // Cow 实现了 Deref，可以直接调用 str 的方法
```

**理解**：`to_mut()` 会触发克隆，只在需要修改时才调用。

### 陷阱 3：误解 Cow 的适用场景

```rust
// 不适合：总是需要修改
fn always_modify(input: &str) -> Cow<str> {
    Cow::Owned(input.to_uppercase())  // 总是 Owned，不如直接返回 String
}

// 适合：有时需要修改，有时不需要
fn sometimes_modify(input: &str) -> Cow<str> {
    if input.is_empty() {
        Cow::Borrowed("default")  // 不修改，零成本
    } else {
        Cow::Owned(input.to_uppercase())  // 修改，分配内存
    }
}
```

**理解**：Cow 适合「有时借用，有时拥有」的场景。如果总是需要拥有，直接用 `String` 或 `Vec` 更清晰。

### 陷阱 4：生命周期陷阱

```rust
use std::borrow::Cow;

// 错误！返回的 Cow 引用了局部变量
// fn bad_lifetime() -> Cow<'static, str> {
//     let s = String::from("hello");
//     Cow::Borrowed(&s)  // ❌ s 在函数结束时被销毁
// }

// 正确：返回 Owned 变体
fn good_lifetime() -> Cow<'static, str> {
    Cow::Owned(String::from("hello"))  // ✅ 拥有数据
}

// 或者用字符串字面量
fn literal_lifetime() -> Cow<'static, str> {
    Cow::Borrowed("hello")  // ✅ 字符串字面量是 'static
}
```

## 5. 底层原理（进阶）

### Cow 的完整定义

```rust
pub enum Cow<'a, B: ?Sized + 'a>
where
    B: ToOwned,
{
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

### 泛型参数解析

#### 'a：生命周期参数

`'a` 标注 `Borrowed` 变体中引用的生命周期。

```rust
let s = String::from("hello");
let cow: Cow<str> = Cow::Borrowed(&s);
// 编译器推断：Cow<'a, str>，其中 'a 是 &s 的生命周期
// cow 不能比 s 活得更久
```

#### B: ToOwned

B 必须实现 `ToOwned`，这样才能在需要时从借用创建拥有类型。

```rust
// str 实现了 ToOwned
impl ToOwned for str {
    type Owned = String;
    fn to_owned(&self) -> String {
        String::from(self)
    }
}

// 所以 Cow<str> 可以在需要时调用 to_owned() 创建 String
let mut cow: Cow<str> = Cow::Borrowed("hello");
cow.to_mut();  // 内部调用 "hello".to_owned() → String
```

#### B: ?Sized

`?Sized` 表示「B 可以是动态大小类型（DST）」，比如 `str`、`[T]`。

```rust
// 如果没有 ?Sized
// Cow<String> ✅  String 大小固定
// Cow<str> ❌     str 大小不固定，编译错误

// 有了 ?Sized
Cow<str>    // ✅ 最常用的形式
Cow<[i32]>  // ✅ 切片的 Cow
```

> **什么是动态大小类型？**
> 编译期不知道大小的类型，如 `str`（长度运行时才知道）、`[T]`（长度运行时才知道）。虽然 `str` 本身大小不固定，但 `&str` 大小固定（指针 + 长度），所以 `Borrowed(&'a str)` 可以存储。

#### B: 'a

`B: 'a` 表示「B 中的所有引用都必须至少活到 'a」。对于 `str`、`[i32]` 这种不包含引用的类型，这个约束自动满足。

### 核心方法实现

```rust
impl<'a, B: ?Sized + ToOwned> Cow<'a, B> {
    // 获取可变引用（写时克隆的关键）
    pub fn to_mut(&mut self) -> &mut <B as ToOwned>::Owned {
        match *self {
            Cow::Borrowed(b) => {
                *self = Cow::Owned(b.to_owned());  // 克隆！
                match *self {
                    Cow::Owned(ref mut o) => o,
                    _ => unreachable!(),
                }
            }
            Cow::Owned(ref mut o) => o,  // 直接返回
        }
    }
    
    // 转换为拥有类型
    pub fn into_owned(self) -> <B as ToOwned>::Owned {
        match self {
            Cow::Borrowed(b) => b.to_owned(),  // 克隆
            Cow::Owned(o) => o,  // 直接返回
        }
    }
}
```

## 6. 最佳实践

1. **返回 Cow 而不是立即 into_owned()**：让调用者决定是否需要拥有所有权
2. **只在需要修改时调用 to_mut()**：读取操作直接用 `Deref`，不触发克隆
3. **用 Cow 优化热路径**：在性能敏感的代码中，避免不必要的内存分配

## 总结

1. **Cow 解决「总是克隆」vs「总是借用」的两难**：不需要修改时零成本借用，需要修改时按需分配
2. **核心方法 to_mut()**：实现写时克隆，只在第一次修改时克隆，后续修改复用内存
3. **适用场景**：有时需要修改、有时不需要修改的数据处理，避免不必要的内存分配
