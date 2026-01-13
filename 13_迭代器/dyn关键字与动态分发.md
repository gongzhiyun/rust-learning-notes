# dyn 关键字与动态分发

`dyn` 是 Rust 中用于表示**动态分发（Dynamic Dispatch）**的关键字。当你看到 `dyn Trait` 时，意思是：编译时不知道具体类型，但知道它实现了某个 Trait，可以在运行时调用该 Trait 的方法。

## 1. 核心概念：静态分发 vs 动态分发

### 静态分发（Static Dispatch）

编译阶段确定具体类型，为每种类型生成专门代码（单态化）。

```rust
// 泛型/静态分发
fn static_dispatch<T: MyTrait>(item: T) {
    item.do_something();
}
```

- **优点**：性能极高，函数调用可被内联
- **缺点**：二进制文件较大；不能在 `Vec` 里同时放不同类型

### 动态分发（Dynamic Dispatch）

编译阶段不知道具体类型，运行时通过查表决定调用哪个函数。

```rust
// 特型对象/动态分发
fn dynamic_dispatch(item: &dyn MyTrait) {
    item.do_something();
}
```

- **优点**：灵活性高，可处理异构集合；代码体积小
- **缺点**：有轻微运行时开销；无法内联优化

## 2. 常见用法

`dyn Trait` 大小在编译时不确定（Unsized），必须通过**指针**使用：

- `&dyn Trait` — 引用
- `Box<dyn Trait>` — 堆分配
- `Rc<dyn Trait>` / `Arc<dyn Trait>` — 引用计数

```rust
trait Animal {
    fn speak(&self);
}

struct Dog;
impl Animal for Dog {
    fn speak(&self) { println!("Woof!"); }
}

struct Cat;
impl Animal for Cat {
    fn speak(&self) { println!("Meow!"); }
}

fn main() {
    // 动态分发：允许异构集合
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog),
        Box::new(Cat),
    ];

    for animal in animals {
        animal.speak(); // 运行时查表调用对应方法
    }
}
```

## 3. 底层原理：胖指针与 Vtable

`&dyn Trait` 或 `Box<dyn Trait>` 底层是**胖指针（Fat Pointer）**，包含两部分：

| 组成部分 | 说明 |
| --- | --- |
| data pointer | 指向实际数据的内存地址 |
| vptr | 指向虚函数表（vtable）的地址 |

**Vtable** 是编译器为每个 `impl Trait` 生成的静态表格，记录了具体类型实现 Trait 方法的函数地址。

调用 `animal.speak()` 的过程：

1. 跟随 `vptr` 找到 vtable
2. 在 vtable 中找到 `speak` 的函数指针
3. 调用该函数，将 `data pointer` 作为 `self` 传入

## 4. 使用场景

| 场景 | 推荐方式 | 原因 |
| --- | --- | --- |
| 追求极致性能 | `impl Trait` / 泛型 | 静态分发无运行时开销 |
| 集合中存储不同类型 | `dyn Trait` | 只有 trait object 能抹平类型差异 |
| 减少二进制体积 | `dyn Trait` | 泛型会为每个类型复制代码 |
| 复杂返回类型 | 两者皆可 | 闭包等场景用 `Box<dyn Trait>` 更简洁 |

## 5. 限制：对象安全性

只有满足**对象安全（Object Safe）**的 Trait 才能用于 `dyn`。

以下情况**不能**用于 `dyn`：

```rust
trait NotObjectSafe {
    fn returns_self(&self) -> Self;  // 错误！返回 Self
    fn generic<T>(&self, t: T);      // 错误！泛型参数
}
```

- **返回 `Self`**：运行时不知道 `Self` 具体多大
- **泛型参数**：vtable 无法为无限可能的泛型存储函数指针

## 总结

1. `dyn` 标志着使用 **Trait Object（特型对象）**
2. 通过 **Fat Pointer + Vtable** 实现运行时多态
3. 牺牲少量性能，换取异构集合的灵活性
