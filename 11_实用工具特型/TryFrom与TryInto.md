# Rust 实用工具特型：TryFrom 与 TryInto

**可能失败的类型转换**

简单说，`TryFrom` 和 `TryInto` 是 `From` 和 `Into` 的「可能失败」版本。比如把 `i32` 转成 `u8`，负数或超过 255 的就转不了，这时候用 `TryFrom` 返回 `Result`，而不是像 `From` 那样直接 panic 或丢失数据。

## 1. 核心概念

### From/Into 的局限

`From` 和 `Into` 适合「永远成功」的转换：

```rust
// 永远成功：&str 转 String
let s: String = "hello".into();

// 永远成功：i32 转 i64
let big: i64 = 42i32.into();
```

但有些转换可能失败：

```rust
// 问题：i32 转 u8，负数咋办？超过 255 咋办？
// 用 From 的话只能 panic 或截断，都不好
fn bad_convert(n: i32) -> u8 {
    if n < 0 || n > 255 {
        panic!("超出范围");  // 不优雅
    }
    n as u8
}
```

这就是 `TryFrom` 的用武之地。

### TryFrom：可能失败的转换

```rust
// TryFrom trait 定义
pub trait TryFrom<T>: Sized {
    // 关联类型：定义转换失败时返回的错误类型
    // 可以是 String、自定义枚举、或任何实现了 Error trait 的类型
    type Error;
    
    // 转换方法：尝试从类型 T 创建 Self
    // - 参数 value: 要转换的源值（获取所有权）
    // - 返回值：Result<Self, Self::Error>
    //   - Ok(Self)：转换成功，返回新值
    //   - Err(Self::Error)：转换失败，返回错误信息
    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

`TryFrom` 回答的问题是："我能从类型 T 创建 Self 吗？可能失败。"

> **为什么需要关联类型 Error？**
> 不同的转换可能有不同的失败原因。比如整数转换失败是因为溢出，字符串解析失败可能是格式错误。用关联类型让每个实现可以定义自己的错误类型，更灵活也更精确。

```rust
use std::convert::TryFrom;

fn main() {
    // 成功的转换
    let small = u8::try_from(100i32);
    println!("{:?}", small);  // Ok(100)

    // 失败的转换
    let too_big = u8::try_from(300i32);
    println!("{:?}", too_big);  // Err(TryFromIntError)

    // 负数也失败
    let negative = u8::try_from(-1i32);
    println!("{:?}", negative);  // Err(TryFromIntError)
}
```

- **返回值**：`Result<Self, Self::Error>`
- **成功**：`Ok(转换后的值)`
- **失败**：`Err(错误信息)`

### TryInto：TryFrom 的镜像

```rust
// TryInto trait 定义
pub trait TryInto<T>: Sized {
    // 关联类型：转换失败时的错误类型
    type Error;
    
    // 转换方法：尝试把 Self 转换成类型 T
    // - 参数 self: 消费自身（获取所有权）
    // - 返回值：Result<T, Self::Error>
    //   - Ok(T)：转换成功，返回目标类型的值
    //   - Err(Self::Error)：转换失败，返回错误信息
    fn try_into(self) -> Result<T, Self::Error>;
}
```

和 `From`/`Into` 一样，实现 `TryFrom` 就自动获得 `TryInto`：

```rust
use std::convert::TryInto;

fn main() {
    // TryFrom 用法：目标类型调用
    let result1 = u8::try_from(100i32);

    // TryInto 用法：源类型调用（自动获得）
    let result2: Result<u8, _> = 100i32.try_into();

    println!("{:?}, {:?}", result1, result2);  // Ok(100), Ok(100)
}
```

> **为什么有两个 trait？**
> 和 `From`/`Into` 一样，`TryFrom` 从目标类型角度看（"我能从什么创建"），`TryInto` 从源类型角度看（"我能转换成什么"）。实现 `TryFrom` 就够了，编译器自动提供 `TryInto`。

## 2. 标准库实现

### 整数类型转换

Rust 标准库为所有整数类型实现了 `TryFrom`：

```rust
use std::convert::TryFrom;

