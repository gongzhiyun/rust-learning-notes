# Drop

`Drop` trait 是 Rust 的析构函数机制，当值离开作用域时自动调用。

Rust 没有垃圾回收器，内存管理完全依赖所有权系统。当一个值的所有者离开作用域，Rust 会自动销毁这个值。对于简单类型（如 `i32`），直接回收内存即可；但对于持有外部资源的类型（文件句柄、网络连接、锁），仅回收内存是不够的——你需要关闭文件、断开连接、释放锁。`Drop` trait 就是为此设计的：它让你定义「值被销毁前要做什么」。

与 C++ 的析构函数类似，但 Rust 的 `Drop` 有几个关键区别：

- **确定性调用**：不依赖 GC，离开作用域立即执行
- **所有权保证**：编译器确保每个值只被 drop 一次
- **与 `Copy` 互斥**：按位复制的类型不能有自定义析构逻辑

## 1. 核心概念

### trait 定义

```rust
trait Drop {
    fn drop(&mut self);
}
```

几个关键点：

**1. 接收 `&mut self` 而非 `self`**

`drop` 拿到的是可变引用，不是所有权。这是因为在 `drop` 被调用时，值还没有被销毁——你需要访问它来做清理工作。`drop` 返回后，Rust 才真正释放内存。

**2. 无返回值**

`drop` 返回 `()`，无法报告错误。如果清理可能失败（如刷新缓冲区），应该提供显式的 `close()` 方法让调用者处理错误，`drop` 中只做尽力而为的清理。

**3. 不能直接调用**

```rust
let s = String::from("hello");
s.drop();  // 错误！explicit destructor calls not allowed
```

Rust 禁止直接调用 `.drop()`，因为作用域结束时还会再调用一次，导致 double free。要提前释放，使用 `std::mem::drop(value)`。

**4. 编译器自动实现递归 drop**

即使你不实现 `Drop`，Rust 也会自动 drop 结构体的每个字段。只有需要**额外**清理逻辑时才手动实现。

### 基本示例

```rust
struct FileHandle {
    name: String,
}

impl Drop for FileHandle {
    fn drop(&mut self) {
        println!("关闭文件: {}", self.name);
    }
}

fn main() {
    let f = FileHandle { name: "data.txt".into() };
    println!("使用文件...");
} // 离开作用域，自动调用 drop
// 输出：
// 使用文件...
// 关闭文件: data.txt
```

## 2. Drop 顺序

### 变量：后声明先 drop

```rust
struct Item(&'static str);

impl Drop for Item {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let a = Item("a");
    let b = Item("b");
    let c = Item("c");
}
// 输出：drop c, drop b, drop a（栈的 LIFO 顺序）
```

### 结构体字段：按声明顺序 drop

```rust
struct Container {
    first: Item,   // 先 drop
    second: Item,  // 后 drop
}

fn main() {
    let c = Container {
        first: Item("first"),
        second: Item("second"),
    };
}
// 输出：drop first, drop second
```

## 3. 手动提前释放

### 使用 `std::mem::drop`

```rust
fn main() {
    let data = vec![1, 2, 3];
    // ... 使用 data
    drop(data);  // 提前释放
    // data 已不可用
    println!("data 已释放，可以做其他事");
}
```

`drop` 函数的实现极其简单：

```rust
fn drop<T>(_x: T) {}  // 接收所有权，函数结束时自动 drop
```

### 常见场景

| 场景 | 说明 |
| --- | --- |
| 释放锁 | 提前释放 `MutexGuard` 避免死锁 |
| 关闭连接 | 不等作用域结束就关闭数据库/网络连接 |
| 释放大内存 | 尽早释放不再需要的大型数据结构 |

```rust
use std::sync::Mutex;

fn example(mutex: &Mutex<i32>) {
    let mut guard = mutex.lock().unwrap();
    *guard += 1;
    drop(guard);  // 提前释放锁
    // 其他不需要锁的操作...
}
```

## 4. Drop 与 Copy 互斥

实现了 `Copy` 的类型**不能**实现 `Drop`：

```rust
#[derive(Copy, Clone)]
struct Point { x: i32, y: i32 }

impl Drop for Point {
    fn drop(&mut self) {}
}
// 错误！Copy 类型不能有 Drop
```

**原因**：`Copy` 类型按位复制，不涉及所有权转移。如果允许 `Drop`，复制后两个值都会调用 `drop`，导致资源重复释放。

## 5. Drop 与 panic

`drop` 方法中 panic 会导致程序 abort（在 unwinding 过程中 panic 是致命的）：

```rust
impl Drop for BadDrop {
    fn drop(&mut self) {
        panic!("drop 中 panic");  // 危险！
    }
}
```

**最佳实践**：`drop` 中不要 panic，错误应该静默处理或记录日志。

## 6. ManuallyDrop

当你需要完全控制 drop 时机时：

```rust
use std::mem::ManuallyDrop;

let mut data = ManuallyDrop::new(String::from("hello"));
// data 不会自动 drop

// 手动 drop（unsafe）
unsafe {
    ManuallyDrop::drop(&mut data);
}
```

**使用场景**：FFI、自定义内存管理、union 中包含需要 drop 的类型。

## 常见陷阱

### 循环引用导致内存泄漏

`Rc` 通过引用计数管理内存：计数归零时才调用 `drop`。但循环引用会让计数永远无法归零。

```rust
use std::rc::Rc;
use std::cell::RefCell;

// 一个简单的节点，可以指向另一个节点
struct Node {
    name: &'static str,
    next: Option<Rc<RefCell<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("{} 被释放", self.name);
    }
}
```

**正常情况（无循环）**：

```rust
fn main() {
    let a = Rc::new(RefCell::new(Node { name: "A", next: None }));
    let b = Rc::new(RefCell::new(Node { name: "B", next: None }));

    // B 指向 A（单向）
    b.borrow_mut().next = Some(Rc::clone(&a));
    // 现在：A 计数=2, B 计数=1
}
// 离开作用域：
// 输出：B 被释放（B 计数 1->0，释放，A 计数 2->1）
// 输出：A 被释放（A 计数 1->0，释放）
```

**循环引用（内存泄漏）**：

```rust
fn main() {
    let a = Rc::new(RefCell::new(Node { name: "A", next: None }));
    let b = Rc::new(RefCell::new(Node { name: "B", next: None }));

    // A 指向 B
    a.borrow_mut().next = Some(Rc::clone(&b));  // B 计数: 1 -> 2
    // B 指向 A（形成循环！）
    b.borrow_mut().next = Some(Rc::clone(&a));  // A 计数: 1 -> 2

    // 现在：A 计数=2, B 计数=2
    // A 被 b.next 引用，B 被 a.next 引用
}
// 离开作用域：
// 变量 b 销毁 -> B 计数 2->1（不为 0，不释放）
// 变量 a 销毁 -> A 计数 2->1（不为 0，不释放）
// 结果：什么都没打印！A 和 B 互相引用，永远无法释放
```

**解决方案**：用 `Weak` 代替其中一个方向的 `Rc`，`Weak` 不增加引用计数。

## 总结

1. `Drop` 是自动析构机制，离开作用域时调用
2. 不能直接调用 `.drop()`，用 `std::mem::drop()` 提前释放
3. `Drop` 和 `Copy` 互斥
4. drop 顺序：变量后声明先 drop，字段按声明顺序 drop
