# Sized

`Sized` 是一个标记 trait，告诉编译器「这个类型占多少字节我知道」。

为什么这很重要？因为 Rust 需要在编译期确定每个变量在栈上占多少空间。`i32` 永远是 4 字节，`String` 永远是 24 字节（指针+长度+容量），编译器可以放心地分配内存。但 `str` 呢？`"hi"` 是 2 字节，`"hello world"` 是 11 字节——编译器没法提前知道，所以 `str` 不是 `Sized`。

好消息是：你几乎不需要手动处理 `Sized`。编译器会自动为所有固定大小的类型实现它，你要关心的只是少数例外情况。

## 1. 核心概念

### 固定大小 vs 动态大小

Rust 中的类型分两种：

**固定大小类型（Sized）**：编译期就能确定占多少字节。

```rust
let a: i32 = 42;           // 永远 4 字节
let b: (u8, u16) = (1, 2); // 永远 4 字节（1 + 1填充 + 2）
let c: [i32; 3] = [1,2,3]; // 永远 12 字节
```

**动态大小类型（DST）**：大小取决于运行时的值。

```rust
// str 的大小取决于字符串内容
let s1: &str = "hi";          // 指向 2 字节的数据
let s2: &str = "hello world"; // 指向 11 字节的数据

// [i32] 的大小取决于元素个数
let slice: &[i32] = &[1, 2, 3, 4, 5];  // 指向 20 字节的数据
```

注意：`str` 和 `[i32]` 本身是 DST，但 `&str` 和 `&[i32]` 是固定大小的（都是 16 字节的胖指针）。

### 为什么要区分？

编译器生成代码时，必须知道每个类型占多少字节——不管是在栈上分配局部变量，还是在堆上分配内存，都需要一个确定的大小：

```rust
fn example() {
    let x: i32;    // 编译器：分配 4 字节
    let s: String; // 编译器：分配 24 字节
    let ???: str;  // 编译器：分配多少？不知道！编译失败
}
```

这就是为什么你不能直接声明 `let s: str`，只能用 `&str` 或 `Box<str>`——引用和智能指针的大小是固定的，它们把「不确定大小」的问题转移到了运行时：通过指针间接访问数据，指针本身的大小是确定的。

### Rust 中的三种 DST

| DST 类型 | 例子 | 数据位置 | 说明 |
| --- | --- | --- | --- |
| 切片 | `[T]` | 栈或堆 | 可能指向栈上数组 `&arr[..]`，也可能指向堆上 `Vec` 的数据 |
| 字符串切片 | `str` | 栈或堆 | 字面量在静态存储区，`String` 的数据在堆上 |
| trait 对象 | `dyn Trait` | 栈或堆 | 取决于原始值的位置 |

DST 的「大小不确定」是编译期的问题，与数据存放在哪无关。编译器需要在编译时知道类型大小才能生成正确的代码，而 DST 的大小只有运行时才知道。

### Sized trait

```rust
// 标准库定义（简化）
#[lang = "sized"]
pub trait Sized {
    // 没有方法，纯标记 trait
}
```

`Sized` 是编译器的「契约」：实现了它的类型保证在编译期有确定的大小。

**三个关键点**：

1. **自动实现**：编译器为所有固定大小类型自动实现，你不需要写 `impl Sized for MyType`
2. **不能手动实现**：这是编译器的特权，你无法手动 `impl Sized`
3. **隐式约束**：泛型参数默认带 `T: Sized`，这是 Rust 中唯一一个默认添加的 trait 约束

## 2. 隐式 Sized 约束

这是 `Sized` 最重要也最容易忽略的特性：

```rust
// 这两个函数签名完全等价
fn foo<T>(t: T) { }
fn foo<T: Sized>(t: T) { }  // 编译器自动添加 Sized 约束
```

**为什么默认要求 Sized？**

因为函数参数需要放在栈上，编译器必须知道分配多少空间：

```rust
fn print_value<T>(value: T) {
    // 编译器需要知道 T 多大，才能在栈上为 value 分配空间
}
```

## 3. ?Sized：放宽约束

当你想让泛型参数接受 DST 时，使用 `?Sized`（读作 "maybe sized" 或 "optionally sized"）：

```rust
// 默认：T 必须是 Sized
fn sized_only<T>(t: &T) { }

// 放宽：T 可以是 DST
fn any_type<T: ?Sized>(t: &T) { }
```

### 不加 ?Sized 会怎样？

```rust
fn print_ref<T>(t: &T) {
    println!("{:p}", t);
}

fn main() {
    let s: &str = "hello";
    print_ref(s);  // 错误！str 不满足 Sized 约束
}
```

编译器报错：

```text
error[E0277]: the size for values of type `str` cannot be known at compilation time
```

因为 `T` 默认要求 `Sized`，而 `str` 是 DST。加上 `?Sized` 就能编译：

```rust
fn print_ref<T: ?Sized>(t: &T) {
    println!("{:p}", t);
}

fn main() {
    let s: &str = "hello";
    print_ref(s);  // 正确！
}
```

**关键理解**：`?Sized` 约束的是泛型参数 `T`，不是整个函数参数的类型。