fn main() {
    // i32 → u8：检查范围 [0, 255]
    assert!(u8::try_from(100i32).is_ok());
    assert!(u8::try_from(-1i32).is_err());
    assert!(u8::try_from(256i32).is_err());

    // u32 → i16：检查范围 [-32768, 32767]
    assert!(i16::try_from(1000u32).is_ok());
    assert!(i16::try_from(40000u32).is_err());

    // usize → u32：在 64 位系统上可能失败
    let big: usize = usize::MAX;
    assert!(u32::try_from(big).is_err());
}
```

**关键点**：

- 有符号 ↔ 无符号：检查负数
- 大类型 → 小类型：检查范围
- 平台相关类型（`usize`/`isize`）：检查平台差异

### 自动实现 TryInto

```rust
// 标准库为 TryFrom 自动实现 TryInto
// 这是一个泛型实现（blanket implementation）
impl<T, U> TryInto<U> for T
where
    U: TryFrom<T>,  // 约束：U 必须实现了 TryFrom<T>
{
    // TryInto 的错误类型和 TryFrom 的错误类型保持一致
    type Error = U::Error;

    // try_into 的实现：直接调用目标类型的 try_from
    // self: 源类型的值（类型 T）
    // 返回：Result<U, U::Error>
    fn try_into(self) -> Result<U, U::Error> {
        // 把 self 传给 U::try_from，让目标类型来做转换
        // 这样就不需要重复写转换逻辑了
        U::try_from(self)
    }
}
```

所以你只需要实现 `TryFrom`，就能免费获得 `TryInto`。

> **什么是 blanket implementation？**
> 这是 Rust 的一种泛型实现技巧：为所有满足某个条件的类型自动实现 trait。这里的条件是"U 实现了 TryFrom<T>"，那么 T 就自动获得 TryInto<U>。这样避免了重复代码，也保证了 TryFrom 和 TryInto 的行为一致。

## 3. 自定义实现

### 场景 1：范围检查

```rust
use std::convert::TryFrom;

// 自定义类型：1-100 的分数
// 用 newtype 模式包装 u8，添加业务约束
#[derive(Debug, PartialEq)]
struct Score(u8);

// 为 Score 实现 TryFrom<u8>
impl TryFrom<u8> for Score {
    // 错误类型：用静态字符串，简单直接
    type Error = &'static str;

    fn try_from(value: u8) -> Result<Self, Self::Error> {
        // 验证范围：分数必须在 1-100 之间
        if value >= 1 && value <= 100 {
            // 验证通过：创建 Score 实例
            Ok(Score(value))
        } else {
            // 验证失败：返回错误信息
            Err("分数必须在 1-100 之间")
        }
    }
}

fn main() {
    // 成功：85 在有效范围内
    let score = Score::try_from(85).unwrap();
    println!("分数: {}", score.0);  // 85

    // 失败：0 太小
    match Score::try_from(0) {
        Ok(_) => println!("成功"),
        Err(e) => println!("错误: {}", e),  // 错误: 分数必须在 1-100 之间
    }

    // 失败：101 太大
    match Score::try_from(101) {
        Ok(_) => println!("成功"),
        Err(e) => println!("错误: {}", e),  // 错误: 分数必须在 1-100 之间
    }
}
```

### 场景 2：字符串解析

```rust
use std::convert::TryFrom;

// 自定义类型：邮箱地址
// 用 newtype 包装 String，确保只能存储有效邮箱
#[derive(Debug)]
struct Email(String);

// 为 Email 实现 TryFrom<String>
impl TryFrom<String> for Email {
    type Error = &'static str;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        // 简单验证：检查是否包含 @ 和 .
        // 实际项目中应该用正则表达式或专门的邮箱验证库
        if value.contains('@') && value.contains('.') {
            // 验证通过：创建 Email 实例
            Ok(Email(value))
        } else {
            // 验证失败：返回错误信息
            Err("无效的邮箱格式")
        }
    }
}

// 也支持 &str：方便调用者传字符串字面量
impl TryFrom<&str> for Email {
    type Error = &'static str;

    fn try_from(value: &str) -> Result<Self, Self::Error> {
        // 复用 TryFrom<String> 的实现
        // 先把 &str 转成 String，再调用上面的 try_from
        Email::try_from(value.to_string())
    }
}

fn main() {
    // 成功：有效的邮箱格式
    let email = Email::try_from("user@example.com").unwrap();
    println!("邮箱: {}", email.0);

    // 失败：缺少 @ 和 .
    match Email::try_from("invalid-email") {
        Ok(_) => println!("成功"),
        Err(e) => println!("错误: {}", e),  // 错误: 无效的邮箱格式
    }
}
```

### 场景 3：枚举转换

```rust
use std::convert::TryFrom;

// HTTP 状态码枚举
// 只定义常用的几个状态码
#[derive(Debug, PartialEq)]
enum HttpStatus {
    Ok,
    NotFound,
    InternalError,
}

// 为 HttpStatus 实现 TryFrom<u16>
impl TryFrom<u16> for HttpStatus {
    // 错误类型：用 String，可以包含动态信息
    type Error = String;

