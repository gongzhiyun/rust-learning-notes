# Rust 实用工具特型：ToOwned

**从借用到拥有的桥梁**

简单说，`ToOwned` trait 让你从借用类型（如 `&str`、`&[T]`）创建拥有所有权的类型（如 `String`、`Vec<T>`）。它是 `Clone` 的泛化版本，专门处理「借用 → 拥有」的转换。

```rust
// 最常见的用法
let s: &str = "hello";
let owned: String = s.to_owned();  // &str → String（复制数据到堆上）
println!("{}", s);  // ✅ s 仍然可用，to_owned() 只是借用

let slice: &[i32] = &[1, 2, 3];
let vec: Vec<i32> = slice.to_owned();  // &[i32] → Vec<i32>（克隆每个元素）
println!("{:?}", slice);  // ✅ slice 仍然可用
```

## 1. 核心概念

### ToOwned vs Clone：有什么区别？

`Clone` 和 `ToOwned` 都是用来复制数据的，但适用场景不同：

#### Clone：同类型复制

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // String → String
```

- **输入**：`&Self`
- **输出**：`Self`
- **用途**：从拥有所有权的类型创建副本

#### ToOwned：借用到拥有

```rust
let s: &str = "hello";
let owned = s.to_owned();  // &str → String
```

- **输入**：`&Self`（借用类型）
- **输出**：`Self::Owned`（拥有所有权的类型）
- **用途**：从借用类型创建拥有所有权的副本
- **底层行为**：复制数据（类似 `Clone`，但跨类型）

> **为什么需要 ToOwned？**
> `Clone` 要求输入和输出是同一类型，但 `&str` 不能克隆成 `&str`（引用不拥有数据）。`ToOwned` 解决了这个问题：它让借用类型（`&str`）能转换成拥有所有权的类型（`String`）。
>
> **底层发生了什么？**
> - `&str → String`：分配堆内存，复制字符串的字节数据
> - `&[T] → Vec<T>`：分配堆内存，对每个元素调用 `clone()`
> - 所以 `to_owned()` 是有成本的，涉及内存分配和数据复制

### 关键区别对比

| 特性 | `Clone` | `ToOwned` |
| --- | --- | --- |
| **输入类型** | `&Self` | `&Self`（借用类型） |
| **输出类型** | `Self` | `Self::Owned`（可能不同） |
| **典型例子** | `String → String` | `&str → String` |
| **使用场景** | 复制拥有所有权的值 | 从借用创建拥有所有权的值 |
| **底层行为** | 复制数据（可能涉及堆分配） | 复制数据（必然涉及堆分配） |

```rust
// Clone：同类型复制
let s = String::from("hello");
let cloned: String = s.clone();  // String → String（复制堆数据）

// ToOwned：借用 → 拥有（也是复制数据）
let s: &str = "hello";
let owned: String = s.to_owned();  // &str → String（分配堆内存 + 复制数据）
```

## 2. 底层原理

### Trait 定义

```rust
pub trait ToOwned {
    // 关联类型：定义拥有所有权的目标类型
    // 比如 str 的 Owned 是 String，[T] 的 Owned 是 Vec<T>
    type Owned: Borrow<Self>;
    
    // 核心方法：从借用创建拥有所有权的副本
    // - 参数 &self: 借用类型的引用（如 &str、&[T]）
    // - 返回值：拥有所有权的类型（如 String、Vec<T>）
    fn to_owned(&self) -> Self::Owned;
    
    // 可选方法：克隆数据到已有的拥有类型中
    // - 参数 &self: 源数据的引用
    // - 参数 owned: 目标拥有类型的可变引用
    // - 作用：避免重新分配内存，复用已有的 owned
    fn clone_into(&self, target: &mut Self::Owned) {
        *target = self.to_owned();
    }
}
```

### 关键约束：Borrow 一致性

注意 trait 定义中的约束：`type Owned: Borrow<Self>`

这意味着：**拥有类型必须能借用回原始类型**

```rust
// str 的 ToOwned 实现
impl ToOwned for str {
    type Owned = String;  // String 必须实现 Borrow<str>
    
    fn to_owned(&self) -> String {
        String::from(self)
    }
}