```rust
fn example<T: ?Sized>(t: &T) { }
//         ^^^^^^^^^ T 可以是 str（DST）
//                   ^^ 但 &T 永远是固定大小（胖指针）
```

### 为什么 `Box<T>` 可以持有 DST？

`Box<T>` 本身的大小是固定的：

- `Box<i32>`：8 字节（一个指针）
- `Box<str>`：16 字节（指针 + 长度）
- `Box<dyn Trait>`：16 字节（指针 + vtable 指针）

```rust
use std::mem::size_of;

println!("{}", size_of::<Box<i32>>());       // 8
println!("{}", size_of::<Box<str>>());       // 16（胖指针）
println!("{}", size_of::<Box<dyn Send>>());  // 16（胖指针）
```

`Box` 的定义使用了 `?Sized`，所以它的泛型参数 `T` 可以是 DST：

```rust
// 标准库中 Box 的定义（简化）
pub struct Box<T: ?Sized>(/* 指针 */);
```

### 直接按值传递 DST 不行

```rust
fn bad<T: ?Sized>(t: T) { }      // 错误！T 可能是 DST，无法确定栈空间
fn good<T: ?Sized>(t: &T) { }    // 正确，&T 大小固定（8 或 16 字节）
fn also_good<T: ?Sized>(t: Box<T>) { }  // 正确，Box<T> 大小固定
```

### 实际应用

标准库中大量使用 `?Sized` 来提高灵活性：

```rust
// std::borrow::Borrow 的定义
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}

// 这样 String 可以实现 Borrow<str>
impl Borrow<str> for String {
    fn borrow(&self) -> &str { self }
}
```

```rust
// AsRef 也使用 ?Sized
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

// 让函数同时接受 &str 和 &String
fn print_str<S: AsRef<str>>(s: S) {
    println!("{}", s.as_ref());
}
```

## 4. 胖指针

指向非 Sized 类型的引用是**胖指针**（fat pointer），包含额外的元数据：

| 指针类型 | 大小 | 内容 |
| --- | --- | --- |
| `&i32` | 8 字节 | 数据指针 |
| `&[i32]` | 16 字节 | 数据指针 + 长度 |
| `&str` | 16 字节 | 数据指针 + 长度 |
| `&dyn Trait` | 16 字节 | 数据指针 + vtable 指针 |

```rust
use std::mem::size_of;

println!("{}", size_of::<&i32>());        // 8
println!("{}", size_of::<&[i32]>());      // 16
println!("{}", size_of::<&dyn Animal>()); // 16
```

## 5. 结构体中的非 Sized 字段

结构体可以包含一个非 Sized 字段，但必须是**最后一个字段**：

```rust
// 正确：非 Sized 字段在最后
struct MyStr {
    len: usize,
    data: str,  // 非 Sized，必须放最后
}

// 错误：非 Sized 字段不在最后
struct Bad {
    data: str,  // 错误！
    len: usize,
}
```

这种结构体本身也变成非 Sized，通常用于自定义 DST。

## 6. 泛型中的 Sized 约束

### 默认约束

```rust
struct Wrapper<T> {
    value: T,  // T 默认是 Sized
}

// 等价于
struct Wrapper<T: Sized> {
    value: T,
}
```

### 放宽约束

```rust
struct Wrapper<T: ?Sized> {
    value: T,  // 错误！非 Sized 类型不能直接作为字段
}

// 正确做法：通过引用或指针持有
struct Wrapper<T: ?Sized> {
    ptr: *const T,  // 原始指针
}

// 或者使用 Box
struct BoxWrapper<T: ?Sized> {
    value: Box<T>,
}
```

## 常见陷阱

### 忘记 ?Sized 导致 API 不够灵活

```rust
// 不够灵活：只能接受 Sized 类型
fn process<T>(data: &T) { }

// 无法这样调用
let s: &str = "hello";
// process(s);  // 错误！str 不是 Sized
```

**正确做法**：

```rust
fn process<T: ?Sized>(data: &T) { }

let s: &str = "hello";
process(s);  // 正确！
```

### 混淆 `str` 和 `&str`

```rust
// str 是类型，但不能直接使用
fn bad(s: str) { }  // 错误！

// &str 是对 str 的引用，可以使用
fn good(s: &str) { }  // 正确
```

## 最佳实践

| 场景 | 建议 |
| --- | --- |
| 泛型函数接受引用参数 | 考虑使用 `T: ?Sized` 提高灵活性 |
| 定义接受字符串的函数 | 用 `impl AsRef<str>` 或 `&str` |
| 需要存储任意类型 | 使用 `Box<T>` 或 `Box<dyn Trait>` |
| trait 的泛型参数 | 根据是否需要支持 DST 决定是否加 `?Sized` |

## 总结

1. `Sized` 是标记 trait，表示编译期已知大小，由编译器自动实现
2. 泛型参数默认带有 `T: Sized` 约束，用 `?Sized` 放宽
3. 非 Sized 类型（`str`、`[T]`、`dyn Trait`）只能通过引用或智能指针使用
4. 指向非 Sized 类型的引用是胖指针，包含额外元数据