    fn try_from(code: u16) -> Result<Self, Self::Error> {
        // 用 match 匹配已知的状态码
        match code {
            // 200：成功
            200 => Ok(HttpStatus::Ok),
            // 404：未找到
            404 => Ok(HttpStatus::NotFound),
            // 500：服务器错误
            500 => Ok(HttpStatus::InternalError),
            // 其他状态码：不支持，返回错误
            // 用 format! 宏生成包含具体状态码的错误信息
            _ => Err(format!("未知状态码: {}", code)),
        }
    }
}

fn main() {
    // 成功：已知的状态码
    assert_eq!(HttpStatus::try_from(200), Ok(HttpStatus::Ok));
    assert_eq!(HttpStatus::try_from(404), Ok(HttpStatus::NotFound));

    // 失败：未知的状态码
    match HttpStatus::try_from(999) {
        Ok(_) => println!("成功"),
        Err(e) => println!("{}", e),  // 未知状态码: 999
    }
}
```

## 4. 常见陷阱

### 陷阱 1：忘记处理错误

```rust
use std::convert::TryFrom;

fn main() {
    // 错误！可能 panic
    let value = u8::try_from(300i32).unwrap();

    // 正确：处理错误
    match u8::try_from(300i32) {
        Ok(v) => println!("成功: {}", v),
        Err(e) => println!("转换失败: {:?}", e),
    }

    // 或者用 ? 操作符传播错误
    fn convert() -> Result<u8, std::num::TryFromIntError> {
        let value = u8::try_from(300i32)?;
        Ok(value)
    }
}
```

**理解**：`try_from` 返回 `Result`，必须处理错误，不要直接 `unwrap`。

### 陷阱 2：TryInto 需要类型标注

```rust
use std::convert::TryInto;

fn main() {
    let n = 100i32;

    // 错误！编译器不知道要转成什么类型
    // let result = n.try_into();

    // 正确：明确指定目标类型
    let result: Result<u8, _> = n.try_into();

    // 或者用 TryFrom，不需要标注
    let result = u8::try_from(n);

    println!("{:?}", result);
}
```

**理解**：和 `Into` 一样，`TryInto` 可能有多个目标类型，需要明确指定。

### 陷阱 3：不要在 From 里做可能失败的转换

```rust
// 错误！From 不应该 panic
// impl From<i32> for Score {
//     fn from(value: i32) -> Self {
//         if value < 1 || value > 100 {
//             panic!("分数超出范围");  // 不优雅
//         }
//         Score(value as u8)
//     }
// }

// 正确：用 TryFrom
use std::convert::TryFrom;

struct Score(u8);

impl TryFrom<i32> for Score {
    type Error = &'static str;

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 1 && value <= 100 {
            Ok(Score(value as u8))
        } else {
            Err("分数必须在 1-100 之间")
        }
    }
}
```

**理解**：`From` 应该是无错转换，可能失败的转换用 `TryFrom`。

### 陷阱 4：错误类型太简单

```rust
// 不好：错误信息不够详细
impl TryFrom<i32> for Score {
    type Error = &'static str;

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 1 && value <= 100 {
            Ok(Score(value as u8))
        } else {
            Err("无效")  // 太简单了
        }
    }
}

// 更好：提供详细的错误信息
#[derive(Debug)]
enum ScoreError {
    TooSmall(i32),
    TooLarge(i32),
}

impl TryFrom<i32> for Score {
    type Error = ScoreError;

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value < 1 {
            Err(ScoreError::TooSmall(value))
        } else if value > 100 {
            Err(ScoreError::TooLarge(value))
        } else {
            Ok(Score(value as u8))
        }
    }
}

fn main() {
    match Score::try_from(150) {
        Ok(_) => println!("成功"),
        Err(ScoreError::TooLarge(v)) => println!("分数 {} 太大了", v),
        Err(ScoreError::TooSmall(v)) => println!("分数 {} 太小了", v),
    }
}
```

## 5. 最佳实践

1. **可能失败就用 TryFrom**：不要在 `From` 里 panic 或返回默认值
2. **提供详细的错误信息**：用自定义错误类型，不要只用 `&str`
3. **优先实现 TryFrom**：自动获得 `TryInto`，避免重复代码

## 总结

1. **TryFrom 是可能失败的 From**：返回 `Result`，不会 panic
2. **实现 TryFrom，免费获得 TryInto**：只需实现一个，编译器自动提供另一个
3. **提供详细的错误信息**：用自定义错误类型，方便调试和处理