// 验证约束
use std::borrow::Borrow;
let s = String::from("hello");
let borrowed: &str = s.borrow();  // ✅ String 实现了 Borrow<str>
```

这个约束保证了「借用 → 拥有 → 借用」的循环是一致的：

```rust
let original: &str = "hello";
let owned: String = original.to_owned();  // &str → String
let back: &str = owned.borrow();          // String → &str
assert_eq!(original, back);  // ✅ 数据一致
```

### 标准库的具体实现

标准库为常见的借用类型实现了 `ToOwned`：

```rust
// str 实现 ToOwned
impl ToOwned for str {
    type Owned = String;
    
    fn to_owned(&self) -> String {
        // 1. 分配堆内存（容量等于字符串长度）
        // 2. 复制字符串的字节数据到堆上
        // 3. 返回拥有这块堆内存的 String
        String::from(self)
    }
}

// [T] 实现 ToOwned
impl<T: Clone> ToOwned for [T] {
    type Owned = Vec<T>;
    
    fn to_owned(&self) -> Vec<T> {
        // 1. 分配堆内存（容量等于切片长度）
        // 2. 对切片中的每个元素调用 clone()
        // 3. 返回拥有这块堆内存的 Vec<T>
        // 注意：这里需要 T: Clone，因为要复制每个元素
        self.to_vec()
    }
}
```

### 为什么 String 也能调用 to_owned()？

你可能注意到，不只是 `&str` 能用 `to_owned()`，`String` 也可以：

```rust
let s = String::from("hello");
let owned = s.to_owned();  // String 也能用 to_owned()
```

这是因为标准库有一个泛型实现（blanket implementation）：

```rust
// 所有实现 Clone 的类型自动实现 ToOwned
impl<T: Clone> ToOwned for T {
    type Owned = T;  // 拥有类型就是自己
    
    fn to_owned(&self) -> T {
        // 直接调用 clone()
        // 对于已经拥有所有权的类型，to_owned() 就等于 clone()
        self.clone()
    }
}
```

这个泛型实现让所有实现了 `Clone` 的类型（如 `String`、`Vec<T>`）自动获得 `ToOwned`。好处是：

- **统一接口**：不用区分是借用类型还是拥有类型，都能用 `to_owned()`
- **代码复用**：写泛型代码时，只需要 `T: ToOwned` 约束就能处理两种情况

```rust
// 泛型函数：既能处理 &str，也能处理 String
fn process<T: ToOwned + ?Sized>(input: &T) -> T::Owned {
    input.to_owned()
}

let s1: &str = "hello";
let owned1 = process(s1);  // &str → String

let s2 = String::from("world");
let owned2 = process(&s2);  // String → String（通过 clone）
```

## 3. 使用场景

### 字符串处理

```rust
// 从字符串字面量创建 String
let s: &str = "hello";
let owned: String = s.to_owned();

// 函数返回 String
fn get_greeting(name: &str) -> String {
    let mut greeting = "Hello, ".to_owned();
    greeting.push_str(name);
    greeting
}
```

### 切片转 Vec

```rust
// 从切片创建 Vec
let slice: &[i32] = &[1, 2, 3];
let vec: Vec<i32> = slice.to_owned();
```

### Cow（写时克隆）

`ToOwned` 是 `Cow` 的核心依赖。`Cow` 能根据情况自动选择借用或拥有：

```rust
use std::borrow::Cow;

// 处理用户输入：如果已经是小写就不分配内存，否则转小写
fn normalize(input: &str) -> Cow<str> {
    if input.chars().all(|c| c.is_lowercase()) {
        Cow::Borrowed(input)  // 已经是小写，直接借用（零成本）
    } else {
        Cow::Owned(input.to_lowercase())  // 需要转换，分配新字符串
    }
}

let s1 = "hello";  // 已经是小写
let result1 = normalize(s1);  // Borrowed，不分配内存

let s2 = "Hello";  // 有大写字母
let result2 = normalize(s2);  // Owned，调用 to_owned() 分配内存
```

> **什么是 Cow？**
> `Cow`（Clone on Write）让你延迟决定是借用还是拥有。不需要修改时零成本借用，需要修改时才调用 `to_owned()` 分配内存。

## 4. 自定义实现

假设你有一个自定义的借用类型和拥有类型，需要实现它们之间的转换：

```rust
use std::borrow::Borrow;

// 拥有类型
struct MyString(String);

// 借用类型
#[repr(transparent)]
struct MyStr(str);

