# dyn

在 Rust 中，`dyn` 是一个关键字，用于表示 **动态分发（Dynamic Dispatch）**。它是 "dynamic" 的缩写。

简单来说，当你看到 `dyn Trait` 时，意思是：“我手里有一个对象，我**在编译时不知道**它具体是什么类型（是结构体 A 还是结构体 B），但我知道它**实现了某个 Trait**，所以我可以在运行时调用该 Trait 的方法。”

## 1. 核心概念：静态分发 vs 动态分发

理解 `dyn` 的最好方法是将其与 Rust 默认的“静态分发”进行对比。

#### 静态分发 (Static Dispatch) - `impl Trait` 或 泛型

在编译阶段，编译器就已经知道具体的类型，并为每种类型生成专门的代码（这叫单态化，Monomorphization）。

```rust
// 泛型/静态分发
fn static_dispatch<T: MyTrait>(item: T) {
    item.do_something();
}
```

- **优点**：性能极高（函数调用直接硬编码，甚至被内联）。
- **缺点**：生成的二进制文件较大（代码膨胀）；灵活性差（比如不能在一个 `Vec` 里同时放 `Dog` 和 `Cat`）。

#### 动态分发 (Dynamic Dispatch) - `dyn Trait`

在编译阶段，编译器**不知道**具体类型，只知道它实现了该 Trait。具体调用哪个函数，是在程序**运行**时通过查表决定的。

```rust
// 特型对象/动态分发
fn dynamic_dispatch(item: &dyn MyTrait) {
    item.do_something();
}
```

- **优点**：灵活性高（可以处理异构集合）；代码体积小。

- **缺点**：有轻微的运行时开销（查表）；无法被编译器内联优化。

## 2. dyn的常见用法

由于 `dyn Trait` 的大小在编译时是不确定的（Unsized），所以你**不能**直接拥有一个 `dyn Trait` 类型的变量。你必须总是通过**指针**（引用或智能指针）来使用它。

常见的形式有：

- `&dyn Trait`
- `Box<dyn Trait>`
- `Rc<dyn Trait>` / `Arc<dyn Trait>`

**代码示例**

假设我们有一个 `Animal` Trait 和两个实现者 `Dog` 和 `Cat`。

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
    // 静态分发：类型必须一致
    let dog = Dog;
    let cat = Cat;

    // 动态分发：允许异构集合
    // 我们创建了一个包含 Box<dyn Animal> 的 Vec
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(dog),
        Box::new(cat),
    ];

    for animal in animals {
        // 在运行时，程序会查找 animal 具体是啥，然后调用对应的 speak
        animal.speak(); 
    }
}
```



## 3. 底层原理： 胖指针（Fat Pointer） 与 Vtable

当使用 `&dyn Trait` 或 `Box<dyn Trait>` 时，Rust 在底层使用的是一种特殊的指针，称为**胖指针 (Fat Pointer)**。

普通的指针（如 `&i32`）只是一个 64 位的内存地址。 **胖指针**包含两个部分（通常是 128 位）：

1. **data pointer**：指向实际数据的内存地址（例如 `Dog` 结构体的实例）。
2. **vptr (virtual pointer)**：指向一个 **虚函数表 (vtable)** 的地址。

**Vtable (虚函数表)** 是编译器为每个 `impl Trait` 生成的一个静态表格，里面记录了具体类型（如 `Dog`）实现 Trait 方法（如 `speak`）的函数地址。

**当你调用 `animal.speak()` 时，过程如下：**

1. 程序跟随胖指针中的 `vptr` 找到 vtable。
2. 在 vtable 中找到 `speak` 方法的函数指针。
3. 调用该函数指针，并将 `data pointer` 作为 `self` 传入。

## 4. 什么时候使用dyn？

| **场景**                     | **推荐方式**        | **原因**                                                     |
| ---------------------------- | ------------------- | ------------------------------------------------------------ |
| **追求极致性能**             | `impl Trait` / 泛型 | 静态分发无运行时开销，利于 CPU 分支预测和内联。              |
| **需要在集合中存储不同类型** | `dyn Trait`         | `Vec<T>` 中的 T 必须统一，只有 `Box<dyn Trait>` 能抹平类型差异。 |
| **减少二进制体积**           | `dyn Trait`         | 泛型会为每个类型复制一份代码，`dyn` 只有一份代码。           |
| **作为函数返回值**           | 两者皆可            | 如果返回类型很复杂或是闭包，用 `Box<dyn Trait>` 有时写起来更简单（虽然 `impl Trait` 也可以）。 |

## 5. 限制：对象安全性（Object Safety）

并不是所有的 Trait 都可以变成 `dyn Trait`。只有满足 **对象安全 (Object Safe)** 的 Trait 才可以。

如果一个 Trait 有以下情况，它就**不能**用于 `dyn`：

1. **方法返回 `Self` 类型**：因为运行时不知道 `Self` 具体多大。
   - *例子：* `fn clone(&self) -> Self;` (这也是为什么 `Clone` trait 通常不能用于 trait object)。
2. **方法有泛型参数**：因为 vtable 没法为无限可能的泛型参数存储函数指针。
   - *例子：* `fn foo<T>(&self, t: T);`

## 总结

- **`dyn`** 标志着我们在使用 **Trait Object（特型对象）**。
- 它通过 **Fat Pointer（胖指针）** 和 **Vtable（虚表）** 实现运行时的多态。
- 它牺牲了一点点运行时性能，换取了极大的灵活性（允许在同一个容器中存储不同类型）。