// 第一步：让拥有类型能借用回借用类型
impl Borrow<MyStr> for MyString {
    fn borrow(&self) -> &MyStr {
        unsafe { &*(self.0.as_str() as *const str as *const MyStr) }
    }
}

// 第二步：为借用类型实现 ToOwned
impl ToOwned for MyStr {
    type Owned = MyString;  // 指定拥有类型
    
    fn to_owned(&self) -> MyString {
        MyString(self.0.to_string())  // 分配内存，复制数据
    }
}
```

核心要点：
1. **Owned 类型必须实现 `Borrow<Self>`**：保证「借用 → 拥有 → 借用」的一致性
2. **to_owned() 负责分配内存**：从借用类型创建拥有类型，涉及堆分配和数据复制

## 5. 常见陷阱

### 陷阱 1：混淆 to_owned() 和 clone()

```rust
let s = String::from("hello");

// 两者效果相同，但语义不同
let owned1 = s.to_owned();  // 强调「获得所有权」
let owned2 = s.clone();     // 强调「复制」

// 对于 &str，只能用 to_owned()
let s: &str = "hello";
let owned = s.to_owned();  // ✅
// let cloned = s.clone();  // ❌ &str 没有实现 Clone
```

**理解**：

- `to_owned()`：从借用创建拥有类型（`&str → String`）
- `clone()`：复制拥有类型（`String → String`）

### 陷阱 2：忘记 ToOwned 的成本

```rust
// 错误！每次循环都分配内存 + 复制数据
let s: &str = "hello";
for _ in 0..1000 {
    let owned = s.to_owned();  // 每次都：分配堆内存 + 复制 5 个字节
    // 使用 owned...
}
```

**正确做法**：

```rust
// 如果只是读取，直接用借用
let s: &str = "hello";
for _ in 0..1000 {
    // 直接使用 s，零成本（不分配，不复制）
    println!("{}", s);
}

// 如果需要拥有类型，在循环外创建
let s: &str = "hello";
let owned = s.to_owned();  // 只分配一次，只复制一次
for _ in 0..1000 {
    // 使用 owned...
    println!("{}", owned);
}
```

**理解**：`to_owned()` 的成本包括：
- **内存分配**：在堆上分配新空间
- **数据复制**：把原始数据复制到新空间
- 对于 `&[T]`，还要对每个元素调用 `clone()`

### 陷阱 3：违反 Borrow 约束

```rust
use std::borrow::Borrow;

struct BadType(String);

// ❌ 错误！Owned 类型没有实现 Borrow<Self>
// impl ToOwned for str {
//     type Owned = BadType;  // BadType 没有实现 Borrow<str>
//     fn to_owned(&self) -> BadType {
//         BadType(self.to_string())
//     }
// }

// ✅ 正确：Owned 类型必须实现 Borrow<Self>
impl Borrow<str> for BadType {
    fn borrow(&self) -> &str {
        &self.0
    }
}

// 现在可以实现 ToOwned 了
// impl ToOwned for str {
//     type Owned = BadType;
//     fn to_owned(&self) -> BadType {
//         BadType(self.to_string())
//     }
// }
```

### 陷阱 4：clone_into() 的性能陷阱

```rust
let s: &str = "hello";
let mut owned = String::new();

// 看起来高效，实际上可能不是
s.clone_into(&mut owned);  // 默认实现：owned = s.to_owned()

// 更好的做法：直接赋值
owned = s.to_owned();

// 或者用 clear + push_str 复用内存
owned.clear();
owned.push_str(s);
```

**理解**：`clone_into()` 的默认实现不会复用内存，除非类型有自定义实现。

## 6. 最佳实践

1. **优先用 to_owned() 而不是 to_string()**：语义更清晰，强调获得所有权而不是格式化
2. **避免不必要的 to_owned()**：能用借用就别拥有，减少内存分配
3. **用 Cow 延迟分配**：只在需要修改时才调用 to_owned()，不需要修改时零成本借用

## 总结

1. **ToOwned 是借用到拥有的桥梁**：让 `&str` 能转成 `String`，`&[T]` 能转成 `Vec<T>`
2. **和 Clone 的区别**：`Clone` 是同类型复制，`ToOwned` 是借用到拥有的转换
3. **核心约束**：`Owned` 类型必须实现 `Borrow<Self>`，保证「借用 → 拥有 → 借用」的一致性
